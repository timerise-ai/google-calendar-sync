# Testing

The suite runs the real flows against two doubles: a store with the contract's compare-and-swap
semantics, and a Google that pages, answers 409 on a reused event id and 404/410 on deletes. No network,
no emulator, no framework: `node:test` and `node:assert`. The tests themselves are in
[test-suite.md](test-suite.md).

## Running

Node 22.18+ runs TypeScript directly by stripping types; the templates avoid syntax that needs
transpiling (no enums, no constructor parameter properties). Node's loader needs file extensions, so
rewrite the extensionless imports in a copy:

```bash
cp -R calendar-sync /tmp/cs-run
find /tmp/cs-run -name '*.ts' -exec perl -pi -e "s/from '(\.{1,2}\/[^']+)'/from '\$1.ts'/g" {} +
for tz in UTC Europe/Warsaw Pacific/Auckland America/Los_Angeles; do
  TZ=$tz node --test --experimental-strip-types /tmp/cs-run/rules.test.ts /tmp/cs-run/flows.test.ts
done
```

Copy both files as they are and run them as they are: no converted assertions, no shim runner, no
dropped test; 32 tests per time zone is the count to report. In a host with vitest or jest, the files run
unchanged once the runner's TypeScript transform is on
(`node:test`'s `test` maps to the runner's `test`, or import from `vitest`).

**Run under at least two process time zones.** The all-day mapping must not depend on the server's
`TZ`; a test that passes only in UTC is how "works in production, fails on my laptop" starts.

## What the suite pins

| Test | Guards against |
|---|---|
| multi-day all-day event | A vacation blocking only its first day |
| spring-forward all-day event, Santiago midnight gap | 24 h days and missing midnights |
| own bookings not imported | Group sessions blocked after the first booking |
| free / declined / cancelled / working location | Busy time the user does not have |
| holiday calendars block | Holidays ignored once transparency is respected |
| diff deletes duplicates | Phantom blocks that survive the event's deletion |
| channel tokens bind to their channel | A captured token replayed under another channel |
| event ids valid and stable | Duplicate events on retry; ids Google rejects |
| `Z` kept, marker private, hosts without e-mail dropped | Shifted events; guests' busy time skipped; rejected events |
| writers writable, writable on first listing | Shared calendars unusable; a dead toggle after connecting |
| connect: state single use, expiry, user from server | Calendar attached to another account |
| without the events scope nothing is writable | Booking writes failing with 403 |
| every page imported; failed page deletes nothing | Truncated calendars; mass deletion on a transient error |
| a burst converges | Duplicate blocks from concurrent notifications |
| renewal new-before-old, kept on failure | Calendars losing their channel with no error |
| renewal heals a lost channel | `syncBusy: true` with no channel, forever |
| pushed twice is one event; mirror never busy | Duplicate events; self-blocking |
| cancel twice, re-confirm restores | Errors on repeat; 409 on a re-confirmed booking |
| revoked grant marks reauth, reconnect keeps settings | Empty calendar lists; users re-configuring after every revoke |
| other account tears down | Old account's blocks and channels left behind |
| calendar removed in Google | Busy time from a calendar the user unsubscribed from |

## Proving the tests can fail

A suite that passed on its first run has not shown it can catch anything. Mutate the code back toward
each defect and confirm a test fails. Each of these must turn the suite red:

| Mutation | File |
|---|---|
| `to = localMidnight(addDays(start.date, 1))` instead of `end.date` | `busyRules.ts` |
| remove the `own-booking` skip | `busyRules.ts` |
| ignore `transparency` | `busyRules.ts` |
| never delete rows missing from `desired` | `busyRules.ts` |
| stop after the first page | `busySync.ts` |
| stop the old channel before starting the new one | `channels.ts` |
| writable only for `owner` | `calendars.ts` |
| no update on 409 | `bookingEvents.ts` |
| accept a token without comparing its channel id | `webhook.ts` |
| auth request not deleted on consume | `testing/memoryStore.ts` |

## Doubles

```ts
// calendar-sync/testing/memoryStore.ts
import type {
  AuthRequest,
  BusyBlock,
  CalendarConnection,
  CalendarStore,
  LinkedCalendar,
  WatchChannel,
} from '../types';

/** Yields to the event loop, so concurrent callers interleave the way they would against a database. */
const tick = () => new Promise<void>(resolve => setImmediate(resolve));

/** The store contract in memory: a test double, and the executable statement of each method's semantics. */
export function createMemoryStore() {
  const authRequests = new Map<string, AuthRequest>();
  const connections = new Map<string, CalendarConnection>();
  const calendars = new Map<string, LinkedCalendar>();
  const channels = new Map<string, WatchChannel>();
  const busy = new Map<string, BusyBlock>();

  const store: CalendarStore = {
    async saveAuthRequest(request) {
      authRequests.set(request.state, { ...request });
    },
    async consumeAuthRequest(state) {
      const request = authRequests.get(state) ?? null;
      authRequests.delete(state);
      return request;
    },
    async deleteExpiredAuthRequests(before) {
      let count = 0;
      for (const [state, request] of authRequests) {
        if (request.expiresAt < before) {
          authRequests.delete(state);
          count++;
        }
      }
      return count;
    },

    async getConnection(userId) {
      const connection = connections.get(userId);
      return connection ? { ...connection } : null;
    },
    async saveConnection(connection) {
      connections.set(connection.userId, { ...connection });
    },
    async patchConnection(userId, patch) {
      const current = connections.get(userId);
      if (current) connections.set(userId, { ...current, ...patch });
    },
    async deleteConnection(userId) {
      connections.delete(userId);
    },

    async listCalendars(userId) {
      return [...calendars.values()].filter(c => c.userId === userId).map(c => ({ ...c }));
    },
    async getCalendar(calendarId) {
      const calendar = calendars.get(calendarId);
      return calendar ? { ...calendar } : null;
    },
    async mutateCalendar(calendarId, fn) {
      for (;;) {
        const snapshot = calendars.get(calendarId);
        const before = snapshot ? { ...snapshot } : null;
        await tick();
        const next = fn(before ? { ...before } : null);
        await tick();
        // Compare-and-swap: someone wrote in between, run `fn` again on the new state.
        if ((calendars.get(calendarId)?.version ?? 0) !== (before?.version ?? 0)) continue;
        if (!next) return { before, after: before };
        const after = { ...next, id: calendarId, version: (before?.version ?? 0) + 1 };
        calendars.set(calendarId, after);
        return { before, after: { ...after } };
      }
    },
    async deleteCalendar(calendarId) {
      calendars.delete(calendarId);
    },
    async listBusySyncCalendars(cursor, limit) {
      return [...calendars.values()]
        .filter(c => c.syncBusy && (cursor === null || c.id > cursor))
        .sort((a, b) => (a.id < b.id ? -1 : 1))
        .slice(0, limit)
        .map(c => ({ ...c }));
    },

    async saveChannel(channel) {
      channels.set(channel.id, { ...channel });
    },
    async getChannel(channelId) {
      const channel = channels.get(channelId);
      return channel ? { ...channel } : null;
    },
    async deleteChannel(channelId) {
      channels.delete(channelId);
    },
    async listChannelsExpiringBefore(before, limit) {
      return [...channels.values()].filter(c => c.expiresAt < before).slice(0, limit);
    },

    async listBusyBlocks(calendarId) {
      await tick();
      return [...busy.values()].filter(b => b.calendarId === calendarId).map(b => ({ ...b }));
    },
    async applyBusyDiff(_calendarId, upserts, deleteIds) {
      await tick();
      for (const block of upserts) busy.set(block.id, { ...block });
      for (const id of deleteIds) busy.delete(id);
    },
    async deleteBusyBlocks(calendarId) {
      for (const [id, block] of busy) if (block.calendarId === calendarId) busy.delete(id);
    },
  };

  return { store, authRequests, connections, calendars, channels, busy };
}
```

```ts
// calendar-sync/testing/fakeGoogle.ts
import {
  GoogleApiError,
  SCOPE_EVENTS,
  SCOPE_READ,
  type GoogleApi,
  type GoogleCalendarListEntry,
  type GoogleEvent,
} from '../googleApi';

/** Google Calendar in memory: enough of its behaviour (paging, 409 on ids, 404 on delete) to test the flows. */
export function createFakeGoogle(options: { account?: string; pageSize?: number } = {}) {
  const state = {
    account: options.account ?? 'host@example.com',
    grantedScopes: [SCOPE_READ, SCOPE_EVENTS],
    calendarList: [] as GoogleCalendarListEntry[],
    events: new Map<string, Map<string, GoogleEvent>>(),
    timeZone: 'Europe/Warsaw',
    pageSize: options.pageSize ?? 2500,
    channels: new Map<string, { calendarId: string; resourceId: string }>(),
    refreshFails: null as null | 'invalid_grant',
    watchFails: false,
    failPage: null as null | number,
    calls: [] as string[],
    issued: 0,
  };

  const eventsOf = (calendarId: string) => {
    let events = state.events.get(calendarId);
    if (!events) state.events.set(calendarId, (events = new Map()));
    return events;
  };
  const tokens = () => ({
    access_token: `access-${++state.issued}`,
    refresh_token: `refresh-${state.issued}`,
    expires_in: 3600,
    scope: state.grantedScopes.join(' '),
    token_type: 'Bearer',
  });

  const api: GoogleApi = {
    authUrl: ({ state: s, codeChallenge }) => `https://accounts.example/auth?state=${s}&cc=${codeChallenge}`,
    async exchangeCode() {
      state.calls.push('exchangeCode');
      return tokens();
    },
    async refresh() {
      state.calls.push('refresh');
      if (state.refreshFails) throw new GoogleApiError(400, state.refreshFails, 'Token has been expired or revoked.');
      const { refresh_token: _drop, ...rest } = tokens();
      return rest;
    },
    async revoke() {
      state.calls.push('revoke');
    },
    async primaryCalendarId() {
      return state.account;
    },
    async listCalendars() {
      state.calls.push('listCalendars');
      return state.calendarList.map(entry => ({ ...entry }));
    },
    async listEvents(_token, calendarId, { pageToken }) {
      const page = Number(pageToken ?? 0);
      state.calls.push(`listEvents:${calendarId}:${page}`);
      if (state.failPage === page) throw new GoogleApiError(503, 'backendError', 'Backend Error');
      const all = [...eventsOf(calendarId).values()];
      const items = all.slice(page * state.pageSize, (page + 1) * state.pageSize);
      const more = (page + 1) * state.pageSize < all.length;
      return { items, timeZone: state.timeZone, ...(more ? { nextPageToken: String(page + 1) } : {}) };
    },
    async insertEvent(_token, calendarId, event) {
      state.calls.push(`insert:${event.id}`);
      const events = eventsOf(calendarId);
      if (event.id && events.has(event.id)) throw new GoogleApiError(409, 'duplicate', 'The requested identifier already exists.');
      const stored = { ...event, id: event.id ?? `gen${events.size}` };
      events.set(stored.id, stored);
      return stored;
    },
    async updateEvent(_token, calendarId, eventId, event) {
      state.calls.push(`update:${eventId}`);
      const stored = { ...event, id: eventId };
      eventsOf(calendarId).set(eventId, stored);
      return stored;
    },
    async deleteEvent(_token, calendarId, eventId) {
      state.calls.push(`delete:${eventId}`);
      const events = eventsOf(calendarId);
      const event = events.get(eventId);
      if (!event) throw new GoogleApiError(404, 'notFound', 'Not Found');
      if (event.status === 'cancelled') throw new GoogleApiError(410, 'deleted', 'Resource has been deleted');
      events.set(eventId, { ...event, status: 'cancelled' });
    },
    async watchEvents(_token, calendarId, channel) {
      state.calls.push(`watch:${calendarId}`);
      if (state.watchFails) throw new GoogleApiError(503, 'backendError', 'Backend Error');
      const resourceId = `res-${calendarId}`;
      state.channels.set(channel.id, { calendarId, resourceId });
      return { id: channel.id, resourceId, expiration: String(Date.parse('2026-10-23T00:00:00Z')) };
    },
    async stopChannel(_token, channel) {
      state.calls.push(`stop:${channel.id}`);
      state.channels.delete(channel.id);
    },
  };

  return { api, state, eventsOf };
}
```

## Checklist

- [ ] Suite green under at least two `TZ` values
- [ ] Every mutation in the table turns it red
- [ ] A host-specific store passes the flow tests when swapped in for the memory store
