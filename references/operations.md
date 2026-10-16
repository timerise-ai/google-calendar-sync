# Operations

What an operator and a host must be able to see and do, the configuration, and the runbook for the
failures this integration actually has.

## Health, derived

Each calendar's health is computed on read from the connection, the row and its channel. Nothing stores
a status, so nothing can report "fine" because the job that would have updated it never ran.

| Health | Meaning | What fixes it |
|---|---|---|
| `reauth-required` | Google refused the refresh token | The user reconnects (same account keeps settings) |
| `sync-failing` | The last sync or transition recorded an error | Read `lastSyncError`; the next notification or cron run retries |
| `channel-expired` | The channel's expiry passed without renewal | The cron is not running or is failing |
| `no-channel` | Busy sync is on but no channel exists | Next cron run opens one |
| `stale` | Last successful sync older than `staleAfterMs` | The cron is not reaching this calendar |
| `never-synced` | Busy sync on, first sync not finished | Wait, or look at the logs |
| `polling` | Holiday calendar, synced by the cron only | Nothing |
| `ok` / `off` | Working / both syncs off | Nothing |

`HEALTH_SEVERITY` orders them for an admin list: worst first.

```ts
// calendar-sync/health.ts
import { isHolidayCalendar } from './busyRules';
import type { CalendarConnection, LinkedCalendar, WatchChannel } from './types';

export type CalendarHealth =
  | 'off'
  | 'ok'
  | 'polling'
  | 'reauth-required'
  | 'never-synced'
  | 'sync-failing'
  | 'no-channel'
  | 'channel-expired'
  | 'stale';

/** Problems first, in the order an operator should fix them. */
export const HEALTH_SEVERITY: Record<CalendarHealth, number> = {
  'reauth-required': 0,
  'sync-failing': 1,
  'channel-expired': 2,
  'no-channel': 3,
  stale: 4,
  'never-synced': 5,
  polling: 6,
  ok: 7,
  off: 8,
};

/**
 * Derived, never stored: a stored status needs a job to keep it true, and a failed job makes
 * everything look healthy. `staleAfterMs` should exceed the cron interval with margin.
 */
export function deriveCalendarHealth(input: {
  connection: CalendarConnection;
  calendar: LinkedCalendar;
  channel: WatchChannel | null;
  now: Date;
  staleAfterMs: number;
}): CalendarHealth {
  const { connection, calendar, channel, now, staleAfterMs } = input;
  if (connection.status === 'reauth_required') return 'reauth-required';
  if (!calendar.syncBusy) return calendar.syncBookings ? 'ok' : 'off';
  if (calendar.lastSyncError) return 'sync-failing';
  if (!isHolidayCalendar(calendar.externalId)) {
    if (!channel) return 'no-channel';
    if (channel.expiresAt <= now) return 'channel-expired';
  }
  if (!calendar.lastSyncedAt) return 'never-synced';
  if (now.getTime() - calendar.lastSyncedAt.getTime() > staleAfterMs) return 'stale';
  return isHolidayCalendar(calendar.externalId) ? 'polling' : 'ok';
}
```

## What each surface should show

| Surface | Shows | Lets you |
|---|---|---|
| Host's settings screen | account e-mail, connection status, each calendar with its two toggles, health, last synced | connect, reconnect, disconnect, toggle, rename |
| Admin view (per business) | every host's connection status and worst calendar health, sorted by `HEALTH_SEVERITY` | see who must reconnect before their availability is wrong |
| Logs | the `kind` values below | filter one failure class |

The reconnect prompt is the most important element on the host's screen: a `reauth_required`
connection silently stops blocking busy time, and the host's next double booking is how they find out.
Consider e-mailing the host when their connection flips to `reauth_required`.

## Log kinds

| Kind | Level | Meaning |
|---|---|---|
| `google-calendar-connected` / `-disconnected` | info | Connection lifecycle |
| `google-calendar-reauth-required` | warn | `invalid_grant` on refresh |
| `google-calendar-synced` | info | Per sync: events listed, upserted, deleted, skipped by reason |
| `google-calendar-sync-failed` | error | Sync aborted; nothing was deleted |
| `google-calendar-renewed` | info | Cron report |
| `google-calendar-renew-failed` | error | One calendar's renewal or resync failed |
| `google-calendar-channel-stop-failed` | warn | Usually harmless: Google already forgot the channel |
| `google-calendar-webhook-bad-token` / `-resource-mismatch` | warn | Forged or misrouted notification |
| `google-calendar-webhook-unknown-channel` | info | Old channel delivering after renewal |
| `google-calendar-event-write-failed` / `-delete-failed` | error | A booking's event was not written / not removed |
| `google-calendar-transition-failed` | error | A toggle's side effect failed; the cron completes it |

The `skipped` counts on `google-calendar-synced` answer most "why is this event not blocking?"
questions: `free`, `declined`, `own-booking`.

## Configuration

| Variable | Example | Notes |
|---|---|---|
| `GOOGLE_CALENDAR_CLIENT_ID` | `...apps.googleusercontent.com` | OAuth client, *Web application* |
| `GOOGLE_CALENDAR_CLIENT_SECRET` | | Secret manager |
| `GOOGLE_CALENDAR_REDIRECT_URI` | `https://api.example.com/calendar/google/callback` | Registered in the console, per environment |
| `GOOGLE_CALENDAR_NOTIFICATION_URL` | `https://api.example.com/calendar/google/notifications` | Public HTTPS, trusted certificate |
| `CALENDAR_CHANNEL_SECRET` | 32+ random bytes | Rotating it invalidates every open channel's token: delete all channel records, then run the cron, which opens fresh ones |
| `CALENDAR_TOKEN_KEY` | `openssl rand -hex 32` | Losing it means every user reconnects |
| `CALENDAR_RETURN_URL` | `https://app.example.com/settings/calendar` | Where the callback lands |
| `CRON_SECRET` | | Bearer for the cron route |

Schedule the cron every 12 hours (Cloud Scheduler, Vercel Cron, Kubernetes CronJob):

```bash
curl -X POST -H "Authorization: Bearer $CRON_SECRET" https://api.example.com/calendar/google/cron
```

Its run time grows with the number of busy calendars (one channel check and one full sync each, eight
at a time). Give the HTTP timeout room, or run it as a job rather than a request once you have
thousands of calendars.

## Runbook

| Symptom | Check | Fix |
|---|---|---|
| Every connection `reauth_required` a week after connecting | Consent screen publishing status | Publish the app; users reconnect once |
| Busy time stopped updating for everyone | `google-calendar-renewed` in the last 12 h | Restore the cron; the next run renews and resyncs all |
| One calendar's busy time is wrong | Its health and `lastSyncError`; `skipped` counts | Toggle busy sync off and on to rebuild it from scratch |
| Group session full after one booking | `own-booking` count is 0 while bookings exist | Outbound events lack the private marker; see provenance |
| Bookings not appearing in Google | `google-calendar-event-write-failed` | Usually `reauth-required` or a calendar no longer writable |
| No notifications locally | The notification URL is `localhost` | Expected: Google cannot reach it. Use a tunnel with a trusted certificate, or call the cron route by hand |

## Development

Google cannot deliver notifications to `localhost`. Either expose the API through an HTTPS tunnel and
set `GOOGLE_CALENDAR_NOTIFICATION_URL` to the tunnel's URL, or leave notifications off locally and run
the cron route by hand after changing an event; it resyncs every busy calendar. The OAuth redirect does
work on `http://localhost` for a *Web application* client, as long as that exact URI is registered.

## Optional extensions

- **Retry queue for outbound writes.** Failed `pushBookingEvent` results are logged and returned, not
  retried. A host that needs guarantees can store failed `(bookingId, calendarId)` pairs and replay them
  from the cron; the event id makes replays safe.
- **Incremental sync with `syncToken`.** Cuts quota for very large calendars, at the cost of handling
  `410 Gone` (full resync) and keeping the token per calendar. The full-window resync here is simpler and
  repairs itself.
