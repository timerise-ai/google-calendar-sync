---
name: google-calendar-sync
description: >
  Build two-way Google Calendar sync for a booking or scheduling backend: hosts connect their Google
  account with OAuth, events on chosen calendars block their availability (push notifications plus a
  renewal cron), and confirmed bookings are written to their calendar as events, with Google Meet
  links. Use when: (1) an app with bookings, appointments, classes or shifts must respect staff's
  Google Calendar busy time or put bookings into it, (2) existing calendar sync shows phantom busy
  time, missed or duplicate events, group sessions blocked after the first booking, silently stopped
  syncing, or users stuck after revoking access, (3) the user mentions: Google Calendar integration,
  two-way calendar sync, connect Google Calendar, busy time, free/busy, events.watch, push
  notifications, X-Goog-Channel-Token, channel renewal, sync calendar exceptions, calendar.events
  scope, refresh token, invalid_grant, offline access, PKCE, Google Meet link, conferenceData,
  extendedProperties, hangoutLink. Carries a dependency-free Google client on fetch, a pure busy-time
  mapper and diff, single-flight sync, idempotent booking events, make-before-break channel renewal,
  derived health per calendar, Firestore and Supabase stores, and a 32-test node:test suite run under
  four process time zones. TypeScript on any Node server (Next.js route handlers, Express); storage
  and the booking source are seams. Not Outlook/Microsoft 365 (microsoft-calendar-sync), not a calendar UI, not ICS feeds.
---

# Google Calendar Sync

Two directions, one rule each. **Inbound**: Google is the truth about a host's other commitments, so
busy blocks are a disposable mirror of a time window, rebuilt by a full resync and never edited by
hand. **Outbound**: the booking store is the truth about bookings, so an event written to Google is
addressed by an id derived from the booking, and is never read back in as busy time. Most defects in
calendar integrations are one direction leaking into the other.

## When to use

- Hosts connect Google Calendar and pick calendars that block their availability.
- Bookings appear in the host's Google Calendar, optionally with a Meet link, and vanish on cancel.
- Auditing an existing Google sync: duplicates, ghosts, silent stops, reconnect loops.
- Moving that sync between Firestore and Supabase/Postgres.

## When NOT to use

- **Generating bookable slots, subtracting busy time** → `time-slot-generator`. This skill *produces*
  the busy blocks it consumes.
- **Outlook / Microsoft 365** → `microsoft-calendar-sync`: the same architecture and record shapes on
  Microsoft Graph.
- **Showing a calendar UI or sending invitations** → the host's own e-mail; this writes events silently.
- **Read-only availability without per-user OAuth** → Google's free/busy query with a service account
  and domain-wide delegation is a different design.

## Architecture

```
          connect (PKCE, single-use state) ─▶ callback ─▶ encrypted tokens ─▶ calendar rows
                                                                                  │
 inbound   Google ── X-Goog-* ─▶ /notifications ─▶ syncBusy (lease) ─▶ list all pages ─▶ toBusyBlock ─▶ diffBusy ─▶ busy blocks ─▶ availability
              ▲                                         ▲
              └── watch channel ◀── cron: renew new-before-old, resync every busy calendar
 outbound  booking confirmed ─▶ pushBookingEvent ─▶ insert id=bk<sha256> (409 → update) ─▶ Meet link back
           booking cancelled ─▶ removeBookingEvent ─▶ delete by id (404/410 = done)
```

## Critical facts

1. **Busy sync is a full window resync, not an event stream.** A notification says only "something
   changed". List every page of `[now, now + horizon)` and diff. A partial list must abort, never delete.
2. **Ids are deterministic everywhere.** Calendar rows, busy blocks and booking events are hashes of
   what they represent. Concurrent notifications, retries and double triggers then converge instead of
   duplicating.
3. **Your own events come back.** A booking written to a calendar that also has busy sync is
   re-imported unless it carries the private `bookingId` marker and is skipped. With seat-based
   services, that blocks the rest of a group session after the first booking.
4. **Google only tells you what changed while the channel is alive.** Channels expire, notifications
   are not reliably retried, and the horizon moves. The cron renews channels and resyncs every busy
   calendar; without it, busy time silently freezes.
5. **`invalid_grant` is a state, not an error.** Revoked access, six months unused, or seven days in
   "Testing" mode. Mark the connection `reauth_required`, surface it, and let reconnecting the same
   account keep every setting.
6. **All-day events span `start.date` to the exclusive `end.date`, at local midnights of the calendar's
   zone.** Timed events carry offsets. Send outbound instants with their `Z`; stripping it shifts events.
7. **Free and declined events do not block.** Google's own free/busy ignores them. Holiday calendars are
   the exception: they are only enabled in order to block.

## Hard rules

> **Never take the user id from the OAuth callback URL.** The state is a key to a server-side,
> single-use, ten-minute record. Otherwise anyone attaches their calendar to someone else's account.

> **Never trust a notification before its channel token verifies against the channel id it arrived on.**
> The endpoint is public; the token is the only credential.

> **Never stop the old channel before the new one exists.** If the new watch fails, the calendar has
> no channel, nothing renews it, and its busy time freezes while the UI says "on".

> **Never store tokens in clear text.** A refresh token is long-lived read/write access to a calendar.

> **Never let a Google failure fail a booking.** Outbound writes log and return; the booking stands.

## Adaptation Contract

| Seam | The skill ships | The host supplies |
|---|---|---|
| Domain entities | Connection, LinkedCalendar, WatchChannel, BusyBlock, BookingForCalendar | Its names (staff, resource, appointment…) |
| Tenant scope | Per user: one Google account per `userId` | Its user id; org scoping stays in its guard |
| Auth guard | `authenticate(request) → userId \| null` | Session, Clerk, NextAuth, Firebase, Supabase |
| Data access | `CalendarStore` + Firestore and Supabase implementations | Its ORM or SDK |
| Bookings | `BookingSource` + `pushBookingEvent` / `removeBookingEvent` | Calls them from its booking status flow |
| Availability | Busy blocks per user, `listHostBusy` query | Its slot engine (`time-slot-generator`) |
| HTTP | Web `Request → Response` handlers | Next.js route handlers, Express adapter |
| Background work | `renewChannels`, every 12 h; optional `defer` for webhooks | Cloud Scheduler, Vercel Cron, `after()` |
| Strings | `describeBooking` (event title/body), error codes | Its i18n |
| Secrets | Client id/secret, channel secret, 32-byte token key | Its secret manager |
| Tests | `node:test` suite, memory store, fake Google | Its runner |

## Quick start

1. Create the Google Cloud OAuth client and consent screen —
   [oauth-and-tokens.md](references/oauth-and-tokens.md).
2. Copy the types and decide the rename — [data-model.md](references/data-model.md).
3. Copy the Google client — [google-api.md](references/google-api.md).
4. Copy inbound sync — [busy-sync.md](references/busy-sync.md); outbound —
   [booking-events.md](references/booking-events.md); lifecycle —
   [channels-and-settings.md](references/channels-and-settings.md).
5. Pick a store — [firestore-backend.md](references/firestore-backend.md) or
   [supabase-backend.md](references/supabase-backend.md).
6. Mount the routes and the cron — [http-routes.md](references/http-routes.md).
7. Copy the suite and run it under two `TZ` values — [testing.md](references/testing.md),
   [test-suite.md](references/test-suite.md).
8. Give operators the health view — [operations.md](references/operations.md).
9. Porting an existing integration? Fix order in [provenance.md](references/provenance.md).

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Entities, store contract, rename | LinkedCalendar, BusyBlock, WatchChannel, CalendarStore, mutateCalendar, version, compare-and-swap | [data-model.md](references/data-model.md) |
| Connecting an account | OAuth, PKCE, state, consent screen, scopes, refresh_token, invalid_grant, reauth, revoke, AES-256-GCM, Testing mode | [oauth-and-tokens.md](references/oauth-and-tokens.md) |
| Talking to Google | fetch, calendarList, events.list, pageToken, events.watch, channels.stop, 409 duplicate, 410, rate limit | [google-api.md](references/google-api.md) |
| Busy time in | toBusyBlock, transparency, declined, all-day, end.date, DST, diffBusy, lease, resyncRequested, horizon | [busy-sync.md](references/busy-sync.md) |
| Bookings out | event id, base32hex, extendedProperties private, Meet, conferenceData, hangoutLink, sendUpdates, backfill | [booking-events.md](references/booking-events.md) |
| Channels and toggles | startChannel, renewChannels, make-before-break, orphan channel, holiday calendar, syncBusy, syncBookings, planTransitions | [channels-and-settings.md](references/channels-and-settings.md) |
| Routes and webhook | X-Goog-Channel-ID, X-Goog-Resource-State, sync, exists, 204, callback redirect, cron secret, Next.js, Express | [http-routes.md](references/http-routes.md) |
| Firestore | firebase-admin, runTransaction, batch 500, calendarBusyBlocks, composite index, security rules | [firestore-backend.md](references/firestore-backend.md) |
| Supabase / Postgres | schema.sql, RLS, service role, optimistic concurrency, 23505, PostgREST 1000 rows | [supabase-backend.md](references/supabase-backend.md) |
| Proving it | node:test, memory store, fake Google, TZ=, mutation check | [testing.md](references/testing.md), [test-suite.md](references/test-suite.md) |
| Running it | health, stale, channel-expired, env vars, quotas, dashboards, runbook, ngrok, localhost | [operations.md](references/operations.md) |
| Why it is built this way | provenance, defects, kept deliberately, port order | [provenance.md](references/provenance.md) |
