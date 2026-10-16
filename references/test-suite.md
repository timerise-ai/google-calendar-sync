# Test suite

32 tests in two files. How to run them, what each pins and the doubles they use: [testing.md](testing.md).

## Rules (pure functions)

```ts
// calendar-sync/rules.test.ts
import assert from 'node:assert/strict';
import { test } from 'node:test';

import { bookingEventId, buildBookingEvent, defaultDescribeBooking } from './bookingEvents';
import { BOOKING_MARKER, diffBusy, localMidnight, toBusyBlock, type BusyContext } from './busyRules';
import { applySettings, mergeCalendarRow, planTransitions } from './calendars';
import { decrypt, encrypt, signChannelToken, verifyChannelToken } from './crypto';
import { deriveCalendarHealth } from './health';
import type { BookingForCalendar, BusyBlock, CalendarConnection, LinkedCalendar } from './types';

const ctx: BusyContext = {
  userId: 'u1',
  calendarId: 'cal1',
  calendarTimeZone: 'Europe/Warsaw',
  respectTransparency: true,
};
const block = (decision: ReturnType<typeof toBusyBlock>) => {
  assert.ok('block' in decision, `expected a block, got ${JSON.stringify(decision)}`);
  return decision.block;
};

test('timed event keeps its instants', () => {
  const b = block(
    toBusyBlock(
      { id: 'e1', start: { dateTime: '2026-10-20T09:00:00+02:00' }, end: { dateTime: '2026-10-20T10:30:00+02:00' } },
      ctx
    )
  );
  assert.equal(b.from.toISOString(), '2026-10-20T07:00:00.000Z');
  assert.equal(b.to.toISOString(), '2026-10-20T08:30:00.000Z');
  assert.equal(b.allDay, false);
});

test('a multi-day all-day event blocks every day, not only the first', () => {
  const b = block(toBusyBlock({ id: 'vacation', start: { date: '2026-08-03' }, end: { date: '2026-08-08' } }, ctx));
  assert.equal(b.from.toISOString(), '2026-08-02T22:00:00.000Z');
  assert.equal(b.to.toISOString(), '2026-08-07T22:00:00.000Z');
  assert.equal(b.allDay, true);
});

test('an all-day event on the spring-forward day is 23 hours, midnight to midnight', () => {
  const b = block(toBusyBlock({ id: 'dst', start: { date: '2026-03-29' }, end: { date: '2026-03-30' } }, ctx));
  assert.equal(b.from.toISOString(), '2026-03-28T23:00:00.000Z');
  assert.equal(b.to.toISOString(), '2026-03-29T22:00:00.000Z');
});

test('localMidnight falls on the first instant of a day whose midnight does not exist', () => {
  // Chile starts DST at 00:00 on 2026-09-06: the day begins at 01:00 local (-03).
  assert.equal(localMidnight('2026-09-06', 'America/Santiago').toISOString(), '2026-09-06T04:00:00.000Z');
  assert.equal(localMidnight('2026-10-17', 'Pacific/Auckland').toISOString(), '2026-10-16T11:00:00.000Z');
  assert.equal(localMidnight('2026-11-01', 'America/Los_Angeles').toISOString(), '2026-11-01T07:00:00.000Z');
});

test('events this integration wrote for bookings are not imported back as busy time', () => {
  const decision = toBusyBlock(
    {
      id: 'bkabc',
      start: { dateTime: '2026-10-20T09:00:00Z' },
      end: { dateTime: '2026-10-20T10:00:00Z' },
      extendedProperties: { private: { [BOOKING_MARKER]: 'booking-1' } },
    },
    ctx
  );
  assert.deepEqual(decision, { skip: 'own-booking' });
});

test('free, declined, cancelled and working-location events do not block', () => {
  const times = { start: { dateTime: '2026-10-20T09:00:00Z' }, end: { dateTime: '2026-10-20T10:00:00Z' } };
  assert.deepEqual(toBusyBlock({ id: 'a', ...times, transparency: 'transparent' }, ctx), { skip: 'free' });
  assert.deepEqual(
    toBusyBlock({ id: 'b', ...times, attendees: [{ email: 'me@x', self: true, responseStatus: 'declined' }] }, ctx),
    { skip: 'declined' }
  );
  assert.deepEqual(toBusyBlock({ id: 'c', ...times, status: 'cancelled' }, ctx), { skip: 'cancelled' });
  assert.deepEqual(toBusyBlock({ id: 'd', ...times, eventType: 'workingLocation' }, ctx), {
    skip: 'working-location',
  });
  assert.deepEqual(toBusyBlock({ id: 'e', start: { dateTime: 'garbage' }, end: {} }, ctx), { skip: 'invalid-time' });
});

test('holiday calendars block even though their events are marked free', () => {
  const event = { id: 'h', start: { date: '2026-12-25' }, end: { date: '2026-12-26' }, transparency: 'transparent' as const };
  assert.ok('block' in toBusyBlock(event, { ...ctx, respectTransparency: false }));
});

test('diffBusy deletes duplicates left by older code and writes only what changed', () => {
  const at = (h: number) => new Date(Date.UTC(2026, 9, 20, h));
  const desired: BusyBlock[] = [
    { id: 'k1', userId: 'u1', calendarId: 'cal1', eventId: 'e1', from: at(9), to: at(10), allDay: false },
    { id: 'k2', userId: 'u1', calendarId: 'cal1', eventId: 'e2', from: at(12), to: at(13), allDay: false },
  ];
  const existing: BusyBlock[] = [
    { ...desired[0]! },
    { ...desired[1]!, from: at(11) }, // moved in Google
    { ...desired[0]!, id: 'legacy-random-id' }, // duplicate of e1 under another id
  ];
  const { upserts, deletes } = diffBusy(existing, desired);
  assert.deepEqual(upserts.map(b => b.id), ['k2']);
  assert.deepEqual(deletes, ['legacy-random-id']);
  assert.deepEqual(diffBusy(desired, desired), { upserts: [], deletes: [] });
});

test('tokens: encryption round-trips and rejects tampering; channel tokens bind to their channel', () => {
  const key = 'a'.repeat(64);
  const sealed = encrypt('{"refreshToken":"r"}', key);
  assert.equal(decrypt(sealed, key), '{"refreshToken":"r"}');
  const parts = sealed.split(':');
  parts[2] = Buffer.from('tampered').toString('base64');
  assert.throws(() => decrypt(parts.join(':'), key));

  const token = signChannelToken('6f1c2c4e-8a3b-4d0e-9a51-1f2e3d4c5b6a', 'secret');
  assert.ok(token.length <= 256);
  assert.equal(verifyChannelToken(token, 'secret'), '6f1c2c4e-8a3b-4d0e-9a51-1f2e3d4c5b6a');
  assert.equal(verifyChannelToken(token, 'other-secret'), null);
  assert.equal(verifyChannelToken(token.replace('6f1c', '0000'), 'secret'), null);
  assert.equal(verifyChannelToken(token, ''), null);
  assert.throws(() => signChannelToken('x', ''));
});

const booking: BookingForCalendar = {
  bookingId: 'booking/with odd chars',
  start: new Date('2026-10-20T07:00:00Z'),
  end: new Date('2026-10-20T08:00:00Z'),
  timeZone: 'Europe/Warsaw',
  title: 'Haircut',
  guestName: 'Ada',
  guestEmail: 'ada@example.com',
  hostIds: ['h1', 'h2', 'h3'],
  hostEmails: { h1: 'h1@example.com', h2: 'h2@example.com' },
};

test('booking event ids are valid Google ids and stable per booking', () => {
  const id = bookingEventId(booking.bookingId);
  assert.match(id, /^[a-v0-9]{5,1024}$/);
  assert.equal(id, bookingEventId(booking.bookingId));
  assert.notEqual(id, bookingEventId('another'));
});

test('booking event: instants keep their Z, the marker is private, hosts without e-mail are left out', () => {
  const event = buildBookingEvent(booking, 'h1', defaultDescribeBooking(booking));
  assert.equal(event.start?.dateTime, '2026-10-20T07:00:00.000Z');
  assert.equal(event.start?.timeZone, 'Europe/Warsaw');
  assert.equal(event.extendedProperties?.private?.[BOOKING_MARKER], booking.bookingId);
  assert.equal(event.extendedProperties?.shared, undefined);
  assert.deepEqual(event.attendees?.map(a => a.email), ['ada@example.com', 'h2@example.com']);
  assert.equal(event.conferenceData, undefined);
});

const google = (accessRole: 'owner' | 'writer' | 'reader') => ({ id: 'team@group.calendar.google.com', summary: 'Team', accessRole });

test('calendar rows: writers are writable, and a new row is writable on its first listing', () => {
  const row = mergeCalendarRow('u1', google('writer'), null, true);
  assert.equal(row.isWritable, true);
  assert.equal(mergeCalendarRow('u1', google('reader'), null, true).isWritable, false);
  assert.equal(mergeCalendarRow('u1', google('owner'), null, false).isWritable, false);
});

test('calendar rows: losing write access drops booking sync but keeps busy sync and the title', () => {
  const current: LinkedCalendar = {
    ...mergeCalendarRow('u1', google('owner'), null, true),
    title: 'Renamed',
    syncBusy: true,
    syncBookings: true,
  };
  const next = mergeCalendarRow('u1', google('reader'), current, true);
  assert.equal(next.syncBookings, false);
  assert.equal(next.syncBusy, true);
  assert.equal(next.title, 'Renamed');
});

test('settings: transitions and guards', () => {
  const base = mergeCalendarRow('u1', google('reader'), null, true);
  assert.throws(() => applySettings('u1', base, { syncBookings: true }), /calendar-not-writable/);
  assert.throws(() => applySettings('someone-else', base, { syncBusy: true }), /calendar-not-found/);
  const on = applySettings('u1', base, { syncBusy: true });
  assert.deepEqual(planTransitions(base, on), ['start-busy']);
  assert.deepEqual(planTransitions(on, base), ['stop-busy']);
  const writable = mergeCalendarRow('u1', google('owner'), null, true);
  assert.deepEqual(planTransitions(writable, applySettings('u1', writable, { syncBookings: true })), [
    'backfill-bookings',
  ]);
  assert.deepEqual(planTransitions(applySettings('u1', writable, { syncBookings: true }), writable), []);
});

test('health is derived, problems first', () => {
  const now = new Date('2026-10-20T12:00:00Z');
  const connection = { status: 'connected' } as CalendarConnection;
  const calendar: LinkedCalendar = {
    ...mergeCalendarRow('u1', google('owner'), null, true),
    syncBusy: true,
    channelId: 'c1',
    lastSyncedAt: new Date('2026-10-20T06:00:00Z'),
  };
  const channel = { id: 'c1', userId: 'u1', calendarId: calendar.id, resourceId: 'r', createdAt: now, expiresAt: new Date('2026-10-25T00:00:00Z') };
  const day = 26 * 3_600_000;
  assert.equal(deriveCalendarHealth({ connection, calendar, channel, now, staleAfterMs: day }), 'ok');
  assert.equal(deriveCalendarHealth({ connection, calendar, channel: null, now, staleAfterMs: day }), 'no-channel');
  assert.equal(
    deriveCalendarHealth({ connection, calendar, channel: { ...channel, expiresAt: now }, now, staleAfterMs: day }),
    'channel-expired'
  );
  assert.equal(
    deriveCalendarHealth({ connection: { ...connection, status: 'reauth_required' }, calendar, channel, now, staleAfterMs: day }),
    'reauth-required'
  );
  assert.equal(
    deriveCalendarHealth({ connection, calendar: { ...calendar, lastSyncedAt: new Date('2026-10-18T00:00:00Z') }, channel, now, staleAfterMs: day }),
    'stale'
  );
});
```

## Flows (against the doubles)

```ts
// calendar-sync/flows.test.ts
import assert from 'node:assert/strict';
import { test } from 'node:test';

import { bookingEventId, pushBookingEvent, removeBookingEvent } from './bookingEvents';
import { syncBusy } from './busySync';
import { refreshCalendarList, updateCalendarSettings } from './calendars';
import { renewChannels } from './channels';
import { completeConnect, disconnect, startConnect } from './connect';
import { signChannelToken } from './crypto';
import { SCOPE_READ } from './googleApi';
import { createFakeGoogle } from './testing/fakeGoogle';
import { createMemoryStore } from './testing/memoryStore';
import { withAccessToken } from './tokens';
import { handleNotification } from './webhook';
import { DEFAULT_SYNC_SETTINGS, type BookingForCalendar, type CalendarSyncDeps } from './types';

const USER = 'user-1';
const PRIMARY = 'host@example.com';

function setup(options: { pageSize?: number } = {}) {
  const memory = createMemoryStore();
  const fake = createFakeGoogle(options);
  fake.state.calendarList = [
    { id: PRIMARY, summary: 'Host', accessRole: 'owner', primary: true },
    { id: 'pl.polish#holiday@group.v.calendar.google.com', summary: 'Holidays', accessRole: 'reader' },
  ];
  const clock = { now: new Date('2026-10-16T08:00:00Z') };
  const upcoming: BookingForCalendar[] = [];
  const logs: string[] = [];
  const deps: CalendarSyncDeps = {
    store: memory.store,
    google: fake.api,
    config: {
      ...DEFAULT_SYNC_SETTINGS,
      clientId: 'id',
      clientSecret: 'secret',
      redirectUri: 'https://api.example/calendar/google/callback',
      notificationUrl: 'https://api.example/calendar/google/notifications',
      channelSecret: 'channel-secret',
      encryptionKey: 'b'.repeat(64),
    },
    bookings: { listUpcomingForPrimaryHost: async () => upcoming },
    log: {
      info: kind => logs.push(kind),
      warn: kind => logs.push(kind),
      error: kind => logs.push(kind),
    },
    now: () => clock.now,
  };
  return { deps, memory, fake, clock, upcoming, logs };
}

async function connect(env: ReturnType<typeof setup>) {
  const url = await startConnect(env.deps, USER);
  const state = new URL(url).searchParams.get('state') ?? '';
  return completeConnect(env.deps, { state, code: 'code' });
}

async function connectWithBusySync(env: ReturnType<typeof setup>) {
  await connect(env);
  const [primary] = (await refreshCalendarList(env.deps, USER)).filter(c => c.externalId === PRIMARY);
  assert.ok(primary);
  return updateCalendarSettings(env.deps, USER, primary.id, { syncBusy: true });
}

const timed = (id: string, startIso: string, endIso: string) => ({
  id,
  start: { dateTime: startIso },
  end: { dateTime: endIso },
});

test('connect: the state is single use, the user comes from the server, tokens are stored encrypted', async () => {
  const env = setup();
  const url = await startConnect(env.deps, USER);
  const state = new URL(url).searchParams.get('state') ?? '';
  const result = await completeConnect(env.deps, { state, code: 'code' });
  assert.equal(result.userId, USER);
  await assert.rejects(completeConnect(env.deps, { state, code: 'code' }), /invalid-or-expired-state/);
  await assert.rejects(completeConnect(env.deps, { state: '../users/x', code: 'code' }), /invalid-or-expired-state/);
  const stored = env.memory.connections.get(USER);
  assert.ok(stored && !stored.encryptedTokens.includes('refresh-'));
});

test('connect: an expired state is refused', async () => {
  const env = setup();
  const url = await startConnect(env.deps, USER);
  env.clock.now = new Date(env.clock.now.getTime() + 11 * 60_000);
  const state = new URL(url).searchParams.get('state') ?? '';
  await assert.rejects(completeConnect(env.deps, { state, code: 'code' }), /invalid-or-expired-state/);
});

test('connect: without the events scope nothing is writable', async () => {
  const env = setup();
  env.fake.state.grantedScopes = [SCOPE_READ];
  await connect(env);
  const calendars = await refreshCalendarList(env.deps, USER);
  assert.ok(calendars.every(c => !c.isWritable));
});

test('busy sync imports every page, not only the first 2500 events', async () => {
  const env = setup({ pageSize: 2 });
  const events = env.fake.eventsOf(PRIMARY);
  for (let i = 0; i < 5; i++) {
    const id = `ev${i}`;
    events.set(id, timed(id, `2026-10-2${i}T09:00:00Z`, `2026-10-2${i}T10:00:00Z`));
  }
  await connectWithBusySync(env);
  assert.equal(env.memory.busy.size, 5);
});

test('a failed page deletes nothing', async () => {
  const env = setup({ pageSize: 2 });
  const events = env.fake.eventsOf(PRIMARY);
  for (let i = 0; i < 4; i++) events.set(`ev${i}`, timed(`ev${i}`, `2026-10-2${i}T09:00:00Z`, `2026-10-2${i}T10:00:00Z`));
  const calendar = await connectWithBusySync(env);
  assert.equal(env.memory.busy.size, 4);

  env.fake.state.failPage = 1;
  await assert.rejects(syncBusy(env.deps, calendar.id));
  assert.equal(env.memory.busy.size, 4);
  assert.match(env.memory.calendars.get(calendar.id)?.lastSyncError ?? '', /Backend Error/);
});

test('a burst of notifications converges without duplicate blocks and still catches the last change', async () => {
  const env = setup();
  const events = env.fake.eventsOf(PRIMARY);
  events.set('a', timed('a', '2026-10-20T09:00:00Z', '2026-10-20T10:00:00Z'));
  const calendar = await connectWithBusySync(env);
  const channel = env.memory.calendars.get(calendar.id)?.channelId ?? '';
  const headers = {
    channelId: channel,
    channelToken: signChannelToken(channel, 'channel-secret'),
    resourceState: 'exists',
    resourceId: `res-${PRIMARY}`,
  };

  events.set('b', timed('b', '2026-10-21T09:00:00Z', '2026-10-21T10:00:00Z'));
  const burst = Promise.all([1, 2, 3, 4, 5].map(() => handleNotification(env.deps, headers)));
  // Changed while the first sync is running: the queued rerun must see it.
  events.set('c', timed('c', '2026-10-22T09:00:00Z', '2026-10-22T10:00:00Z'));
  assert.deepEqual(await burst, [204, 204, 204, 204, 204]);

  assert.deepEqual([...env.memory.busy.values()].map(b => b.eventId).sort(), ['a', 'b', 'c']);
  assert.equal(env.memory.calendars.get(calendar.id)?.syncLeaseUntil, null);
});

test('webhook: handshake, forged token, foreign resource, unknown channel', async () => {
  const env = setup();
  const calendar = await connectWithBusySync(env);
  const channel = env.memory.calendars.get(calendar.id)?.channelId ?? '';
  const good = {
    channelId: channel,
    channelToken: signChannelToken(channel, 'channel-secret'),
    resourceState: 'exists',
    resourceId: `res-${PRIMARY}`,
  };
  assert.equal(await handleNotification(env.deps, { ...good, resourceState: 'sync' }), 200);
  assert.equal(await handleNotification(env.deps, { ...good, channelToken: signChannelToken(channel, 'guess') }), 401);
  assert.equal(await handleNotification(env.deps, { ...good, channelId: 'other' }), 401);
  assert.equal(await handleNotification(env.deps, { ...good, resourceId: 'res-elsewhere' }), 400);
  const gone = 'f00dfeed-0000-4000-8000-000000000000';
  assert.equal(
    await handleNotification(env.deps, { ...good, channelId: gone, channelToken: signChannelToken(gone, 'channel-secret') }),
    204
  );
});

test('renewal opens the new channel before stopping the old one, and keeps the old one if that fails', async () => {
  const env = setup();
  const calendar = await connectWithBusySync(env);
  const first = env.memory.calendars.get(calendar.id)?.channelId;
  env.clock.now = new Date('2026-10-21T08:00:00Z'); // channel expires 2026-10-23, inside the 48 h window

  env.fake.state.watchFails = true;
  const failed = await renewChannels(env.deps);
  assert.equal(failed.failed, 1);
  assert.equal(env.memory.calendars.get(calendar.id)?.channelId, first);
  assert.ok(first && env.memory.channels.has(first));

  env.fake.state.watchFails = false;
  const report = await renewChannels(env.deps);
  const second = env.memory.calendars.get(calendar.id)?.channelId;
  assert.equal(report.channelsRenewed, 1);
  assert.notEqual(second, first);
  assert.ok(first && !env.memory.channels.has(first));
  assert.ok(env.fake.state.calls.indexOf(`stop:${first}`) > env.fake.state.calls.lastIndexOf(`watch:${PRIMARY}`));
});

test('renewal heals a busy calendar that lost its channel', async () => {
  const env = setup();
  const calendar = await connectWithBusySync(env);
  const channelId = env.memory.calendars.get(calendar.id)?.channelId ?? '';
  env.memory.channels.delete(channelId);
  await renewChannels(env.deps);
  const healed = env.memory.calendars.get(calendar.id)?.channelId;
  assert.ok(healed && healed !== channelId && env.memory.channels.has(healed));
});

test('turning busy sync off stops the channel and removes the blocks', async () => {
  const env = setup();
  env.fake.eventsOf(PRIMARY).set('a', timed('a', '2026-10-20T09:00:00Z', '2026-10-20T10:00:00Z'));
  const calendar = await connectWithBusySync(env);
  assert.equal(env.memory.busy.size, 1);
  await updateCalendarSettings(env.deps, USER, calendar.id, { syncBusy: false });
  assert.equal(env.memory.busy.size, 0);
  assert.equal(env.memory.channels.size, 0);
  assert.equal(env.fake.state.channels.size, 0);
});

const booking: BookingForCalendar = {
  bookingId: 'b-1',
  start: new Date('2026-10-20T07:00:00Z'),
  end: new Date('2026-10-20T08:00:00Z'),
  timeZone: 'Europe/Warsaw',
  title: 'Yoga class',
  guestEmail: 'guest@example.com',
  hostIds: [USER],
  hostEmails: { [USER]: PRIMARY },
};

test('outbound: a booking pushed twice is one event; its mirror never becomes busy time', async () => {
  const env = setup();
  const calendar = await connectWithBusySync(env);
  await updateCalendarSettings(env.deps, USER, calendar.id, { syncBookings: true });

  assert.deepEqual((await pushBookingEvent(env.deps, booking)).map(r => r.ok), [true]);
  assert.deepEqual((await pushBookingEvent(env.deps, booking)).map(r => r.ok), [true]);
  const events = [...env.fake.eventsOf(PRIMARY).values()];
  assert.equal(events.length, 1);
  assert.equal(events[0]?.id, bookingEventId('b-1'));

  await syncBusy(env.deps, calendar.id);
  assert.equal(env.memory.busy.size, 0);
});

test('outbound: cancelling deletes the event, twice is harmless, re-confirming restores it', async () => {
  const env = setup();
  await connect(env);
  const [primary] = (await refreshCalendarList(env.deps, USER)).filter(c => c.externalId === PRIMARY);
  assert.ok(primary);
  await updateCalendarSettings(env.deps, USER, primary.id, { syncBookings: true });
  await pushBookingEvent(env.deps, booking);

  assert.deepEqual((await removeBookingEvent(env.deps, booking)).map(r => r.ok), [true]);
  assert.deepEqual((await removeBookingEvent(env.deps, booking)).map(r => r.ok), [true]);
  assert.equal(env.fake.eventsOf(PRIMARY).get(bookingEventId('b-1'))?.status, 'cancelled');

  await pushBookingEvent(env.deps, booking);
  assert.equal(env.fake.eventsOf(PRIMARY).get(bookingEventId('b-1'))?.status, 'confirmed');
});

test('outbound: turning booking sync on backfills upcoming bookings', async () => {
  const env = setup();
  env.upcoming.push(booking, { ...booking, bookingId: 'b-2' });
  await connect(env);
  const [primary] = (await refreshCalendarList(env.deps, USER)).filter(c => c.externalId === PRIMARY);
  assert.ok(primary);
  await updateCalendarSettings(env.deps, USER, primary.id, { syncBookings: true });
  assert.equal(env.fake.eventsOf(PRIMARY).size, 2);
});

test('a revoked grant marks the connection, surfaces as reauth-required, and reconnecting keeps settings', async () => {
  const env = setup();
  const calendar = await connectWithBusySync(env);
  env.clock.now = new Date(env.clock.now.getTime() + 2 * 3_600_000); // access token expired
  env.fake.state.refreshFails = 'invalid_grant';

  await assert.rejects(withAccessToken(env.deps, USER, async () => 'never'), /reauth-required/);
  assert.equal(env.memory.connections.get(USER)?.status, 'reauth_required');
  await assert.rejects(refreshCalendarList(env.deps, USER), /reauth-required/);

  env.fake.state.refreshFails = null;
  const result = await connect(env);
  assert.equal(result.sameAccount, true);
  assert.equal(env.memory.connections.get(USER)?.status, 'connected');
  const after = env.memory.calendars.get(calendar.id);
  assert.equal(after?.syncBusy, true);
  assert.ok(after?.channelId && after.channelId !== calendar.channelId);
});

test('connecting a different Google account tears the old one down', async () => {
  const env = setup();
  env.fake.eventsOf(PRIMARY).set('a', timed('a', '2026-10-20T09:00:00Z', '2026-10-20T10:00:00Z'));
  await connectWithBusySync(env);
  env.fake.state.account = 'other@example.com';
  const result = await connect(env);
  assert.equal(result.sameAccount, false);
  assert.equal(env.memory.calendars.size, 0);
  assert.equal(env.memory.busy.size, 0);
  assert.equal(env.memory.channels.size, 0);
});

test('disconnect revokes and forgets everything', async () => {
  const env = setup();
  env.fake.eventsOf(PRIMARY).set('a', timed('a', '2026-10-20T09:00:00Z', '2026-10-20T10:00:00Z'));
  await connectWithBusySync(env);
  await disconnect(env.deps, USER);
  assert.ok(env.fake.state.calls.includes('revoke'));
  assert.equal(env.memory.connections.size, 0);
  assert.equal(env.memory.calendars.size, 0);
  assert.equal(env.memory.busy.size, 0);
  assert.equal(env.fake.state.channels.size, 0);
});

test('a calendar removed in Google is torn down on the next listing', async () => {
  const env = setup();
  await connectWithBusySync(env);
  env.fake.state.calendarList = env.fake.state.calendarList.filter(c => c.id !== PRIMARY);
  const rows = await refreshCalendarList(env.deps, USER);
  assert.deepEqual(rows.map(r => r.externalId), ['pl.polish#holiday@group.v.calendar.google.com']);
  assert.equal(env.memory.channels.size, 0);
});
```
