# Channels and settings

A watch channel is Google's promise to POST to your URL when a calendar's events change. It expires,
it can fail to open, and nothing renews it automatically. This file covers opening, renewing and
stopping channels, and the settings changes that set them in motion.

## Channel lifecycle

| Moment | Action | Failure handling |
|---|---|---|
| Busy sync switched on | `startChannel`, then first `syncBusy` | Row keeps `syncBusy: true` and `lastSyncError`; the cron opens the channel |
| Every cron run | channel missing or expiring within `renewBeforeMs`: new channel, **then** stop old | Old channel kept; error recorded; retried next run |
| Every cron run | `syncBusy` on every busy calendar | Recorded on the row |
| Busy sync switched off | row unlinked, channel stopped, blocks deleted | Stop is best effort; the local record always goes |
| Calendar gone from Google, disconnect, other account | `teardownCalendar` | Same |
| Channel no row points at | stopped by the cron | Orphans from races or crashes do not live forever |

- **The token carries the channel id only, signed.** Google echoes it in `X-Goog-Channel-Token` and caps
  it at 256 characters. A token holding the calendar id, the user id and a signature can pass that with
  a long shared-calendar id, and then the watch call fails for that calendar only. Everything else is
  looked up from the stored channel.
- **Read the expiry Google returns.** The lifetime is Google's choice and not documented as a fixed
  number. With the defaults (renew within 48 h, cron every 12 h) a channel has four chances to renew.
- **New before old.** Google allows overlapping channels on one calendar. Opening the new one first
  means there is never a moment without one; the old id keeps delivering briefly and gets a 204.
- **A channel opened while busy sync was being switched off is stopped again.** `startChannel` points
  the row at the new channel only if `syncBusy` is still on, in the same compare-and-swap.
- **Holiday calendars cannot be watched.** They are synced by the cron only, which is plenty for data
  that changes yearly; health reports them as `polling`.

## Settings

`updateCalendarSettings` writes the row (one compare-and-swap, so concurrent toggles serialize), then
runs what the change implies:

| From, to | Transition | Effect |
|---|---|---|
| `syncBusy` off, on | `start-busy` | open channel, first sync |
| `syncBusy` on, off | `stop-busy` | stop channel, delete blocks |
| `syncBookings` off, on | `backfill-bookings` | write upcoming bookings of this host |
| `syncBookings` on, off | none | existing events stay |
| `title` | none | display only |

Guards: the calendar must belong to the caller (`calendar-not-found` otherwise, never "forbidden", so
ids cannot be probed), and booking sync needs `isWritable` (`calendar-not-writable`). A transition that
fails leaves the setting on and records the error; the cron completes it. Reverting the toggle instead
would turn a transient Google error into a setting the user has to find and fix.

## Calendar list

`refreshCalendarList` runs whenever the user opens the settings screen:

- **New calendars** get rows with both syncs off.
- **Existing rows** keep the user's settings and title, and take Google's current `accessRole`.
- **`writer` counts as writable**, like `owner`: a shared team calendar the user may edit is a
  legitimate target for bookings. Losing write access turns booking sync off.
- **Writability is correct on the first listing.** The settings screen can offer booking sync right after
  connecting; a row that starts non-writable until a second visit reads as a broken toggle.
- **Calendars Google no longer lists are torn down**: channel stopped, blocks deleted, row removed.
  Otherwise a calendar the user unsubscribed from keeps blocking their time and its channel keeps
  failing.
- **Errors propagate.** A revoked grant must reach the UI as `reauth-required`, not as an empty list
  that looks like "you have no calendars".

## Code

```ts
// calendar-sync/channels.ts
import { randomUUID } from 'node:crypto';

import { isHolidayCalendar } from './busyRules';
import { syncBusy } from './busySync';
import { signChannelToken } from './crypto';
import { withAccessToken } from './tokens';
import type { CalendarSyncDeps, LinkedCalendar, WatchChannel } from './types';

/**
 * Open a push channel on the calendar and point the row at it. Returns null for calendars Google
 * cannot watch, and when busy sync was switched off while the channel was being created (the new
 * channel is stopped again rather than left delivering to a calendar nobody syncs).
 */
export async function startChannel(
  deps: CalendarSyncDeps,
  calendar: LinkedCalendar
): Promise<WatchChannel | null> {
  if (isHolidayCalendar(calendar.externalId)) return null;

  const id = randomUUID();
  const response = await withAccessToken(deps, calendar.userId, accessToken =>
    deps.google.watchEvents(accessToken, calendar.externalId, {
      id,
      address: deps.config.notificationUrl,
      token: signChannelToken(id, deps.config.channelSecret),
    })
  );
  const channel: WatchChannel = {
    id,
    userId: calendar.userId,
    calendarId: calendar.id,
    resourceId: response.resourceId,
    // Read the expiry Google grants; never assume a lifetime.
    expiresAt: new Date(Number(response.expiration ?? 0)),
    createdAt: deps.now(),
  };
  await deps.store.saveChannel(channel);

  const { after } = await deps.store.mutateCalendar(calendar.id, current =>
    current?.syncBusy ? { ...current, channelId: id } : null
  );
  if (after?.channelId !== id) {
    await stopChannel(deps, channel);
    return null;
  }
  return channel;
}

/** Best effort at Google, always locally. A channel Google forgot about only costs a 204 later. */
export async function stopChannel(deps: CalendarSyncDeps, channel: WatchChannel): Promise<void> {
  try {
    await withAccessToken(deps, channel.userId, accessToken =>
      deps.google.stopChannel(accessToken, { id: channel.id, resourceId: channel.resourceId })
    );
  } catch (error) {
    deps.log.warn('google-calendar-channel-stop-failed', {
      channelId: channel.id,
      message: error instanceof Error ? error.message : String(error),
    });
  }
  await deps.store.deleteChannel(channel.id);
}

export async function enableBusySync(deps: CalendarSyncDeps, calendar: LinkedCalendar): Promise<void> {
  // Channel first: a change made while the first sync runs then still arrives as a notification.
  await startChannel(deps, calendar);
  await syncBusy(deps, calendar.id);
}

export async function disableBusySync(deps: CalendarSyncDeps, calendar: LinkedCalendar): Promise<void> {
  const { before } = await deps.store.mutateCalendar(calendar.id, current =>
    current ? { ...current, channelId: null, syncLeaseUntil: null, resyncRequested: false } : null
  );
  const channel = before?.channelId ? await deps.store.getChannel(before.channelId) : null;
  if (channel) await stopChannel(deps, channel);
  await deps.store.deleteBusyBlocks(calendar.id);
}

/** Stop the channel, delete the busy blocks and the row. Used on disconnect and when Google drops a calendar. */
export async function teardownCalendar(deps: CalendarSyncDeps, calendar: LinkedCalendar): Promise<void> {
  const channel = calendar.channelId ? await deps.store.getChannel(calendar.channelId) : null;
  if (channel) await stopChannel(deps, channel);
  await deps.store.deleteBusyBlocks(calendar.id);
  await deps.store.deleteCalendar(calendar.id);
}

async function mapPool<T>(items: T[], size: number, fn: (item: T) => Promise<void>): Promise<void> {
  let next = 0;
  const workers = Array.from({ length: Math.min(size, items.length) }, async () => {
    while (next < items.length) {
      const item = items[next++] as T;
      await fn(item);
    }
  });
  await Promise.all(workers);
}

export interface RenewReport {
  calendars: number;
  channelsRenewed: number;
  synced: number;
  failed: number;
  orphansStopped: number;
}

/**
 * The cron. For every calendar with busy sync: open a channel if it has none or it expires within
 * `renewBeforeMs` (new before old: Google allows overlapping channels, so there is no gap), then
 * resync, which moves the horizon forward and repairs any notification that was lost. Finally stops
 * channels no calendar points at. A failure is recorded on the row and retried on the next run.
 */
export async function renewChannels(deps: CalendarSyncDeps, concurrency = 8): Promise<RenewReport> {
  const report: RenewReport = { calendars: 0, channelsRenewed: 0, synced: 0, failed: 0, orphansStopped: 0 };
  const renewBy = new Date(deps.now().getTime() + deps.config.renewBeforeMs);

  let cursor: string | null = null;
  for (;;) {
    const page = await deps.store.listBusySyncCalendars(cursor, 100);
    if (page.length === 0) break;
    cursor = page[page.length - 1]?.id ?? null;
    report.calendars += page.length;

    await mapPool(page, concurrency, async calendar => {
      try {
        const current = calendar.channelId ? await deps.store.getChannel(calendar.channelId) : null;
        if (!isHolidayCalendar(calendar.externalId) && (!current || current.expiresAt <= renewBy)) {
          const fresh = await startChannel(deps, calendar);
          if (fresh) report.channelsRenewed++;
          if (fresh && current) await stopChannel(deps, current);
        }
        const outcome = await syncBusy(deps, calendar.id);
        if (outcome.status === 'synced') report.synced++;
      } catch (error) {
        report.failed++;
        const message = error instanceof Error ? error.message : String(error);
        await deps.store.mutateCalendar(calendar.id, row => (row ? { ...row, lastSyncError: message } : null));
        deps.log.error('google-calendar-renew-failed', { calendarId: calendar.id, message });
      }
    });
  }

  const expiring = await deps.store.listChannelsExpiringBefore(renewBy, 500);
  for (const channel of expiring) {
    const calendar = await deps.store.getCalendar(channel.calendarId);
    if (calendar?.syncBusy && calendar.channelId === channel.id) continue;
    await stopChannel(deps, channel);
    report.orphansStopped++;
  }

  await deps.store.deleteExpiredAuthRequests(deps.now());
  deps.log.info('google-calendar-renewed', { ...report });
  return report;
}
```

```ts
// calendar-sync/calendars.ts
import { backfillBookings } from './bookingEvents';
import { disableBusySync, enableBusySync, teardownCalendar } from './channels';
import { calendarRowId } from './crypto';
import type { GoogleCalendarListEntry } from './googleApi';
import { withAccessToken } from './tokens';
import {
  CalendarSyncError,
  type CalendarSettingsPatch,
  type CalendarSyncDeps,
  type LinkedCalendar,
} from './types';

/** Writers can insert events too; limiting this to owners hides shared team calendars. */
export function canWriteRole(role: GoogleCalendarListEntry['accessRole']): boolean {
  return role === 'owner' || role === 'writer';
}

/**
 * The row for one Google calendar: new rows start with both syncs off; existing rows keep the
 * user's settings and title, take the current access role, and lose booking sync if no longer writable.
 */
export function mergeCalendarRow(
  userId: string,
  entry: GoogleCalendarListEntry,
  current: LinkedCalendar | null,
  connectionCanWrite: boolean
): LinkedCalendar {
  const isWritable = connectionCanWrite && canWriteRole(entry.accessRole);
  if (current) {
    return {
      ...current,
      accessRole: entry.accessRole,
      isWritable,
      syncBookings: current.syncBookings && isWritable,
    };
  }
  return {
    id: calendarRowId(userId, entry.id),
    userId,
    externalId: entry.id,
    title: entry.summaryOverride ?? entry.summary ?? entry.id,
    accessRole: entry.accessRole,
    isWritable,
    syncBusy: false,
    syncBookings: false,
    channelId: null,
    lastSyncedAt: null,
    lastSyncError: null,
    syncLeaseUntil: null,
    resyncRequested: false,
    version: 0,
  };
}

function sameRow(a: LinkedCalendar, b: LinkedCalendar): boolean {
  return a.accessRole === b.accessRole && a.isWritable === b.isWritable && a.syncBookings === b.syncBookings;
}

/**
 * Bring the stored rows in line with the account's calendar list. Errors propagate: a revoked
 * connection must surface as `reauth-required`, never as an empty list.
 */
export async function refreshCalendarList(deps: CalendarSyncDeps, userId: string): Promise<LinkedCalendar[]> {
  const connection = await deps.store.getConnection(userId);
  if (!connection) throw new CalendarSyncError('not-connected');
  const entries = await withAccessToken(deps, userId, accessToken => deps.google.listCalendars(accessToken));

  const rows: LinkedCalendar[] = [];
  for (const entry of entries) {
    const { after } = await deps.store.mutateCalendar(calendarRowId(userId, entry.id), current => {
      const next = mergeCalendarRow(userId, entry, current, connection.canWrite);
      return current && sameRow(current, next) ? null : next;
    });
    if (after) rows.push(after);
  }

  const listed = new Set(rows.map(row => row.id));
  for (const stale of await deps.store.listCalendars(userId)) {
    if (!listed.has(stale.id)) await teardownCalendar(deps, stale);
  }
  return rows;
}

export type Transition = 'start-busy' | 'stop-busy' | 'backfill-bookings';

/** What a settings change sets in motion. Turning booking sync off leaves existing events alone. */
export function planTransitions(before: LinkedCalendar, after: LinkedCalendar): Transition[] {
  const transitions: Transition[] = [];
  if (!before.syncBusy && after.syncBusy) transitions.push('start-busy');
  if (before.syncBusy && !after.syncBusy) transitions.push('stop-busy');
  if (!before.syncBookings && after.syncBookings) transitions.push('backfill-bookings');
  return transitions;
}

export function applySettings(
  userId: string,
  current: LinkedCalendar | null,
  patch: CalendarSettingsPatch
): LinkedCalendar {
  if (!current || current.userId !== userId) throw new CalendarSyncError('calendar-not-found');
  if (patch.syncBookings === true && !current.isWritable) {
    throw new CalendarSyncError('calendar-not-writable');
  }
  return {
    ...current,
    ...(patch.title === undefined ? {} : { title: patch.title }),
    ...(patch.syncBusy === undefined ? {} : { syncBusy: patch.syncBusy }),
    ...(patch.syncBookings === undefined ? {} : { syncBookings: patch.syncBookings }),
  };
}

/**
 * Change one calendar's settings and run what the change implies. The row is written first and the
 * side effects follow; if one fails, the error is recorded and the cron finishes the job (it opens
 * missing channels and resyncs every calendar with busy sync).
 */
export async function updateCalendarSettings(
  deps: CalendarSyncDeps,
  userId: string,
  calendarId: string,
  patch: CalendarSettingsPatch
): Promise<LinkedCalendar> {
  const { before, after } = await deps.store.mutateCalendar(calendarId, current =>
    applySettings(userId, current, patch)
  );
  if (!before || !after) throw new CalendarSyncError('calendar-not-found');

  for (const transition of planTransitions(before, after)) {
    try {
      if (transition === 'start-busy') await enableBusySync(deps, after);
      if (transition === 'stop-busy') await disableBusySync(deps, after);
      if (transition === 'backfill-bookings') await backfillBookings(deps, after);
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      deps.log.error('google-calendar-transition-failed', { calendarId, transition, message });
      await deps.store.mutateCalendar(calendarId, row => (row ? { ...row, lastSyncError: message } : null));
      if (error instanceof CalendarSyncError) throw error;
    }
  }
  return (await deps.store.getCalendar(calendarId)) ?? after;
}
```

## Checklist

- [ ] Cron scheduled every 12 h (or more often than `renewBeforeMs / 2`)
- [ ] Renewal opens the new channel before stopping the old one
- [ ] Busy-sync toggle failures recorded, not reverted
- [ ] Calendar list refresh tears down calendars Google dropped
- [ ] Settings writes go through `mutateCalendar`, with the ownership check inside `fn`
