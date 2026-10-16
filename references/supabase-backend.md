# Supabase / Postgres backend

`CalendarStore` on `@supabase/supabase-js` with a service-role client, and the schema it needs.

## Schema

```sql
-- calendar-sync/store/schema.sql
-- Server-only tables: RLS is on with no policies, so anon and authenticated clients see nothing.
-- The API uses the service-role key. user_id is the host's user id; add a foreign key to your
-- users table if it is in this database.

create table public.calendar_auth_requests (
  state text primary key,
  user_id text not null,
  code_verifier text not null,
  expires_at timestamptz not null
);
create index calendar_auth_requests_expiry on public.calendar_auth_requests (expires_at);

create table public.calendar_connections (
  user_id text primary key,
  provider text not null default 'google' check (provider = 'google'),
  account_email text not null,
  status text not null check (status in ('connected', 'reauth_required')),
  can_write boolean not null,
  encrypted_tokens text not null,
  connected_at timestamptz not null,
  updated_at timestamptz not null,
  last_error text
);

create table public.calendar_links (
  id text primary key,
  user_id text not null,
  external_id text not null,
  title text not null,
  access_role text not null check (access_role in ('freeBusyReader', 'reader', 'writer', 'owner')),
  is_writable boolean not null,
  sync_busy boolean not null default false,
  sync_bookings boolean not null default false,
  channel_id text,
  last_synced_at timestamptz,
  last_sync_error text,
  sync_lease_until timestamptz,
  resync_requested boolean not null default false,
  version integer not null default 1,
  unique (user_id, external_id),
  check (not sync_bookings or is_writable)
);
create index calendar_links_user on public.calendar_links (user_id);
create index calendar_links_busy on public.calendar_links (id) where sync_busy;

create table public.calendar_channels (
  id text primary key,
  user_id text not null,
  -- No foreign key: a channel must outlive its row until it has been stopped at Google.
  calendar_id text not null,
  resource_id text not null,
  expires_at timestamptz not null,
  created_at timestamptz not null
);
create index calendar_channels_expiry on public.calendar_channels (expires_at);

create table public.calendar_busy_blocks (
  id text primary key,
  user_id text not null,
  calendar_id text not null references public.calendar_links (id) on delete cascade,
  event_id text not null,
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  all_day boolean not null default false,
  check (ends_at > starts_at)
);
create index calendar_busy_blocks_calendar on public.calendar_busy_blocks (calendar_id);
-- Availability reads: a host's blocks overlapping a range.
create index calendar_busy_blocks_user_time on public.calendar_busy_blocks (user_id, ends_at, starts_at);

alter table public.calendar_auth_requests enable row level security;
alter table public.calendar_connections enable row level security;
alter table public.calendar_links enable row level security;
alter table public.calendar_channels enable row level security;
alter table public.calendar_busy_blocks enable row level security;
```

- **RLS on, no policies.** Every table is server-only; with RLS enabled and nothing granted, the anon
  and authenticated roles see nothing, even if a client is pointed at them by mistake. The API uses the
  service-role key, which bypasses RLS.
- **`check (not sync_bookings or is_writable)`** puts the booking-sync guard in the database too, so a
  bug in a later code path cannot write bookings to a read-only calendar.
- **Channels have no foreign key to calendars** on purpose: a channel must outlive its row until it is
  stopped at Google. Busy blocks do cascade: they mean nothing without their calendar.
- **`user_id` is text** so it fits Supabase auth UUIDs and external ids (Firebase, Clerk) alike. Add a
  foreign key if the users table is in this database.

## Concurrency without transactions

supabase-js talks to PostgREST, which runs one statement per request; there is no client-side
transaction. `mutateCalendar` uses optimistic concurrency instead:

1. Read the row and its `version`.
2. Compute the next row with `fn`.
3. `update ... where id = $id and version = $read` and ask for the row back.
4. No row back means someone else wrote first: start again (up to five times).

Creation is `insert`; a unique violation (`23505`) means a concurrent request created it, and the loop
retries as an update. `consumeAuthRequest` is a single `delete ... returning`, atomic on its own.

If you have direct Postgres access (Drizzle, Prisma, `pg`), `select ... for update` inside a transaction
is the simpler equivalent; keep the `version` column anyway, it costs nothing and makes lost updates
visible.

## Notes

- **PostgREST returns at most 1000 rows per request by default**; `listBusyBlocks` pages with `range`
  until a short page. Without that, a calendar with more than 1000 blocks would diff against a partial
  list and re-upsert the rest every run.
- **`in()` filters go into the URL**; deletes chunk at 100 ids to stay under proxy URL limits.
- **The availability query** is `user_id = $1 and ends_at > $from and starts_at < $to`, served by
  `calendar_busy_blocks_user_time`.
- **Expired auth requests** are swept by the cron (`deleteExpiredAuthRequests`), or by `pg_cron`:
  `delete from calendar_auth_requests where expires_at < now()`.

## Code

```ts
// calendar-sync/store/supabase.ts
import type { SupabaseClient } from '@supabase/supabase-js';

import type {
  BusyBlock,
  CalendarConnection,
  CalendarStore,
  GoogleAccessRole,
  LinkedCalendar,
  WatchChannel,
} from '../types';

const CAS_ATTEMPTS = 5;
const CHUNK = 500;
/** `.in()` goes into the URL; keep it well under proxy limits. */
const IN_CHUNK = 100;

interface CalendarRow {
  id: string;
  user_id: string;
  external_id: string;
  title: string;
  access_role: GoogleAccessRole;
  is_writable: boolean;
  sync_busy: boolean;
  sync_bookings: boolean;
  channel_id: string | null;
  last_synced_at: string | null;
  last_sync_error: string | null;
  sync_lease_until: string | null;
  resync_requested: boolean;
  version: number;
}

const date = (value: string | null) => (value ? new Date(value) : null);
const iso = (value: Date | null) => (value ? value.toISOString() : null);

function toCalendar(row: CalendarRow): LinkedCalendar {
  return {
    id: row.id,
    userId: row.user_id,
    externalId: row.external_id,
    title: row.title,
    accessRole: row.access_role,
    isWritable: row.is_writable,
    syncBusy: row.sync_busy,
    syncBookings: row.sync_bookings,
    channelId: row.channel_id,
    lastSyncedAt: date(row.last_synced_at),
    lastSyncError: row.last_sync_error,
    syncLeaseUntil: date(row.sync_lease_until),
    resyncRequested: row.resync_requested,
    version: row.version,
  };
}

function fromCalendar(calendar: LinkedCalendar): CalendarRow {
  return {
    id: calendar.id,
    user_id: calendar.userId,
    external_id: calendar.externalId,
    title: calendar.title,
    access_role: calendar.accessRole,
    is_writable: calendar.isWritable,
    sync_busy: calendar.syncBusy,
    sync_bookings: calendar.syncBookings,
    channel_id: calendar.channelId,
    last_synced_at: iso(calendar.lastSyncedAt),
    last_sync_error: calendar.lastSyncError,
    sync_lease_until: iso(calendar.syncLeaseUntil),
    resync_requested: calendar.resyncRequested,
    version: calendar.version,
  };
}

interface ChannelRow {
  id: string;
  user_id: string;
  calendar_id: string;
  resource_id: string;
  expires_at: string;
  created_at: string;
}

const toChannel = (row: ChannelRow): WatchChannel => ({
  id: row.id,
  userId: row.user_id,
  calendarId: row.calendar_id,
  resourceId: row.resource_id,
  expiresAt: new Date(row.expires_at),
  createdAt: new Date(row.created_at),
});

interface BusyRow {
  id: string;
  user_id: string;
  calendar_id: string;
  event_id: string;
  starts_at: string;
  ends_at: string;
  all_day: boolean;
}

const toBusy = (row: BusyRow): BusyBlock => ({
  id: row.id,
  userId: row.user_id,
  calendarId: row.calendar_id,
  eventId: row.event_id,
  from: new Date(row.starts_at),
  to: new Date(row.ends_at),
  allDay: row.all_day,
});

interface ConnectionRow {
  user_id: string;
  provider: 'google';
  account_email: string;
  status: CalendarConnection['status'];
  can_write: boolean;
  encrypted_tokens: string;
  connected_at: string;
  updated_at: string;
  last_error: string | null;
}

function check<T>(result: { data: T; error: { message: string; code?: string } | null }): T {
  if (result.error) throw new Error(`supabase: ${result.error.message}`);
  return result.data;
}

/** Pass a service-role client: these tables have RLS on and no policies. */
export function createSupabaseCalendarStore(supabase: SupabaseClient): CalendarStore {
  return {
    async saveAuthRequest(request) {
      check(
        await supabase.from('calendar_auth_requests').insert({
          state: request.state,
          user_id: request.userId,
          code_verifier: request.codeVerifier,
          expires_at: request.expiresAt.toISOString(),
        })
      );
    },

    async consumeAuthRequest(state) {
      // DELETE ... RETURNING is atomic: of two concurrent callbacks, one gets the row.
      const row = check(
        await supabase.from('calendar_auth_requests').delete().eq('state', state).select().maybeSingle()
      ) as { state: string; user_id: string; code_verifier: string; expires_at: string } | null;
      return row
        ? { state: row.state, userId: row.user_id, codeVerifier: row.code_verifier, expiresAt: new Date(row.expires_at) }
        : null;
    },

    async deleteExpiredAuthRequests(before) {
      const rows = check(
        await supabase.from('calendar_auth_requests').delete().lt('expires_at', before.toISOString()).select('state')
      );
      return rows?.length ?? 0;
    },

    async getConnection(userId) {
      const row = check(
        await supabase.from('calendar_connections').select('*').eq('user_id', userId).maybeSingle()
      ) as ConnectionRow | null;
      if (!row) return null;
      return {
        userId: row.user_id,
        provider: row.provider,
        accountEmail: row.account_email,
        status: row.status,
        canWrite: row.can_write,
        encryptedTokens: row.encrypted_tokens,
        connectedAt: new Date(row.connected_at),
        updatedAt: new Date(row.updated_at),
        lastError: row.last_error,
      };
    },

    async saveConnection(connection) {
      const row: ConnectionRow = {
        user_id: connection.userId,
        provider: connection.provider,
        account_email: connection.accountEmail,
        status: connection.status,
        can_write: connection.canWrite,
        encrypted_tokens: connection.encryptedTokens,
        connected_at: connection.connectedAt.toISOString(),
        updated_at: connection.updatedAt.toISOString(),
        last_error: connection.lastError,
      };
      check(await supabase.from('calendar_connections').upsert(row));
    },

    async patchConnection(userId, patch) {
      const row: Partial<ConnectionRow> = {};
      if (patch.status !== undefined) row.status = patch.status;
      if (patch.encryptedTokens !== undefined) row.encrypted_tokens = patch.encryptedTokens;
      if (patch.updatedAt !== undefined) row.updated_at = patch.updatedAt.toISOString();
      if (patch.lastError !== undefined) row.last_error = patch.lastError;
      if (patch.canWrite !== undefined) row.can_write = patch.canWrite;
      check(await supabase.from('calendar_connections').update(row).eq('user_id', userId));
    },

    async deleteConnection(userId) {
      check(await supabase.from('calendar_connections').delete().eq('user_id', userId));
    },

    async listCalendars(userId) {
      const rows = check(await supabase.from('calendar_links').select('*').eq('user_id', userId)) as CalendarRow[];
      return rows.map(toCalendar);
    },

    async getCalendar(calendarId) {
      const row = check(
        await supabase.from('calendar_links').select('*').eq('id', calendarId).maybeSingle()
      ) as CalendarRow | null;
      return row ? toCalendar(row) : null;
    },

    async mutateCalendar(calendarId, fn) {
      // supabase-js has no transactions; optimistic concurrency on `version` instead.
      for (let attempt = 0; attempt < CAS_ATTEMPTS; attempt++) {
        const currentRow = check(
          await supabase.from('calendar_links').select('*').eq('id', calendarId).maybeSingle()
        ) as CalendarRow | null;
        const before = currentRow ? toCalendar(currentRow) : null;
        const next = fn(before);
        if (!next) return { before, after: before };

        const row = fromCalendar({ ...next, id: calendarId, version: (before?.version ?? 0) + 1 });
        if (!before) {
          const inserted = await supabase.from('calendar_links').insert(row).select().maybeSingle();
          if (inserted.error?.code === '23505') continue; // created concurrently: retry as an update
          const created = check(inserted) as CalendarRow | null;
          return { before, after: created ? toCalendar(created) : null };
        }
        const updated = check(
          await supabase
            .from('calendar_links')
            .update(row)
            .eq('id', calendarId)
            .eq('version', before.version)
            .select()
            .maybeSingle()
        ) as CalendarRow | null;
        if (updated) return { before, after: toCalendar(updated) };
      }
      throw new Error(`calendar ${calendarId}: too much write contention`);
    },

    async deleteCalendar(calendarId) {
      check(await supabase.from('calendar_links').delete().eq('id', calendarId));
    },

    async listBusySyncCalendars(cursor, limit) {
      let query = supabase.from('calendar_links').select('*').eq('sync_busy', true).order('id').limit(limit);
      if (cursor) query = query.gt('id', cursor);
      return (check(await query) as CalendarRow[]).map(toCalendar);
    },

    async saveChannel(channel) {
      const row: ChannelRow = {
        id: channel.id,
        user_id: channel.userId,
        calendar_id: channel.calendarId,
        resource_id: channel.resourceId,
        expires_at: channel.expiresAt.toISOString(),
        created_at: channel.createdAt.toISOString(),
      };
      check(await supabase.from('calendar_channels').upsert(row));
    },

    async getChannel(channelId) {
      const row = check(
        await supabase.from('calendar_channels').select('*').eq('id', channelId).maybeSingle()
      ) as ChannelRow | null;
      return row ? toChannel(row) : null;
    },

    async deleteChannel(channelId) {
      check(await supabase.from('calendar_channels').delete().eq('id', channelId));
    },

    async listChannelsExpiringBefore(before, limit) {
      const rows = check(
        await supabase
          .from('calendar_channels')
          .select('*')
          .lt('expires_at', before.toISOString())
          .order('expires_at')
          .limit(limit)
      ) as ChannelRow[];
      return rows.map(toChannel);
    },

    async listBusyBlocks(calendarId) {
      const rows: BusyRow[] = [];
      // PostgREST caps a response (1000 rows by default); page until a short page.
      for (let offset = 0; ; offset += 1000) {
        const page = check(
          await supabase
            .from('calendar_busy_blocks')
            .select('*')
            .eq('calendar_id', calendarId)
            .order('id')
            .range(offset, offset + 999)
        ) as BusyRow[];
        rows.push(...page);
        if (page.length < 1000) break;
      }
      return rows.map(toBusy);
    },

    async applyBusyDiff(_calendarId, upserts, deleteIds) {
      for (let i = 0; i < upserts.length; i += CHUNK) {
        const rows: BusyRow[] = upserts.slice(i, i + CHUNK).map(block => ({
          id: block.id,
          user_id: block.userId,
          calendar_id: block.calendarId,
          event_id: block.eventId,
          starts_at: block.from.toISOString(),
          ends_at: block.to.toISOString(),
          all_day: block.allDay,
        }));
        check(await supabase.from('calendar_busy_blocks').upsert(rows));
      }
      for (let i = 0; i < deleteIds.length; i += IN_CHUNK) {
        check(await supabase.from('calendar_busy_blocks').delete().in('id', deleteIds.slice(i, i + IN_CHUNK)));
      }
    },

    async deleteBusyBlocks(calendarId) {
      check(await supabase.from('calendar_busy_blocks').delete().eq('calendar_id', calendarId));
    },
  };
}
```

## Checklist

- [ ] `schema.sql` applied; RLS enabled on all five tables
- [ ] Store built with the service-role key, server-side only
- [ ] Availability reads `calendar_busy_blocks` by `user_id` with the overlap condition
