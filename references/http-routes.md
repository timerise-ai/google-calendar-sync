# HTTP routes

Seven handlers on the Web `Request` and `Response` API, so they mount unchanged in Next.js route handlers,
Hono, Bun or Deno, and through a small adapter in Express.

| Route | Method | Auth | Answers |
|---|---|---|---|
| `/calendar/google/connect` | GET | user | `{ url }` to navigate to |
| `/calendar/google/callback` | GET | none (Google redirect; the state is the credential) | 303 to `appReturnUrl?calendar=connected\|denied\|<error code>` |
| `/calendar/google/calendars` | GET | user | `{ connection, calendars[] }` with health |
| `/calendar/google/calendars/:calendarId` | PATCH | user | the calendar view; 404, 409 on guard failures |
| `/calendar/google/disconnect` | POST | user | 204 |
| `/calendar/google/notifications` | POST | none (channel token) | 200 `sync`, 204 done/unknown, 401 bad token, 400 foreign resource, 500 sync failed |
| `/calendar/google/cron` | GET, POST | `Bearer CRON_SECRET` | the renewal report |

- **The callback always redirects.** A browser is on the other end; a JSON error there is a dead page.
  Cancelling the consent screen arrives as `?error=access_denied` and lands as `calendar=denied`.
- **The notification route has no session and needs none.** Google sends no body; everything is in
  `X-Goog-*` headers. The token is verified before any database read, so junk traffic costs an HMAC.
- **Answer 2xx for anything Google should consider delivered.** Google counts other statuses as
  failures and does not reliably retry them; the cron resync is the safety net, not redelivery.
- **Sync inline or deferred.** By default the notification handler awaits the sync and then answers.
  On platforms that can finish work after the response (`after()` in Next.js, `waitUntil` on Vercel or
  Cloudflare), pass it as `defer` and the response goes out immediately. Plain fire-and-forget
  (`void task()`) is only safe on a long-running server; serverless platforms freeze the instance.
- **The cron secret fails closed.** An empty `cronSecret` rejects everything.
- **Mount the notification route on the public HTTPS host Google can reach** with a publicly trusted
  certificate; self-signed and mismatched certificates are refused by Google.

## Handlers

```ts
// calendar-sync/webhook.ts
import { syncBusy } from './busySync';
import { verifyChannelToken } from './crypto';
import type { CalendarSyncDeps } from './types';

export interface NotificationHeaders {
  channelId: string;
  channelToken: string;
  /** `sync` once when the channel opens, `exists` on every change, `not_exists` when the resource is gone. */
  resourceState: string;
  resourceId: string;
}

export function readNotificationHeaders(headers: Headers): NotificationHeaders {
  return {
    channelId: headers.get('x-goog-channel-id') ?? '',
    channelToken: headers.get('x-goog-channel-token') ?? '',
    resourceState: headers.get('x-goog-resource-state') ?? '',
    resourceId: headers.get('x-goog-resource-id') ?? '',
  };
}

/** Runs the sync. Await it inline, or hand it to the platform (`after()`, `waitUntil`) and return. */
export type Defer = (task: () => Promise<void>) => Promise<void> | void;

/**
 * Google push notifications. The body is empty; everything is in headers, and nothing in them is
 * trusted until the token verifies. Answers 2xx for everything Google should consider delivered:
 * other statuses count as failures and are not reliably retried, so the cron resync is the safety net.
 */
export async function handleNotification(
  deps: CalendarSyncDeps,
  headers: NotificationHeaders,
  defer: Defer = task => task()
): Promise<number> {
  if (headers.resourceState === 'sync') return 200;

  // Verified before any lookup: junk traffic costs an HMAC, not a database read.
  const channelId = verifyChannelToken(headers.channelToken, deps.config.channelSecret);
  if (!channelId || channelId !== headers.channelId) {
    deps.log.warn('google-calendar-webhook-bad-token', { channelId: headers.channelId });
    return 401;
  }

  const channel = await deps.store.getChannel(channelId);
  if (!channel) {
    // Renewed or stopped: the overlap after a renewal delivers to the old id for a while.
    deps.log.info('google-calendar-webhook-unknown-channel', { channelId });
    return 204;
  }
  if (channel.resourceId !== headers.resourceId) {
    deps.log.warn('google-calendar-webhook-resource-mismatch', { channelId });
    return 400;
  }
  if (headers.resourceState !== 'exists') return 204;

  try {
    await defer(async () => {
      await syncBusy(deps, channel.calendarId);
    });
    return 204;
  } catch (error) {
    deps.log.error('google-calendar-webhook-sync-failed', {
      channelId,
      message: error instanceof Error ? error.message : String(error),
    });
    return 500;
  }
}
```

```ts
// calendar-sync/routes.ts
import { refreshCalendarList, updateCalendarSettings } from './calendars';
import { renewChannels } from './channels';
import { completeConnect, disconnect, startConnect } from './connect';
import { safeEqual } from './crypto';
import { deriveCalendarHealth, type CalendarHealth } from './health';
import { handleNotification, readNotificationHeaders, type Defer } from './webhook';
import {
  CalendarSyncError,
  type CalendarErrorCode,
  type CalendarSettingsPatch,
  type CalendarSyncDeps,
  type LinkedCalendar,
} from './types';

export interface CalendarRouteOptions {
  /** The host's auth guard: the signed-in user's id, or null. */
  authenticate: (request: Request) => Promise<string | null>;
  /** Where the browser lands after the callback; `?calendar=<result>` is appended. */
  appReturnUrl: string;
  cronSecret: string;
  defer?: Defer;
  /** Older than this, a synced calendar reports `stale`. Default 26 h for a 12-hourly cron. */
  staleAfterMs?: number;
}

const STATUS: Record<CalendarErrorCode, number> = {
  'not-connected': 409,
  'reauth-required': 409,
  'calendar-not-found': 404,
  'calendar-not-writable': 409,
  'invalid-or-expired-state': 400,
  'no-refresh-token': 400,
  'missing-calendar-scope': 400,
};

const json = (body: unknown, status = 200) => Response.json(body, { status });

function errorResponse(error: unknown): Response {
  if (error instanceof CalendarSyncError) return json({ error: error.code }, STATUS[error.code]);
  throw error;
}

function parsePatch(body: unknown): CalendarSettingsPatch | null {
  if (typeof body !== 'object' || body === null) return null;
  const input = body as Record<string, unknown>;
  const patch: CalendarSettingsPatch = {};
  for (const key of Object.keys(input)) {
    const value = input[key];
    if (key === 'title' && typeof value === 'string' && value.trim() && value.length <= 200) patch.title = value.trim();
    else if ((key === 'syncBusy' || key === 'syncBookings') && typeof value === 'boolean') patch[key] = value;
    else return null;
  }
  return patch;
}

export interface CalendarView {
  id: string;
  title: string;
  externalId: string;
  isWritable: boolean;
  syncBusy: boolean;
  syncBookings: boolean;
  health: CalendarHealth;
  lastSyncedAt: string | null;
  lastSyncError: string | null;
}

/** Framework-free handlers on the Web Request/Response API. Mount them in the host's router. */
export function createCalendarRoutes(deps: CalendarSyncDeps, options: CalendarRouteOptions) {
  const staleAfterMs = options.staleAfterMs ?? 26 * 60 * 60 * 1000;

  async function view(userId: string, calendars: LinkedCalendar[]) {
    const connection = await deps.store.getConnection(userId);
    if (!connection) return { connection: null, calendars: [] };
    const now = deps.now();
    const rows: CalendarView[] = [];
    for (const calendar of calendars) {
      const channel = calendar.channelId ? await deps.store.getChannel(calendar.channelId) : null;
      rows.push({
        id: calendar.id,
        title: calendar.title,
        externalId: calendar.externalId,
        isWritable: calendar.isWritable,
        syncBusy: calendar.syncBusy,
        syncBookings: calendar.syncBookings,
        health: deriveCalendarHealth({ connection, calendar, channel, now, staleAfterMs }),
        lastSyncedAt: calendar.lastSyncedAt?.toISOString() ?? null,
        lastSyncError: calendar.lastSyncError,
      });
    }
    return {
      connection: {
        accountEmail: connection.accountEmail,
        status: connection.status,
        canWrite: connection.canWrite,
      },
      calendars: rows,
    };
  }

  return {
    /** GET returns `{ url }`; the client navigates to it. */
    async connect(request: Request): Promise<Response> {
      const userId = await options.authenticate(request);
      if (!userId) return json({ error: 'unauthenticated' }, 401);
      return json({ url: await startConnect(deps, userId) });
    },

    /** GET, from Google. A browser is on the other end: always redirect, never answer with JSON. */
    async callback(request: Request): Promise<Response> {
      const params = new URL(request.url).searchParams;
      const back = (result: string) => {
        const url = new URL(options.appReturnUrl);
        url.searchParams.set('calendar', result);
        return Response.redirect(url.toString(), 303);
      };
      const code = params.get('code');
      const state = params.get('state');
      // error=access_denied when the user cancels the consent screen.
      if (params.get('error') || !code || !state) return back('denied');
      try {
        await completeConnect(deps, { state, code });
        return back('connected');
      } catch (error) {
        deps.log.error('google-calendar-callback-failed', {
          message: error instanceof Error ? error.message : String(error),
        });
        return back(error instanceof CalendarSyncError ? error.code : 'failed');
      }
    },

    /** GET returns the connection, calendars and their health. Refreshes from Google; falls back to stored rows. */
    async list(request: Request): Promise<Response> {
      const userId = await options.authenticate(request);
      if (!userId) return json({ error: 'unauthenticated' }, 401);
      if (!(await deps.store.getConnection(userId))) return json({ connection: null, calendars: [] });
      let calendars: LinkedCalendar[];
      try {
        calendars = await refreshCalendarList(deps, userId);
      } catch (error) {
        if (!(error instanceof CalendarSyncError)) {
          deps.log.warn('google-calendar-list-refresh-failed', {
            userId,
            message: error instanceof Error ? error.message : String(error),
          });
        }
        calendars = await deps.store.listCalendars(userId);
      }
      return json(await view(userId, calendars));
    },

    /** PATCH `{ title?, syncBusy?, syncBookings? }`. */
    async update(request: Request, calendarId: string): Promise<Response> {
      const userId = await options.authenticate(request);
      if (!userId) return json({ error: 'unauthenticated' }, 401);
      const patch = parsePatch(await request.json().catch(() => null));
      if (!patch) return json({ error: 'invalid-body' }, 400);
      try {
        const calendar = await updateCalendarSettings(deps, userId, calendarId, patch);
        const result = await view(userId, [calendar]);
        return json(result.calendars[0] ?? null);
      } catch (error) {
        return errorResponse(error);
      }
    },

    /** POST. */
    async disconnect(request: Request): Promise<Response> {
      const userId = await options.authenticate(request);
      if (!userId) return json({ error: 'unauthenticated' }, 401);
      await disconnect(deps, userId);
      return new Response(null, { status: 204 });
    },

    /** POST, from Google. Unauthenticated by design; the channel token is the credential. */
    async notifications(request: Request): Promise<Response> {
      const status = await handleNotification(deps, readNotificationHeaders(request.headers), options.defer);
      return new Response(null, { status });
    },

    /**
     * GET (Vercel Cron) or POST, with `Authorization: Bearer <cronSecret>`. Every 12 h with the default
     * 48 h renewal window.
     */
    async cron(request: Request): Promise<Response> {
      const header = request.headers.get('authorization') ?? '';
      // Fails closed: an unset secret matches nothing.
      if (!options.cronSecret || !safeEqual(header, `Bearer ${options.cronSecret}`)) {
        return json({ error: 'unauthorized' }, 401);
      }
      return json(await renewChannels(deps));
    },
  };
}
```

## Wiring

One module builds the dependencies. This example uses Firestore and environment variables; swap the
store and the three declared host functions for your own, and keep the rest as it is. It builds on first
use, not at import: `next build` imports every route module to collect page data, and a `required()` at
module scope fails the build wherever the secrets are not set, such as CI or a preview deployment.

```ts
// calendar-sync/setup.ts: the one place that builds the deps (example: Firestore, env vars)
import { getFirestore } from 'firebase-admin/firestore';

import { createGoogleApi } from './googleApi';
import { createCalendarRoutes } from './routes';
import { createFirestoreCalendarStore } from './store/firestore';
import { DEFAULT_SYNC_SETTINGS, type BookingForCalendar, type CalendarSyncDeps } from './types';

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is not set`);
  return value;
}

// Host seams, declared here so the example compiles: replace with your own.
declare function verifySession(request: Request): Promise<string | null>;
declare function listUpcomingBookingsForHost(userId: string, from: Date): Promise<BookingForCalendar[]>;
declare function saveMeetingUrl(bookingId: string, url: string): Promise<void>;

type CalendarRoutes = ReturnType<typeof createCalendarRoutes>;
let built: { deps: CalendarSyncDeps; routes: CalendarRoutes } | undefined;

function build(): { deps: CalendarSyncDeps; routes: CalendarRoutes } {
  if (built) return built;
  const config = {
    ...DEFAULT_SYNC_SETTINGS,
    clientId: required('GOOGLE_CALENDAR_CLIENT_ID'),
    clientSecret: required('GOOGLE_CALENDAR_CLIENT_SECRET'),
    redirectUri: required('GOOGLE_CALENDAR_REDIRECT_URI'),
    notificationUrl: required('GOOGLE_CALENDAR_NOTIFICATION_URL'),
    channelSecret: required('CALENDAR_CHANNEL_SECRET'),
    encryptionKey: required('CALENDAR_TOKEN_KEY'),
  };
  const deps: CalendarSyncDeps = {
    store: createFirestoreCalendarStore(getFirestore()),
    google: createGoogleApi(config),
    config,
    bookings: { listUpcomingForPrimaryHost: listUpcomingBookingsForHost },
    log: {
      info: (kind, data) => console.log(JSON.stringify({ severity: 'INFO', kind, ...data })),
      warn: (kind, data) => console.warn(JSON.stringify({ severity: 'WARNING', kind, ...data })),
      error: (kind, data) => console.error(JSON.stringify({ severity: 'ERROR', kind, ...data })),
    },
    now: () => new Date(),
    onConferenceLink: saveMeetingUrl,
  };
  const routes = createCalendarRoutes(deps, {
    authenticate: verifySession,
    appReturnUrl: required('CALENDAR_RETURN_URL'),
    cronSecret: required('CRON_SECRET'),
  });
  return (built = { deps, routes });
}

/** The deps for `pushBookingEvent`, `removeBookingEvent` and `listHostBusy`, built on first call. */
export const calendarDeps = (): CalendarSyncDeps => build().deps;
/** The seven handlers, built on first call; a missing variable fails that request, not the build. */
export const calendarRoutes = (): CalendarRoutes => build().routes;
```

Copy the wiring as it is. Each of these was added by an agent and none is needed:

- **Lazy construction of your own.** `setup.ts` already builds on first use; no getter, proxy or default
  value per variable. A missing variable fails the request that needs it, with its name.
- **A default for a variable.** Every variable in [operations.md](operations.md) is required; a fallback
  such as a default `CALENDAR_RETURN_URL` hides a misconfigured deployment.
- **Reordering the webhook.** The `sync` handshake is answered before the token check on purpose: it reads
  nothing and changes nothing, so a forged one costs a 200 and no more.
- **Work of the host's own in the cron route.** The route renews channels and resyncs, every 12 hours.
  Retries, queues or reminders of the host's get their own route and schedule.

Next.js App Router:

```ts
// app/api/calendar/google/[action]/route.ts: Next.js App Router
import { calendarRoutes } from '@/calendar-sync/setup';

export async function GET(request: Request, { params }: { params: Promise<{ action: string }> }) {
  const { action } = await params;
  if (action === 'connect') return calendarRoutes().connect(request);
  if (action === 'callback') return calendarRoutes().callback(request);
  if (action === 'calendars') return calendarRoutes().list(request);
  // Vercel Cron calls with GET and `Authorization: Bearer $CRON_SECRET`; other schedulers POST.
  if (action === 'cron') return calendarRoutes().cron(request);
  return new Response(null, { status: 404 });
}

export async function POST(request: Request, { params }: { params: Promise<{ action: string }> }) {
  const { action } = await params;
  if (action === 'notifications') return calendarRoutes().notifications(request);
  if (action === 'disconnect') return calendarRoutes().disconnect(request);
  if (action === 'cron') return calendarRoutes().cron(request);
  return new Response(null, { status: 404 });
}
```

```ts
// app/api/calendar/google/calendars/[calendarId]/route.ts
import { calendarRoutes } from '@/calendar-sync/setup';

export async function PATCH(request: Request, { params }: { params: Promise<{ calendarId: string }> }) {
  return calendarRoutes().update(request, (await params).calendarId);
}
```

In Next.js 15+, `defer: task => after(task)` (from `next/server`) answers Google before the sync runs.

Express (the adapter turns each request into a Web `Request` once):

```ts
// server/calendarRouter.ts: Express, adapt each request to the Web API once, and send the Response back.
import express, { type Request as ExpressRequest, type Response as ExpressResponse } from 'express';

import { calendarRoutes } from '@/calendar-sync/setup';

function toWebRequest(req: ExpressRequest): Request {
  const headers = new Headers();
  for (const [name, value] of Object.entries(req.headers)) {
    if (typeof value === 'string') headers.set(name, value);
    else if (Array.isArray(value)) for (const item of value) headers.append(name, item);
  }
  const hasBody = req.method !== 'GET' && req.method !== 'HEAD' && req.body !== undefined;
  return new Request(`${req.protocol}://${req.get('host')}${req.originalUrl}`, {
    method: req.method,
    headers,
    body: hasBody ? JSON.stringify(req.body) : undefined,
  });
}

async function send(res: ExpressResponse, response: Response): Promise<void> {
  res.status(response.status);
  response.headers.forEach((value, name) => res.setHeader(name, value));
  res.send(Buffer.from(await response.arrayBuffer()));
}

// Looked up per request, so a missing variable fails that request and not the server's start.
type Handler = Exclude<keyof ReturnType<typeof calendarRoutes>, 'update'>;
const route = (name: Handler) => (req: ExpressRequest, res: ExpressResponse) =>
  calendarRoutes()[name](toWebRequest(req)).then(response => send(res, response));

export const calendarRouter = express.Router();
calendarRouter.use(express.json());
calendarRouter.get('/calendar/google/connect', route('connect'));
calendarRouter.get('/calendar/google/callback', route('callback'));
calendarRouter.get('/calendar/google/calendars', route('list'));
calendarRouter.patch('/calendar/google/calendars/:calendarId', (req, res) =>
  calendarRoutes().update(toWebRequest(req), String(req.params.calendarId)).then(response => send(res, response))
);
calendarRouter.post('/calendar/google/disconnect', route('disconnect'));
calendarRouter.post('/calendar/google/notifications', route('notifications'));
calendarRouter.get('/calendar/google/cron', route('cron'));
calendarRouter.post('/calendar/google/cron', route('cron'));
```

## Checklist

- [ ] Notification URL public, HTTPS, trusted certificate; same value as `GOOGLE_CALENDAR_NOTIFICATION_URL`
- [ ] Callback URL registered in the Google console; same value as `GOOGLE_CALENDAR_REDIRECT_URI`
- [ ] `authenticate` is the host's real session check, returning the user id
- [ ] Cron route scheduled with the bearer secret
- [ ] On serverless, `defer` uses the platform's after-response primitive, or stays inline
