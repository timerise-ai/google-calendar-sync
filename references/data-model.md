# Data model

Five records, one store interface. Everything else in the skill is functions over these.

| Record | Key | Lives | Holds |
|---|---|---|---|
| `AuthRequest` | random 43-char `state` | 10 minutes, deleted on first read | user id, PKCE verifier |
| `CalendarConnection` | `userId` | until disconnect | account e-mail, status, write permission, encrypted tokens |
| `LinkedCalendar` | `calendarRowId(userId, externalId)` | while Google lists the calendar | the two toggles, channel pointer, sync state, lease |
| `WatchChannel` | random UUID (the Google channel id) | until renewed or stopped | Google `resourceId`, expiry |
| `BusyBlock` | `busyBlockId(calendarId, eventId)` | until the event changes or leaves the window | `from`, `to` (UTC), all-day flag |

Booking events are not stored: their Google id is computed from the booking id, so there is nothing
to look up. See [booking-events.md](booking-events.md).

## Rename table

Decide the host's words before copying, and rename everywhere at once.

| Skill | Source app called it | Typical host words |
|---|---|---|
| `userId` (the host) | user | staff, provider, employee, member, resource owner |
| `LinkedCalendar` | user calendar | connected calendar |
| `syncBusy` | `syncExceptions` | "check for conflicts", "block my availability" |
| `syncBookings` | `syncBookings` | "add bookings to calendar" |
| `BusyBlock` | exception (`slotType: 'EXCEPTION'`) | busy time, blocked time, conflict |
| `WatchChannel` | calendar subscription | push channel, webhook subscription |
| `BookingForCalendar` | booking | appointment, reservation, session, shift |

Keep Google's own terms (`externalId`, `resourceId`, `accessRole`, `transparency`) as they are; they
are the API's vocabulary, not the domain's.

## Lifecycle

```
             connect             toggle syncBusy on                  toggle off / calendar gone / disconnect
 (nothing) ──────────▶ rows ─────────────────────────▶ channel + blocks ─────────────────────────────────▶ (nothing)
                        │  invalid_grant                       │  cron every 12 h: renew channel, resync
                        ▼                                      ▼
                 reauth_required ── reconnect same account ──▶ connected, settings kept, channels reopened
                                 └─ reconnect other account ─▶ old rows torn down, new rows
```

## Types

```ts
// calendar-sync/types.ts
import type { GoogleApi } from './googleApi';

/** One Google account per user. `reauth_required` is set when Google answers `invalid_grant`. */
export type ConnectionStatus = 'connected' | 'reauth_required';

export interface CalendarConnection {
  userId: string;
  provider: 'google';
  /** The id of the account's primary calendar, which is its e-mail address. Identifies the account on reconnect. */
  accountEmail: string;
  status: ConnectionStatus;
  /** False when the user unticked the events scope on the consent screen: busy sync works, booking sync does not. */
  canWrite: boolean;
  /** `StoredTokens` as JSON, AES-256-GCM encrypted. Never stored in clear text. */
  encryptedTokens: string;
  connectedAt: Date;
  updatedAt: Date;
  lastError: string | null;
}

export interface StoredTokens {
  accessToken: string;
  refreshToken: string;
  /** Epoch milliseconds. */
  expiresAt: number;
  scope: string;
}

export type GoogleAccessRole = 'freeBusyReader' | 'reader' | 'writer' | 'owner';

/** A calendar of the connected account, as this app knows it. One row per (user, Google calendar). */
export interface LinkedCalendar {
  /** `calendarRowId(userId, externalId)`, deterministic so concurrent listings cannot create duplicates. */
  id: string;
  userId: string;
  /** Google's calendar id: an e-mail address, `…@group.calendar.google.com`, or `…#holiday@group.v.calendar.google.com`. */
  externalId: string;
  title: string;
  accessRole: GoogleAccessRole;
  isWritable: boolean;
  /** Inbound: this calendar's events block the user's availability. */
  syncBusy: boolean;
  /** Outbound: bookings hosted by this user are written to this calendar. */
  syncBookings: boolean;
  channelId: string | null;
  lastSyncedAt: Date | null;
  lastSyncError: string | null;
  syncLeaseUntil: Date | null;
  resyncRequested: boolean;
  /** Owned by the store: incremented on every write, used for compare-and-swap. */
  version: number;
}

export interface CalendarSettingsPatch {
  title?: string;
  syncBusy?: boolean;
  syncBookings?: boolean;
}

/** A Google push-notification channel watching one calendar's events. */
export interface WatchChannel {
  id: string;
  userId: string;
  calendarId: string;
  resourceId: string;
  expiresAt: Date;
  createdAt: Date;
}

/** A busy interval imported from Google. Feeds the availability engine as host busy time. */
export interface BusyBlock {
  /** `busyBlockId(calendarId, eventId)`, deterministic so concurrent syncs converge instead of duplicating. */
  id: string;
  userId: string;
  calendarId: string;
  eventId: string;
  from: Date;
  to: Date;
  allDay: boolean;
}

/** Server-side state of an OAuth round trip, consumed exactly once. */
export interface AuthRequest {
  state: string;
  userId: string;
  codeVerifier: string;
  expiresAt: Date;
}

/** What the outbound side needs to know about a booking. The host maps its own booking onto this. */
export interface BookingForCalendar {
  bookingId: string;
  start: Date;
  end: Date;
  /** IANA zone the event is displayed in, usually the business's zone. */
  timeZone: string;
  title: string;
  guestName?: string;
  guestEmail?: string;
  /** Primary host first. Only the primary host's calendars receive the event. */
  hostIds: string[];
  hostEmails: Record<string, string | undefined>;
  links?: { label: string; url: string }[];
  detailsUrl?: string;
  notes?: string;
  wantsGoogleMeet?: boolean;
}

export interface BookingSource {
  /** Confirmed bookings starting at or after `from` whose primary host is `userId`. */
  listUpcomingForPrimaryHost(userId: string, from: Date): Promise<BookingForCalendar[]>;
}

export interface Logger {
  info(kind: string, data?: Record<string, unknown>): void;
  warn(kind: string, data?: Record<string, unknown>): void;
  error(kind: string, data?: Record<string, unknown>): void;
}

export interface CalendarStore {
  saveAuthRequest(request: AuthRequest): Promise<void>;
  /** Read and delete in one atomic step. Expiry is checked by the caller. */
  consumeAuthRequest(state: string): Promise<AuthRequest | null>;
  deleteExpiredAuthRequests(before: Date): Promise<number>;

  getConnection(userId: string): Promise<CalendarConnection | null>;
  saveConnection(connection: CalendarConnection): Promise<void>;
  patchConnection(
    userId: string,
    patch: Partial<Pick<CalendarConnection, 'status' | 'encryptedTokens' | 'updatedAt' | 'lastError' | 'canWrite'>>
  ): Promise<void>;
  deleteConnection(userId: string): Promise<void>;

  listCalendars(userId: string): Promise<LinkedCalendar[]>;
  getCalendar(calendarId: string): Promise<LinkedCalendar | null>;
  /**
   * Compare-and-swap. `fn` receives the current row (or null) and returns the row to write, or null
   * to leave it unchanged. Retried on a concurrent write, so `fn` must be pure. Errors thrown by `fn`
   * propagate. The store sets `version`.
   */
  mutateCalendar(
    calendarId: string,
    fn: (current: LinkedCalendar | null) => LinkedCalendar | null
  ): Promise<{ before: LinkedCalendar | null; after: LinkedCalendar | null }>;
  deleteCalendar(calendarId: string): Promise<void>;
  /** Calendars with `syncBusy`, ordered by id, starting after `cursor`. */
  listBusySyncCalendars(cursor: string | null, limit: number): Promise<LinkedCalendar[]>;

  saveChannel(channel: WatchChannel): Promise<void>;
  getChannel(channelId: string): Promise<WatchChannel | null>;
  deleteChannel(channelId: string): Promise<void>;
  listChannelsExpiringBefore(before: Date, limit: number): Promise<WatchChannel[]>;

  listBusyBlocks(calendarId: string): Promise<BusyBlock[]>;
  applyBusyDiff(calendarId: string, upserts: BusyBlock[], deleteIds: string[]): Promise<void>;
  deleteBusyBlocks(calendarId: string): Promise<void>;
}

export interface CalendarSyncConfig {
  clientId: string;
  clientSecret: string;
  /** Exactly as registered in the Google Cloud console. */
  redirectUri: string;
  /** Public HTTPS URL of the notifications route, with a publicly trusted certificate. */
  notificationUrl: string;
  /** HMAC key for channel tokens. Fails closed when empty. */
  channelSecret: string;
  /** 32 bytes as 64 hex chars or 44 base64 chars. */
  encryptionKey: string;
  /** How far ahead busy time is imported. Must cover the booking horizon. */
  horizonDays: number;
  /** Channels expiring sooner than this are renewed by the cron. */
  renewBeforeMs: number;
  /** A crashed sync releases its lease after this long. */
  leaseMs: number;
  /** What Google sends attendees when a booking's event is deleted. */
  sendUpdatesOnCancel: 'all' | 'none';
}

export const DEFAULT_SYNC_SETTINGS = {
  horizonDays: 180,
  renewBeforeMs: 48 * 60 * 60 * 1000,
  leaseMs: 2 * 60 * 1000,
  sendUpdatesOnCancel: 'all',
} as const satisfies Partial<CalendarSyncConfig>;

export interface CalendarSyncDeps {
  store: CalendarStore;
  google: GoogleApi;
  config: CalendarSyncConfig;
  bookings: BookingSource;
  log: Logger;
  now: () => Date;
  /** Called with the Meet URL Google generated for a booking's event. */
  onConferenceLink?: (bookingId: string, url: string) => Promise<void>;
  /** Event title and body. Defaults to `defaultDescribeBooking`; supply one for i18n. */
  describeBooking?: (booking: BookingForCalendar) => { summary: string; description: string };
}

export type CalendarErrorCode =
  | 'not-connected'
  | 'reauth-required'
  | 'calendar-not-found'
  | 'calendar-not-writable'
  | 'invalid-or-expired-state'
  | 'no-refresh-token'
  | 'missing-calendar-scope';

export class CalendarSyncError extends Error {
  readonly code: CalendarErrorCode;

  constructor(code: CalendarErrorCode, message: string = code) {
    super(message);
    this.name = 'CalendarSyncError';
    this.code = code;
  }
}
```

## Why the store looks like this

- **`mutateCalendar` is the only way to change a calendar row.** Settings toggles, the sync lease, the
  channel pointer and error state all go through compare-and-swap on `version`. A toggle racing a sync,
  or two syncs racing each other, cannot overwrite one another's fields. `fn` may run several times, so
  it must be pure: no I/O inside, no counters outside that are not reset at its top (see the
  `acquired` flag in [busy-sync.md](busy-sync.md)).
- **`after` is the row as stored after the call.** When `fn` returns null nothing is written and
  `after` equals `before`. Callers rely on this to read the current state without a second query.
- **The sync lease lives on the calendar row**, not in a separate lock table. It needs no cleanup: a
  crashed holder's lease simply expires after `leaseMs`.
- **Channels are their own records** because a webhook arrives with only a channel id, and because a
  channel must outlive its calendar row until it has been stopped at Google.
- **Busy blocks are per user, not per project.** A person is busy across every business they work for.
  If the availability engine stores blocks with a project id, query these by user id and merge them in
  rather than copying them into each project.
- **Tokens are one encrypted string.** The store never sees them in clear text, so a database export,
  backup or search-index sync cannot leak them.

## Checklist

- [ ] Rename decided and applied to types, store names, routes and UI copy
- [ ] `version` owned by the store; `fn` passed to `mutateCalendar` is pure
- [ ] No field of `StoredTokens` persisted outside `encryptedTokens`
- [ ] Busy blocks queried by user id from the availability engine
