# Booking events (outbound)

A confirmed booking becomes an event in its primary host's booking-sync calendars; a cancelled one is
deleted. The event's Google id is a hash of the booking id, so there is no mapping table to keep in
step, and writing the same booking twice is one event.

## Hooks into the host's booking flow

| Booking change | Call | Notes |
|---|---|---|
| Confirmed (or paid, whichever the host calls "booked") | `pushBookingEvent(deps, booking)` | Primary host's calendars with `syncBookings` and `isWritable` |
| Cancelled, expired, refunded | `removeBookingEvent(deps, booking)` | Every host's `syncBookings` calendars |
| Rescheduled into a new booking | `removeBookingEvent(old)` then `pushBookingEvent(new)` | The new booking id gives a new event id |
| Time or details edited in place | `pushBookingEvent(deps, booking)` again | 409 turns into an update that replaces the body |
| Host turns booking sync on | `backfillBookings` (automatic, via settings) | `BookingSource.listUpcomingForPrimaryHost` |

Neither call throws. A Google outage, a revoked grant or a deleted calendar is logged and reported in
the returned results, and the booking flow carries on. Run them after the booking's own transaction
commits, never inside it: a retried transaction would call Google twice, and a slow Google call holds
the transaction open.

## The event

| Field | Value | Why |
|---|---|---|
| `id` | `bk` + 40 hex chars of `sha256("booking\n" + bookingId)` | Hex is a subset of base32hex; stable per booking; 409 on a repeat |
| `start` / `end` | `{ dateTime: instant.toISOString(), timeZone }` | Full instant with `Z`; the zone only affects display |
| `summary` / `description` | `describeBooking(booking)` | The host's wording and language |
| `attendees` | guest, then other hosts with an e-mail | One host without an address would make Google reject the event |
| `extendedProperties.private.bookingId` | the booking id | Marks it as ours, so busy sync skips it |
| `conferenceData.createRequest.requestId` | `booking-<bookingId>` | Only with `wantsGoogleMeet`; stable so retries do not create rooms |
| `status` | `confirmed` | Restores a previously deleted event when written by update |

- **Why private, not shared, extended properties.** Shared properties are copied to every attendee's
  copy of the event. If the guest also uses the app and syncs busy time, their copy would carry the
  marker and be skipped, and the guest would look free during their own appointment.
- **Why the `Z` matters.** Stripping `Z` from a UTC string and sending it with `timeZone` makes Google
  read 07:00 UTC as 07:00 in that zone. The event lands off by the zone's offset.
- **Meet links.** Google creates the conference asynchronously but returns `hangoutLink` on the insert
  response in the common case; `onConferenceLink(bookingId, url)` lets the host store it on the booking
  and put it in its own confirmation e-mail. If the link is missing, it is on the event a moment later
  and a later `pushBookingEvent` returns it.
- **Notifications.** Inserts and updates use `sendUpdates=none`: the host app sends its own
  confirmation. Deletes use `config.sendUpdatesOnCancel`, `all` by default, so Google tells attendees
  the event is gone; guests then get Google's cancellation alongside the app's own. Set `none` if that
  is one e-mail too many.
- **Only the primary host's calendars get the event.** Other hosts are attendees on it and receive it
  through Google. Deletion still walks every host, which covers a primary host that changed after the
  event was written.
- **Turning booking sync off leaves existing events in place.** Deleting a calendar's worth of events
  on a toggle is not reversible; removing them is a separate, explicit action if a host needs one.

## Code

```ts
// calendar-sync/bookingEvents.ts
import { BOOKING_MARKER } from './busyRules';
import { sha256Hex } from './crypto';
import { GoogleApiError, type GoogleEvent } from './googleApi';
import { withAccessToken } from './tokens';
import type { BookingForCalendar, CalendarSyncDeps, LinkedCalendar } from './types';

/**
 * The Google event id for a booking. Hex is a subset of base32hex (a to v, 0 to 9), the only alphabet
 * Google accepts, and 42 characters is inside its 5 to 1024 range. The same booking always maps to the
 * same id, so a retry or a second trigger hits 409 instead of creating a duplicate event.
 */
export function bookingEventId(bookingId: string): string {
  return `bk${sha256Hex(`booking\n${bookingId}`).slice(0, 40)}`;
}

export function defaultDescribeBooking(booking: BookingForCalendar): { summary: string; description: string } {
  const lines = [
    ...(booking.links ?? []).map(link => `${link.label}: ${link.url}`),
    ...(booking.detailsUrl ? [`Booking details: ${booking.detailsUrl}`] : []),
    ...(booking.notes ? ['', booking.notes] : []),
  ];
  return {
    summary: `${booking.guestName || 'Guest'} - ${booking.title}`,
    description: lines.join('\n').trim(),
  };
}

/** The event body for one host's calendar. Pure. */
export function buildBookingEvent(
  booking: BookingForCalendar,
  hostId: string,
  text: { summary: string; description: string }
): GoogleEvent {
  const attendees: NonNullable<GoogleEvent['attendees']> = [];
  if (booking.guestEmail) attendees.push({ email: booking.guestEmail, responseStatus: 'accepted' });
  for (const otherHost of booking.hostIds) {
    const email = booking.hostEmails[otherHost];
    // A host without an address would make Google reject the whole event.
    if (otherHost !== hostId && email) attendees.push({ email, responseStatus: 'accepted' });
  }

  return {
    id: bookingEventId(booking.bookingId),
    status: 'confirmed',
    summary: text.summary,
    description: text.description,
    // A full ISO instant with its Z, plus the zone for display. Stripping the Z from a UTC string
    // makes Google read it as wall clock in `timeZone`, shifting the event by the offset.
    start: { dateTime: booking.start.toISOString(), timeZone: booking.timeZone },
    end: { dateTime: booking.end.toISOString(), timeZone: booking.timeZone },
    attendees,
    // Private: only this calendar's copy carries it, so a guest who also uses this app still sees
    // the event as their own busy time.
    extendedProperties: { private: { [BOOKING_MARKER]: booking.bookingId } },
    ...(booking.wantsGoogleMeet
      ? { conferenceData: { createRequest: { requestId: `booking-${booking.bookingId}` } } }
      : {}),
  };
}

export interface EventWriteResult {
  calendarId: string;
  ok: boolean;
  error?: string;
}

async function writeToCalendar(
  deps: CalendarSyncDeps,
  booking: BookingForCalendar,
  calendar: LinkedCalendar
): Promise<EventWriteResult> {
  const describe = deps.describeBooking ?? defaultDescribeBooking;
  const event = buildBookingEvent(booking, calendar.userId, describe(booking));
  const conference = booking.wantsGoogleMeet ? 1 : 0;
  try {
    const written = await withAccessToken(deps, calendar.userId, async accessToken => {
      try {
        return await deps.google.insertEvent(accessToken, calendar.externalId, event, conference);
      } catch (error) {
        // 409: the id exists, from an earlier attempt or a deleted event. Google's advice is
        // update; status 'confirmed' in the body restores a cancelled one.
        if (!(error instanceof GoogleApiError) || error.status !== 409) throw error;
        return deps.google.updateEvent(accessToken, calendar.externalId, event.id as string, event, conference);
      }
    });
    if (booking.wantsGoogleMeet && written.hangoutLink && deps.onConferenceLink) {
      await deps.onConferenceLink(booking.bookingId, written.hangoutLink);
    }
    return { calendarId: calendar.id, ok: true };
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    deps.log.error('google-calendar-event-write-failed', {
      bookingId: booking.bookingId,
      calendarId: calendar.id,
      message,
    });
    return { calendarId: calendar.id, ok: false, error: message };
  }
}

/**
 * Booking confirmed (or rescheduled into a new booking): write it to every booking-sync calendar
 * of the primary host. Never throws: a Google outage must not fail the booking.
 */
export async function pushBookingEvent(
  deps: CalendarSyncDeps,
  booking: BookingForCalendar
): Promise<EventWriteResult[]> {
  const primaryHostId = booking.hostIds[0];
  if (!primaryHostId) return [];
  try {
    const calendars = (await deps.store.listCalendars(primaryHostId)).filter(c => c.syncBookings && c.isWritable);
    return await Promise.all(calendars.map(calendar => writeToCalendar(deps, booking, calendar)));
  } catch (error) {
    deps.log.error('google-calendar-push-failed', {
      bookingId: booking.bookingId,
      message: error instanceof Error ? error.message : String(error),
    });
    return [];
  }
}

/** Booking cancelled (or the old half of a reschedule): delete its event from every host's booking calendars. */
export async function removeBookingEvent(
  deps: CalendarSyncDeps,
  booking: Pick<BookingForCalendar, 'bookingId' | 'hostIds'>
): Promise<EventWriteResult[]> {
  const eventId = bookingEventId(booking.bookingId);
  const results: EventWriteResult[] = [];
  for (const hostId of booking.hostIds) {
    let calendars: LinkedCalendar[];
    try {
      calendars = (await deps.store.listCalendars(hostId)).filter(c => c.syncBookings);
    } catch {
      continue;
    }
    for (const calendar of calendars) {
      try {
        await withAccessToken(deps, hostId, accessToken =>
          deps.google.deleteEvent(accessToken, calendar.externalId, eventId, deps.config.sendUpdatesOnCancel)
        );
        results.push({ calendarId: calendar.id, ok: true });
      } catch (error) {
        // 404 never written here, 410 already deleted: both are the state we want.
        if (error instanceof GoogleApiError && (error.status === 404 || error.status === 410)) {
          results.push({ calendarId: calendar.id, ok: true });
          continue;
        }
        const message = error instanceof Error ? error.message : String(error);
        deps.log.error('google-calendar-event-delete-failed', { bookingId: booking.bookingId, message });
        results.push({ calendarId: calendar.id, ok: false, error: message });
      }
    }
  }
  return results;
}

/** Booking sync switched on: write the host's upcoming bookings. Idempotent through the event id. */
export async function backfillBookings(deps: CalendarSyncDeps, calendar: LinkedCalendar): Promise<number> {
  const bookings = await deps.bookings.listUpcomingForPrimaryHost(calendar.userId, deps.now());
  let written = 0;
  for (const booking of bookings) {
    const result = await writeToCalendar(deps, booking, calendar);
    if (result.ok) written++;
  }
  return written;
}
```

## Checklist

- [ ] `pushBookingEvent` / `removeBookingEvent` called after the booking transaction commits
- [ ] Rescheduling removes the old booking's event before pushing the new one
- [ ] `describeBooking` supplied in the host's language
- [ ] `onConferenceLink` stores the Meet URL where guests' e-mails read it
- [ ] No mapping table from booking to Google event id
