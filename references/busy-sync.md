# Busy sync (inbound)

A host's Google events become busy blocks the availability engine subtracts. The blocks are a mirror
of one window, `[now, now + horizonDays)`, rebuilt by listing every event in it and diffing against
what is stored. Nothing edits a block in place, so nothing can drift.

## Which events block

Evaluated in this order by `toBusyBlock`; the first match decides.

| Event | Blocks? | Why |
|---|---|---|
| no `id` | no (`no-id`) | Cannot be diffed |
| `status: 'cancelled'` | no | A deleted instance of a series |
| private `bookingId` marker | no (`own-booking`) | Written by this integration; the booking store already counts it |
| `eventType: 'workingLocation'` | no | "Working from home" is not a commitment |
| `transparency: 'transparent'` | no (`free`), except on holiday calendars | Shown as *Free* in Google; Google's own free/busy ignores it |
| own attendee `responseStatus: 'declined'` | no | The user is not going |
| all-day (`start.date`) | yes, `start.date` to `end.date` (exclusive) at local midnights | Multi-day events span every day; DST days are 23/25 h |
| timed (`start.dateTime`) | yes, the instants as given | They carry their offset |
| unparseable or `end <= start` | no (`invalid-time`) | Never write a block the engine cannot reason about |

`outOfOffice` and `focusTime` events are opaque by default and block, which is what their owners
mean by them.

**All-day events default to *Free* in Google Calendar**, so birthdays and reminders do not block, while
a vacation the user marked *Busy* blocks every day of it. Holiday calendars are the exception: whatever
their events' transparency, the only reason to enable one is to block those days.

**The own-booking skip is what keeps group sessions bookable.** The engine already subtracts each
booking's seats. If the event written for the first booking of a ten-seat class came back as a busy
block, the host would be busy for the whole slot and the other nine seats would vanish. Skipping by
the private marker leaves the guest's own copy (if the guest also uses the app) treated as theirs.

## The sync run

```
syncBusy(calendarId)
  lease: row.syncBusy? lease free? -- no lease --> set resyncRequested, return 'queued'
     | acquired
     v
  list every page of [now, now + horizon) -- any page fails --> record lastSyncError, release, throw (nothing deleted)
     v
  toBusyBlock each -> diffBusy(stored, desired) -> applyBusyDiff -> lastSyncedAt
     v
  release: resyncRequested? -- yes (max 3 runs) --> run again
```

- **Single flight per calendar.** Google sends notifications in bursts, often several for one edit.
  Without the lease each would list the calendar in parallel and the slowest, with the oldest data,
  would write last. With it, one run holds the lease and the rest set `resyncRequested`; the holder runs
  once more, so a change that arrived mid-run is still picked up.
- **Deterministic block ids make the diff idempotent.** Even if two runs overlapped (an expired lease,
  two deployments), both write the same ids and the result converges.
- **The diff iterates stored rows, not a map keyed by event id.** Rows with ids from an older scheme,
  or duplicates of one event, are not in the desired set and are deleted. A map would keep one of each
  duplicate and orphan the other forever.
- **Writes only on change.** A title edit in Google does not rewrite the block; a moved event does.
- **The horizon only moves when a sync runs.** A calendar nobody edits gets no notifications, so the
  cron resyncs every busy calendar ([channels-and-settings.md](channels-and-settings.md)). Set
  `horizonDays` at least as far out as bookings can be made.

## Feeding availability

Busy blocks belong to a person. The availability engine asks for a host's blocks overlapping a range
and treats them as host busy time (`kind: 'BLOCK'` in `time-slot-generator`):

```sql
select starts_at, ends_at
from calendar_busy_blocks
where user_id = $1 and ends_at > $2 and starts_at < $3;
```

Firestore: `listHostBusy` in [firestore-backend.md](firestore-backend.md). Blocks are UTC instants;
never convert them with a business or calendar zone on the way in.

## Code

```ts
// calendar-sync/busyRules.ts
import { busyBlockId } from './crypto';
import type { GoogleEvent } from './googleApi';
import type { BusyBlock } from './types';

/** The private extended property that marks an event this integration wrote for a booking. */
export const BOOKING_MARKER = 'bookingId';

export type SkipReason =
  | 'no-id'
  | 'cancelled'
  | 'free'
  | 'declined'
  | 'working-location'
  | 'own-booking'
  | 'invalid-time';

export type BusyDecision = { block: BusyBlock } | { skip: SkipReason };

export interface BusyContext {
  userId: string;
  calendarId: string;
  /** From the events list response; all-day events are anchored to it. */
  calendarTimeZone: string;
  /** False on holiday calendars: their events are usually marked free, and blocking is the point. */
  respectTransparency: boolean;
}

/** Holiday calendars (`...#holiday@group.v.calendar.google.com`) cannot be watched and are only enabled to block. */
export function isHolidayCalendar(externalId: string): boolean {
  return externalId.includes('#holiday');
}

function offsetMs(instant: Date, timeZone: string): number {
  const parts = new Intl.DateTimeFormat('en-US', {
    timeZone,
    hourCycle: 'h23',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  }).formatToParts(instant);
  const get = (type: Intl.DateTimeFormatPartTypes) => Number(parts.find(p => p.type === type)?.value);
  const wallAsUtc = Date.UTC(get('year'), get('month') - 1, get('day'), get('hour'), get('minute'), get('second'));
  return wallAsUtc - (instant.getTime() - instant.getUTCMilliseconds());
}

function localDateOf(instant: Date, timeZone: string): string {
  return new Intl.DateTimeFormat('en-CA', { timeZone, year: 'numeric', month: '2-digit', day: '2-digit' }).format(
    instant
  );
}

/**
 * The first instant of a local calendar date. Independent of the process TZ. Where midnight does
 * not exist (DST starting at 00:00, e.g. America/Santiago), it is the first instant that does.
 */
export function localMidnight(date: string, timeZone: string): Date {
  const [year = NaN, month = NaN, day = NaN] = date.split('-').map(Number);
  const guess = Date.UTC(year, month - 1, day);
  const first = guess - offsetMs(new Date(guess), timeZone);
  const second = guess - offsetMs(new Date(first), timeZone);
  return localDateOf(new Date(second), timeZone) === date ? new Date(second) : new Date(first);
}

export function addDays(date: string, days: number): string {
  const [year = NaN, month = NaN, day = NaN] = date.split('-').map(Number);
  return new Date(Date.UTC(year, month - 1, day + days)).toISOString().slice(0, 10);
}

/** Decide whether one Google event blocks the user, and when. Pure. */
export function toBusyBlock(event: GoogleEvent, ctx: BusyContext): BusyDecision {
  if (!event.id) return { skip: 'no-id' };
  if (event.status === 'cancelled') return { skip: 'cancelled' };
  // Our own booking, mirrored out. The booking store already counts it; importing it again would
  // block the remaining seats of a group booking.
  if (event.extendedProperties?.private?.[BOOKING_MARKER]) return { skip: 'own-booking' };
  if (event.eventType === 'workingLocation') return { skip: 'working-location' };
  if (ctx.respectTransparency && event.transparency === 'transparent') return { skip: 'free' };
  if (event.attendees?.some(a => a.self && a.responseStatus === 'declined')) return { skip: 'declined' };

  let from: Date;
  let to: Date;
  const allDay = Boolean(event.start?.date);
  if (event.start?.date) {
    // end.date is exclusive and spans every day of a multi-day event.
    const zone = event.start.timeZone ?? ctx.calendarTimeZone;
    from = localMidnight(event.start.date, zone);
    to = localMidnight(event.end?.date ?? addDays(event.start.date, 1), zone);
  } else {
    from = new Date(event.start?.dateTime ?? NaN);
    to = new Date(event.end?.dateTime ?? NaN);
  }
  if (Number.isNaN(from.getTime()) || Number.isNaN(to.getTime()) || to <= from) {
    return { skip: 'invalid-time' };
  }

  return {
    block: {
      id: busyBlockId(ctx.calendarId, event.id),
      userId: ctx.userId,
      calendarId: ctx.calendarId,
      eventId: event.id,
      from,
      to,
      allDay,
    },
  };
}

/**
 * What to write so the stored blocks equal `desired`. Every stored row not desired is deleted,
 * iterating rows rather than a map keyed by event, so duplicates left by older code are removed too.
 */
export function diffBusy(
  existing: BusyBlock[],
  desired: BusyBlock[]
): { upserts: BusyBlock[]; deletes: string[] } {
  const wanted = new Map(desired.map(block => [block.id, block]));
  const stored = new Map(existing.map(block => [block.id, block]));

  const deletes = existing.filter(block => !wanted.has(block.id)).map(block => block.id);
  const upserts = [...wanted.values()].filter(block => {
    const current = stored.get(block.id);
    return (
      !current ||
      current.from.getTime() !== block.from.getTime() ||
      current.to.getTime() !== block.to.getTime() ||
      current.allDay !== block.allDay
    );
  });
  return { upserts, deletes };
}
```

```ts
// calendar-sync/busySync.ts
import { diffBusy, isHolidayCalendar, toBusyBlock, type SkipReason } from './busyRules';
import type { GoogleEvent } from './googleApi';
import { withAccessToken } from './tokens';
import type { BusyBlock, CalendarSyncDeps, LinkedCalendar } from './types';

export type SyncOutcome =
  | { status: 'synced'; upserted: number; deleted: number; skipped: Partial<Record<SkipReason, number>> }
  | { status: 'queued' }
  | { status: 'disabled' };

/** Notifications that arrive during a sync queue one more run; this caps the reruns per lease. */
const MAX_RUNS_PER_LEASE = 3;

/**
 * Replace the calendar's busy blocks with what Google has now, inside [now, now + horizonDays].
 * Single-flight per calendar: a lease on the row, and a notification that arrives while it is held
 * sets `resyncRequested` so the holder runs once more instead of a second sync running alongside.
 */
export async function syncBusy(deps: CalendarSyncDeps, calendarId: string): Promise<SyncOutcome> {
  const now = deps.now();
  let acquired = false;
  const { after } = await deps.store.mutateCalendar(calendarId, current => {
    acquired = false;
    if (!current?.syncBusy) return null;
    if (current.syncLeaseUntil && current.syncLeaseUntil > now) {
      return current.resyncRequested ? null : { ...current, resyncRequested: true };
    }
    acquired = true;
    return {
      ...current,
      syncLeaseUntil: new Date(now.getTime() + deps.config.leaseMs),
      resyncRequested: false,
    };
  });
  if (!after?.syncBusy) return { status: 'disabled' };
  if (!acquired) return { status: 'queued' };

  let calendar: LinkedCalendar = after;
  let result: SyncOutcome = { status: 'disabled' };
  for (let run = 1; ; run++) {
    try {
      result = await runOnce(deps, calendar);
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      await deps.store.mutateCalendar(calendarId, current =>
        current ? { ...current, syncLeaseUntil: null, lastSyncError: message } : null
      );
      deps.log.error('google-calendar-sync-failed', { calendarId, userId: calendar.userId, message });
      throw error;
    }

    let rerun = false;
    const released = await deps.store.mutateCalendar(calendarId, current => {
      if (!current) return null;
      rerun = current.resyncRequested && current.syncBusy && run < MAX_RUNS_PER_LEASE;
      return {
        ...current,
        lastSyncedAt: deps.now(),
        lastSyncError: null,
        resyncRequested: false,
        syncLeaseUntil: rerun ? new Date(deps.now().getTime() + deps.config.leaseMs) : null,
      };
    });
    if (!rerun || !released.after) return result;
    calendar = released.after;
  }
}

async function runOnce(deps: CalendarSyncDeps, calendar: LinkedCalendar): Promise<SyncOutcome> {
  const timeMin = deps.now();
  const timeMax = new Date(timeMin.getTime() + deps.config.horizonDays * 86_400_000);

  // Every page, or nothing: a partial list would delete the blocks of every event it missed.
  const { events, timeZone } = await withAccessToken(deps, calendar.userId, async accessToken => {
    const all: GoogleEvent[] = [];
    let zone = 'UTC';
    let pageToken: string | undefined;
    do {
      const page = await deps.google.listEvents(accessToken, calendar.externalId, {
        timeMin,
        timeMax,
        pageToken,
      });
      all.push(...page.items);
      zone = page.timeZone ?? zone;
      pageToken = page.nextPageToken;
    } while (pageToken);
    return { events: all, timeZone: zone };
  });

  const desired: BusyBlock[] = [];
  const skipped: Partial<Record<SkipReason, number>> = {};
  for (const event of events) {
    const decision = toBusyBlock(event, {
      userId: calendar.userId,
      calendarId: calendar.id,
      calendarTimeZone: timeZone,
      respectTransparency: !isHolidayCalendar(calendar.externalId),
    });
    if ('block' in decision) desired.push(decision.block);
    else skipped[decision.skip] = (skipped[decision.skip] ?? 0) + 1;
  }

  const existing = await deps.store.listBusyBlocks(calendar.id);
  const { upserts, deletes } = diffBusy(existing, desired);
  if (upserts.length > 0 || deletes.length > 0) {
    await deps.store.applyBusyDiff(calendar.id, upserts, deletes);
  }
  deps.log.info('google-calendar-synced', {
    calendarId: calendar.id,
    events: events.length,
    upserted: upserts.length,
    deleted: deletes.length,
    skipped,
  });
  return { status: 'synced', upserted: upserts.length, deleted: deletes.length, skipped };
}
```

## Checklist

- [ ] Every page read before any delete
- [ ] Events carrying the private `bookingId` marker skipped
- [ ] All-day events span to `end.date`, anchored to the calendar's zone from the list response
- [ ] `horizonDays` at least the booking window
- [ ] Busy blocks read by user id and treated as host busy time, in UTC
