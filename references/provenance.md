# Provenance

The engineering ledger of this skill, for the person editing it. The skill was written by the engineer who
owns the calendar integration of a booking API, where hosts connect their calendars to block availability
and receive bookings as events, and audited against the earlier implementation of that integration. The
earlier implementation had already been hardened once: single-use OAuth state with PKCE, encrypted tokens,
HMAC-signed channel tokens bound to their channel, persisted token refreshes, and reconnect clean-up that no
longer failed the callback. Those are carried over as they were.

The audit read the integration end to end, together with the slot engine that consumes its output. Each
defect below was confirmed against the earlier implementation; each is fixed in the templates and, where it
can be, pinned by a test in [test-suite.md](test-suite.md). The ledger has four parts: what the audit fixed,
what was kept deliberately and why that is safe, what the audit changed in the design, and what was designed
here and has never run in production.

## Fixed in the templates

### 1. A booking's own event blocked the rest of its session

Events written for bookings were tagged with the booking id, but busy sync never looked at the tag.
On a calendar with both syncs on, the first booking of a group session came back seconds later as a
busy block of unlimited quantity, and the engine (which deliberately ignores the host's bookings of the
same service) still hid the remaining seats. **Shipped:** events carrying the private marker are
skipped, in [busy-sync.md](busy-sync.md).

### 2. Renewal stopped the old channel before opening the new one

If the new watch call failed, the calendar was left with no channel and no channel record, so the cron
never found it again. Busy sync stayed "on" in the UI while its blocks froze. **Shipped:** new before
old, the old kept on failure, and the cron opens a channel for any busy calendar without one, in
[channels-and-settings.md](channels-and-settings.md).

### 3. Busy time froze on quiet calendars

The window was `[now, now + 180 days]`, but it only moved when a notification arrived. A calendar
nobody edited for a month never imported events entering the window, and missed notifications were
never repaired. **Shipped:** the cron resyncs every busy calendar.

### 4. Concurrent notifications left phantom busy blocks

Busy blocks had random ids. Two syncs started by one burst both saw an event as new and both created a
block. The next sync keyed blocks by event id in a map, so one duplicate was updated and deleted with the
event while the other stayed forever, blocking time the host had freed. **Shipped:** deterministic ids, a
per-calendar lease with a queued rerun, and a diff that deletes every stored row not desired.

### 5. Multi-day all-day events blocked only their first day

All-day events were mapped from `start.date` to 23:59:59.999 of the same day; `end.date` was ignored.
A week's vacation blocked Monday. **Shipped:** `start.date` to the exclusive `end.date`, at local
midnights of the calendar's zone, correct on DST days.

### 6. Only the first 2500 events were read

`events.list` was called once with `maxResults: 2500`; `nextPageToken` was ignored, and the diff then
deleted the blocks of every event past the first page. **Shipped:** every page, and any failure aborts
before deleting, in [google-api.md](google-api.md).

### 7. Free and declined events blocked

Events marked *Free* (most all-day events by default) and invitations the host declined were imported as
busy. **Shipped:** both skipped, except on holiday calendars.

### 8. Shared calendars could not receive bookings, and new rows looked read-only

Only `owner` counted as writable, though `writer` can insert events. Separately, a calendar's first
listing stored `isWritable: false` regardless of its role, so the booking-sync toggle was unavailable
until the list was loaded a second time. **Shipped:** `owner` or `writer`, correct on first listing, and a
server-side guard on enabling booking sync.

### 9. A revoked grant looked like "no calendars"

Every listing error returned an empty list, nothing recorded `invalid_grant`, and renewals failed twice a
day in the logs only. **Shipped:** `reauth_required` on the connection, surfaced through health and the
list route.

### 10. Reconnecting wiped every setting

Re-authorizing (the fix for a revoked grant) deleted all calendar rows and busy blocks, even for the same
Google account. **Shipped:** the account e-mail identifies the account; the same account keeps its rows
and gets fresh channels, a different one is torn down.

### 11. Booking events were not idempotent

Events got Google-generated ids. A repeated trigger, or the backfill when booking sync was switched on
(which de-duplicated only against the 180-day window), created a second event. **Shipped:** the event id
is derived from the booking id; 409 becomes an update, in [booking-events.md](booking-events.md).

### 12. Smaller ones

- A host without an e-mail address among a booking's hosts made Google reject the whole event.
- The channel token held the calendar id, user id and channel id in clear, signed: a long calendar id
  could push it past Google's 256-character limit. It now holds only the channel id.
- There was no disconnect: tokens stayed stored and the grant stayed valid at Google.
- Calendars removed in Google kept their rows, channels and busy blocks.
- Without an encryption key configured, tokens were stored in clear text. The templates require the key.
- The OAuth callback answered errors with JSON to a browser.

### 13. The second Next.js route did not compile on its own

Found by the 0.2.0 release check, not by the audit. `app/api/calendar/google/calendars/[calendarId]/route.ts`
shared a code block with the `[action]` route and used `calendarRoutes` without importing it, so the file
an agent copied failed `tsc`. **Shipped:** its own block with its own import, in
[http-routes.md](http-routes.md); the release check writes every block to its own path and compiles it.

## Kept deliberately

- **Full-window resync per notification**, not `syncToken` incremental sync. It costs more quota and
  repairs itself after any lost notification or bug; incremental sync is an optional extension.
- **180-day horizon, 12-hour cron, 48-hour renewal window.** The earlier implementation's numbers; four
  renewal attempts per channel lifetime.
- **Inserts silent, deletes `sendUpdates: 'all'`.** The app sends its own confirmations; Google removes
  the event from attendees' calendars on cancel. Configurable.
- **Only the primary host's calendars receive the event**; other hosts are attendees. Deletion walks
  every host.
- **Holiday calendars are not watched**, only synced by the cron.
- **`workingLocation` events never block.**
- **One Google account per user.**
- **Turning booking sync off leaves existing events.**
- **`access_type=offline` with `prompt=consent`** on every connect.

## Changed in the design

- **Plain `fetch` instead of `googleapis`**, so token refresh and `invalid_grant` are explicit and the
  client can be faked in tests.
- **Private instead of shared extended properties** for the booking marker, so attendees' copies are not
  mistaken for the app's own events.
- **Busy blocks hold UTC instants only.** The earlier implementation also stored a wall-clock copy in the
  calendar's zone, which a consumer once converted with the business's zone and shifted.
- **Top-level collections** instead of per-user subcollections, so the cron, webhook and availability
  queries are single queries.
- **Web `Request`/`Response` handlers** instead of Express routes plus GraphQL resolvers.

## Designed here, not yet run in production

None of these ran in the earlier implementation. The tests pin them; production has not yet.

- Derived per-calendar health with an operator severity order, in [operations.md](operations.md).
- Single-flight sync lease with a queued rerun.
- Disconnect with revocation at Google.
- Granted-scope check on connect (`canWrite`).
- Orphan channel clean-up in the cron.
- Restoring a deleted booking event on re-confirmation (409 turns into an update with `status: 'confirmed'`).
- Supabase/Postgres store and schema, in [supabase-backend.md](supabase-backend.md).
- The test suite, memory store and fake Google, in [testing.md](testing.md).

## If you are fixing an existing integration

An integration with the defects above is fixed in this order, most damaging first:

1. Own booking events re-imported (#1): lost seats on every group session whose host enabled both syncs
   on one calendar.
2. Renewal order and periodic resync (#2, #3): busy time stops updating with no error.
3. Deterministic busy-block ids and a row-based diff (#4): phantom busy time that never clears.
4. All-day span (#5) and free/declined events (#7): wrong availability, visible to guests.
5. Pagination (#6): only matters past 2500 events in the window, catastrophic when it does.
6. Reauth state and same-account reconnect (#9, #10): support load and failures nobody sees.
7. Writable roles (#8), idempotent events (#11), the smaller ones (#12).

Busy blocks stored with random ids need no migration: the first sync after the change deletes them and
writes deterministic ones. Events that integration wrote with a generated id and a *shared* marker do not
match the new id and are not recognised by the private-marker skip. Until the last of them is in the
past, also skip events whose `extendedProperties.shared.bookingId` is set, and delete them on
cancellation by searching `sharedExtendedProperty=bookingId=<id>`.
