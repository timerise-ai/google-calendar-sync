# Adaptation

How the module lands in a host app: what the host must already have, the seams it fills, the order of work,
and the non-negotiables that do not bend.

## The host must already have

- A **Next.js App Router** app on the Node.js runtime, or another Node server that can mount Web `Request`
  and `Response` handlers (an Express adapter ships). The module uses `node:crypto`.
- **Signed-in users** who will be hosts. Any auth works behind `authenticate(request)`.
- **Firestore** or **Supabase** (Postgres) for data. Both stores ship, behind one `CalendarStore` contract.
- **Bookings with a status flow** the host can hook: the module is told when a booking is confirmed or
  cancelled, and it reads upcoming bookings through `BookingSource` for backfill.
- A **scheduler** that can call one URL every 12 hours: Cloud Scheduler, Vercel Cron, or any cron.
- A **public HTTPS URL** with a trusted certificate for Google's notifications. Locally, a tunnel, or the
  cron route called by hand ([operations.md](operations.md)).
- A **Google Cloud project** where the OAuth client and consent screen can be created
  ([oauth-and-tokens.md](oauth-and-tokens.md)).

## Seam contract

| Seam | This skill ships | The host supplies |
|---|---|---|
| Domain words | Connection, LinkedCalendar, WatchChannel, BusyBlock, BookingForCalendar, and a rename table in [data-model.md](data-model.md) | Its own words for each |
| Tenant scope | Per user: one Google account per `userId` | Its user id; org scoping stays in its guard |
| Auth | `authenticate(request): Promise<string \| null>` | Its session lookup: Clerk, NextAuth, Firebase Auth, Supabase Auth |
| Data | `CalendarStore` with Firestore and Supabase implementations | Its project, the index and rules or the schema applied |
| Bookings | `BookingSource`, `pushBookingEvent`, `removeBookingEvent` | Calls from its booking status flow |
| Availability | Busy blocks per user and the `listHostBusy` query | Its slot engine, for example `time-slot-generator` |
| HTTP | Web `Request` and `Response` handlers | Next.js route handlers or the Express adapter |
| Background work | `renewChannels` every 12 hours; an optional `defer` for the webhook | Its scheduler; `after()` from `next/server` for `defer` |
| Strings | `describeBooking` for the event title and body, error codes | Its i18n |
| Secrets | Client id and secret, channel secret, 32-byte token key | Its secret manager ([operations.md](operations.md)) |
| Tests | A `node:test` suite with a memory store and a fake Google | Its runner, if not `node:test` |
| UI | Nothing: the routes return JSON for a settings screen | The settings screen, built with its components |

## Order of work

1. Read the host: auth, data store, booking status flow, scheduler, test runner. Fill the right-hand column
   above and the rename table, and show them to the person asking before writing code.
2. Create the Google Cloud OAuth client and consent screen ([oauth-and-tokens.md](oauth-and-tokens.md)).
3. Copy the types ([data-model.md](data-model.md)), the Google client ([google-api.md](google-api.md)) and
   the pure rules, then the rules suite from [test-suite.md](test-suite.md). Run it under two `TZ` values.
4. Copy inbound sync ([busy-sync.md](busy-sync.md)), outbound events
   ([booking-events.md](booking-events.md)) and the channel lifecycle
   ([channels-and-settings.md](channels-and-settings.md)), then the doubles and the flows suite
   ([testing.md](testing.md)). Run them.
5. Copy the store for the host's database ([firestore-backend.md](firestore-backend.md) or
   [supabase-backend.md](supabase-backend.md)) and apply its index and rules or its schema.
6. Mount the routes and the cron ([http-routes.md](http-routes.md)); wire `authenticate` to the host's
   session and `pushBookingEvent` and `removeBookingEvent` to its booking status flow.
7. Feed the busy blocks to the host's availability, and build the settings screen on the list route.
8. Set the environment, schedule the cron, and give operators the health view
   ([operations.md](operations.md)).

## Non-negotiables

The same five rules as `SKILL.md` and the README, in the same order.

1. **Never take the user id from the OAuth callback URL.** The state is a key to a server-side, single-use,
   ten-minute record. Otherwise anyone attaches their calendar to someone else's account.
2. **Never trust a notification before its channel token verifies against the channel id it arrived on.**
   The endpoint is public; the token is the only credential.
3. **Never stop the old channel before the new one exists.** If the new watch fails, the calendar has no
   channel, nothing renews it, and its busy time stops updating while the UI says "on".
4. **Never store tokens in clear text.** A refresh token is long-lived read and write access to a calendar.
5. **Never let a Google failure fail a booking.** Outbound writes log and return; the booking stands.
