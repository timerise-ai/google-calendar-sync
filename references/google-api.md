# Google API client

Eleven calls over `fetch`, behind one interface. No `googleapis` dependency: the package is large, its
automatic token refresh happens in memory where an `invalid_grant` cannot be turned into a
`reauth_required` state, and an interface over `fetch` is what the tests replace with a fake.
`@googleapis/calendar` can implement the same `GoogleApi` interface if the host prefers it.

## Calls

| Method | HTTP | Notes |
|---|---|---|
| `authUrl` | — | `access_type=offline`, `prompt=consent`, PKCE S256 |
| `exchangeCode` | `POST oauth2.googleapis.com/token` | `grant_type=authorization_code` with `code_verifier` |
| `refresh` | `POST …/token` | No new refresh token in the response, as a rule |
| `revoke` | `POST oauth2.googleapis.com/revoke` | 400 means already revoked: treated as success |
| `primaryCalendarId` | `GET /calendars/primary` | The id is the account e-mail |
| `listCalendars` | `GET /users/me/calendarList` | Paged, 250 per page |
| `listEvents` | `GET /calendars/{id}/events` | `singleEvents=true`, `maxResults=2500`, one page per call |
| `insertEvent` | `POST /calendars/{id}/events` | `sendUpdates=none`, `conferenceDataVersion` 0 or 1 |
| `updateEvent` | `PUT /calendars/{id}/events/{eventId}` | Full replace; used after a 409 |
| `deleteEvent` | `DELETE /calendars/{id}/events/{eventId}` | 404 never existed, 410 already deleted |
| `watchEvents` | `POST /calendars/{id}/events/watch` | `type: 'web_hook'`, returns `resourceId`, `expiration` |
| `stopChannel` | `POST /channels/stop` | Needs the channel id **and** `resourceId` |

## Facts that shape the code

- **Calendar ids contain `@` and `#`** (`team@group.calendar.google.com`,
  `en.usa#holiday@group.v.calendar.google.com`). Every path segment is `encodeURIComponent`-ed.
- **`singleEvents=true` expands recurring series.** Each instance has its own id
  (`<seriesId>_20261020T090000Z`), its own times, and can be moved or cancelled alone. Busy sync wants
  exactly that. Without it a weekly meeting arrives as one event with an `RRULE` the sync would have to
  expand itself.
- **`maxResults` caps at 2500 per page; `nextPageToken` is the only sign there is more.** A calendar of
  a busy shared room passes 2500 instances in six months. Reading one page and treating it as the
  calendar loses everything after it, and a diff against that page deletes their blocks.
- **`timeMin` filters on the event's end, `timeMax` on its start.** An event in progress at `now` is
  included, which is what availability needs.
- **Event ids you choose** must use base32hex (`a`–`v`, `0`–`9`), 5 to 1024 characters, unique per
  calendar. A 409 on insert means the id exists, and Google's guidance is to update instead.
- **`sendUpdates` defaults to none on insert.** This skill states it explicitly on every write so a
  future default change cannot start e-mailing guests.
- **`conferenceDataVersion=1` is required for Meet links** to be created from a `createRequest`, and on
  every later modification that should keep them.
- **Errors** come back as `{ error: { code, message, errors: [{ reason }] } }` from the Calendar API
  and `{ error, error_description }` from the token endpoint. `GoogleApiError.reason` normalizes both:
  `invalid_grant`, `duplicate`, `notFound`, `rateLimitExceeded`, `quotaExceeded`, `backendError`.

## Quotas

Calendar API quotas are per project and per user per minute. The heavy callers are full resyncs. Their
cost is one `events.list` page per 2500 events per calendar, per notification burst (collapsed by the
lease) plus one per cron run. 403/429 `rateLimitExceeded` surfaces as a failed sync: the error is
recorded on the row and the next notification or cron run retries. If you see it routinely, lower the
cron concurrency (`renewChannels(deps, 4)`) before asking Google for more quota.

## Code

```ts
// calendar-sync/googleApi.ts
import type { GoogleAccessRole } from './types';

export const SCOPE_READ = 'https://www.googleapis.com/auth/calendar.readonly';
export const SCOPE_EVENTS = 'https://www.googleapis.com/auth/calendar.events';

const AUTH_URL = 'https://accounts.google.com/o/oauth2/v2/auth';
const TOKEN_URL = 'https://oauth2.googleapis.com/token';
const REVOKE_URL = 'https://oauth2.googleapis.com/revoke';
const API = 'https://www.googleapis.com/calendar/v3';

export class GoogleApiError extends Error {
  readonly status: number;
  /** `errors[0].reason` for API calls (`notFound`, `duplicate`, `rateLimitExceeded`…), `error` for OAuth (`invalid_grant`). */
  readonly reason: string;

  constructor(status: number, reason: string, message: string) {
    super(message);
    this.name = 'GoogleApiError';
    this.status = status;
    this.reason = reason;
  }
}

export interface TokenResponse {
  access_token: string;
  expires_in: number;
  refresh_token?: string;
  scope?: string;
  token_type: string;
}

export interface GoogleCalendarListEntry {
  id: string;
  summary?: string;
  summaryOverride?: string;
  accessRole: GoogleAccessRole;
  primary?: boolean;
}

export interface GoogleEventDateTime {
  /** All-day events: `YYYY-MM-DD`. `end.date` is exclusive. */
  date?: string;
  /** Timed events: RFC 3339 with an offset. */
  dateTime?: string;
  timeZone?: string;
}

export interface GoogleEvent {
  id?: string;
  status?: 'confirmed' | 'tentative' | 'cancelled';
  summary?: string;
  description?: string;
  start?: GoogleEventDateTime;
  end?: GoogleEventDateTime;
  transparency?: 'opaque' | 'transparent';
  eventType?: string;
  attendees?: { email?: string; self?: boolean; responseStatus?: string }[];
  extendedProperties?: { private?: Record<string, string>; shared?: Record<string, string> };
  conferenceData?: { createRequest?: { requestId: string } };
  updated?: string;
  hangoutLink?: string;
}

export interface EventsPage {
  items: GoogleEvent[];
  nextPageToken?: string;
  /** The calendar's zone. All-day events are anchored to it. */
  timeZone?: string;
}

export interface WatchResponse {
  id: string;
  resourceId: string;
  /** Epoch milliseconds, as a string. */
  expiration?: string;
}

export interface GoogleApi {
  authUrl(params: { state: string; codeChallenge: string }): string;
  exchangeCode(code: string, codeVerifier: string): Promise<TokenResponse>;
  refresh(refreshToken: string): Promise<TokenResponse>;
  revoke(token: string): Promise<void>;
  primaryCalendarId(accessToken: string): Promise<string>;
  listCalendars(accessToken: string): Promise<GoogleCalendarListEntry[]>;
  listEvents(
    accessToken: string,
    calendarId: string,
    query: { timeMin: Date; timeMax: Date; pageToken?: string }
  ): Promise<EventsPage>;
  insertEvent(
    accessToken: string,
    calendarId: string,
    event: GoogleEvent,
    conferenceDataVersion: 0 | 1
  ): Promise<GoogleEvent>;
  updateEvent(
    accessToken: string,
    calendarId: string,
    eventId: string,
    event: GoogleEvent,
    conferenceDataVersion: 0 | 1
  ): Promise<GoogleEvent>;
  deleteEvent(
    accessToken: string,
    calendarId: string,
    eventId: string,
    sendUpdates: 'all' | 'none'
  ): Promise<void>;
  watchEvents(
    accessToken: string,
    calendarId: string,
    channel: { id: string; address: string; token: string }
  ): Promise<WatchResponse>;
  stopChannel(accessToken: string, channel: { id: string; resourceId: string }): Promise<void>;
}

interface GoogleErrorBody {
  error?: string | { code?: number; message?: string; errors?: { reason?: string }[]; status?: string };
  error_description?: string;
}

async function toError(response: Response): Promise<GoogleApiError> {
  let body: GoogleErrorBody = {};
  try {
    body = (await response.json()) as GoogleErrorBody;
  } catch {
    // Not JSON: keep the status.
  }
  if (typeof body.error === 'string') {
    return new GoogleApiError(response.status, body.error, body.error_description ?? body.error);
  }
  const reason = body.error?.errors?.[0]?.reason ?? body.error?.status ?? `http_${response.status}`;
  return new GoogleApiError(response.status, reason, body.error?.message ?? reason);
}

export function createGoogleApi(options: {
  clientId: string;
  clientSecret: string;
  redirectUri: string;
  fetch?: typeof fetch;
}): GoogleApi {
  const http = options.fetch ?? fetch;

  async function call<T>(
    accessToken: string,
    method: string,
    path: string,
    init: { query?: Record<string, string | undefined>; body?: unknown } = {}
  ): Promise<T> {
    const url = new URL(API + path);
    for (const [key, value] of Object.entries(init.query ?? {})) {
      if (value !== undefined) url.searchParams.set(key, value);
    }
    const response = await http(url, {
      method,
      headers: {
        Authorization: `Bearer ${accessToken}`,
        ...(init.body === undefined ? {} : { 'Content-Type': 'application/json' }),
      },
      body: init.body === undefined ? undefined : JSON.stringify(init.body),
    });
    if (!response.ok) throw await toError(response);
    if (response.status === 204) return undefined as T;
    const text = await response.text();
    return (text ? JSON.parse(text) : undefined) as T;
  }

  async function token(params: Record<string, string>): Promise<TokenResponse> {
    const response = await http(TOKEN_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        client_id: options.clientId,
        client_secret: options.clientSecret,
        ...params,
      }),
    });
    if (!response.ok) throw await toError(response);
    return (await response.json()) as TokenResponse;
  }

  // Calendar ids contain '@' and '#'; every one goes through this.
  const calendarPath = (calendarId: string) => `/calendars/${encodeURIComponent(calendarId)}`;

  return {
    authUrl({ state, codeChallenge }) {
      const url = new URL(AUTH_URL);
      url.search = new URLSearchParams({
        client_id: options.clientId,
        redirect_uri: options.redirectUri,
        response_type: 'code',
        scope: [SCOPE_READ, SCOPE_EVENTS].join(' '),
        // offline + consent: Google issues a refresh token only on a consent screen, and a
        // reconnect without one would leave the stored refresh token as the only one.
        access_type: 'offline',
        prompt: 'consent',
        include_granted_scopes: 'true',
        state,
        code_challenge: codeChallenge,
        code_challenge_method: 'S256',
      }).toString();
      return url.toString();
    },

    exchangeCode: (code, codeVerifier) =>
      token({
        grant_type: 'authorization_code',
        code,
        code_verifier: codeVerifier,
        redirect_uri: options.redirectUri,
      }),

    refresh: refreshToken => token({ grant_type: 'refresh_token', refresh_token: refreshToken }),

    async revoke(tokenValue) {
      const response = await http(REVOKE_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({ token: tokenValue }),
      });
      // 400 invalid_token: already revoked or expired, which is the goal.
      if (!response.ok && response.status !== 400) throw await toError(response);
    },

    async primaryCalendarId(accessToken) {
      const calendar = await call<{ id: string }>(accessToken, 'GET', '/calendars/primary');
      return calendar.id;
    },

    async listCalendars(accessToken) {
      const entries: GoogleCalendarListEntry[] = [];
      let pageToken: string | undefined;
      do {
        const page = await call<{ items?: GoogleCalendarListEntry[]; nextPageToken?: string }>(
          accessToken,
          'GET',
          '/users/me/calendarList',
          { query: { maxResults: '250', pageToken } }
        );
        entries.push(...(page.items ?? []));
        pageToken = page.nextPageToken;
      } while (pageToken);
      return entries;
    },

    async listEvents(accessToken, calendarId, { timeMin, timeMax, pageToken }) {
      const page = await call<EventsPage>(accessToken, 'GET', `${calendarPath(calendarId)}/events`, {
        query: {
          // Expands recurring series into instances, so every instance is its own busy block.
          singleEvents: 'true',
          timeMin: timeMin.toISOString(),
          timeMax: timeMax.toISOString(),
          maxResults: '2500',
          pageToken,
        },
      });
      return { ...page, items: page.items ?? [] };
    },

    insertEvent: (accessToken, calendarId, event, conferenceDataVersion) =>
      call<GoogleEvent>(accessToken, 'POST', `${calendarPath(calendarId)}/events`, {
        query: { conferenceDataVersion: String(conferenceDataVersion), sendUpdates: 'none' },
        body: event,
      }),

    updateEvent: (accessToken, calendarId, eventId, event, conferenceDataVersion) =>
      call<GoogleEvent>(
        accessToken,
        'PUT',
        `${calendarPath(calendarId)}/events/${encodeURIComponent(eventId)}`,
        { query: { conferenceDataVersion: String(conferenceDataVersion), sendUpdates: 'none' }, body: event }
      ),

    async deleteEvent(accessToken, calendarId, eventId, sendUpdates) {
      await call<void>(
        accessToken,
        'DELETE',
        `${calendarPath(calendarId)}/events/${encodeURIComponent(eventId)}`,
        { query: { sendUpdates } }
      );
    },

    watchEvents: (accessToken, calendarId, channel) =>
      call<WatchResponse>(accessToken, 'POST', `${calendarPath(calendarId)}/events/watch`, {
        body: { ...channel, type: 'web_hook' },
      }),

    async stopChannel(accessToken, channel) {
      await call<void>(accessToken, 'POST', '/channels/stop', { body: channel });
    },
  };
}
```

## Checklist

- [ ] Every calendar id in a path goes through `encodeURIComponent`
- [ ] `listEvents` callers loop on `nextPageToken`
- [ ] `stopChannel` called with both channel id and `resourceId`
- [ ] Errors branch on `GoogleApiError.status` / `.reason`, never on message text
