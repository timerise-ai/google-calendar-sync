# OAuth and tokens

Connecting is a redirect round trip that ends with a refresh token. Three things decide whether it
is safe and whether it keeps working: who the callback attributes the tokens to, what Google actually
granted, and what happens the day the refresh token stops working.

## Google Cloud setup

| Step | Where | Value |
|---|---|---|
| Enable the API | APIs & Services, then Library | Google Calendar API |
| OAuth client | Credentials, then Create OAuth client ID | Type *Web application* |
| Redirect URI | the client's *Authorized redirect URIs* | `https://api.example.com/calendar/google/callback`, exactly as `GOOGLE_CALENDAR_REDIRECT_URI` |
| Scopes | consent screen, then Data access | `.../auth/calendar.readonly`, `.../auth/calendar.events` |
| Audience | consent screen | *External* for customers' accounts; *Internal* only for your own Workspace |
| Publishing | consent screen | *In production*, then Google's verification for the calendar scopes |

**Testing mode is a trap that looks like a bug.** While the app is in *Testing*, refresh tokens expire
after seven days and only listed test users can connect. Every connection then flips to
`reauth_required` a week after it was made. Use *Testing* for development only.

The redirect URI must match byte for byte, scheme and trailing slash included. Register a separate
URI per environment rather than sharing one.

## Scopes

| Scope | Gives | Needed for |
|---|---|---|
| `calendar.readonly` | calendar list, events, watch | listing calendars, busy sync, the account e-mail |
| `calendar.events` | create, update, delete events | booking sync |

Google lets the user untick individual scopes on the consent screen. Read the token response's `scope`
field instead of assuming: without `calendar.readonly` the connection is refused
(`missing-calendar-scope`); without `calendar.events` it is kept with `canWrite: false`, every calendar
is non-writable and booking sync cannot be switched on.

## The round trip

```
GET /connect (signed in) -> AuthRequest{state, userId, verifier, +10 min} -> { url } with state + S256 challenge
Google consent -> GET /callback?code&state -> consume(state) atomically -> exchange(code, verifier)
             -> check scopes -> primary calendar id = account e-mail -> same account? keep : tear down -> save
```

- **The state is a key, not a payload.** Anything put into it (a user id, a provider) can be edited by
  whoever holds the URL; the callback must find the user server-side. Consuming it in a transaction
  makes it single use, and a replayed callback finds nothing.
- **PKCE** binds the code to the server that started the flow. A leaked code is useless without the
  verifier, which never leaves the server.
- **`access_type=offline` and `prompt=consent`** are both required. Google issues a refresh token only
  when a consent screen is shown; without `prompt=consent` a reconnect returns no refresh token and the
  callback fails with `no-refresh-token`.
- **The account e-mail is the id of the `primary` calendar.** No extra scope is needed. It tells a
  reconnect of the same account (keep settings, reopen channels) from a different account (tear down).
- **Old channels can only be stopped with the old account's tokens.** Google lets only the user and
  OAuth client that created a channel stop it, so tear-down runs before the new tokens replace the old.

## Token lifetime

| Event | What happens | Handled by |
|---|---|---|
| Access token near expiry | refreshed 60 s early, saved encrypted | `withAccessToken` |
| A 401 on a call | one forced refresh and one retry | `withAccessToken` |
| Refresh without a new refresh token | the stored one is kept | `toStoredTokens` |
| `invalid_grant` on refresh | connection set to `reauth_required`, error code `reauth-required` | `refreshTokens` |
| Reconnect, same account | `connected` again, settings kept, channels reopened, resync | `completeConnect` |
| Disconnect | channels stopped, blocks and rows deleted, grant revoked, tokens deleted | `disconnect` |

`invalid_grant` means only a new consent helps: the user or an admin revoked access, the token was
unused for six months, it came from an app in *Testing* and is a week old, or newer refresh tokens for
the same account and client pushed it out. Showing "Reconnect Google Calendar" is the fix; retrying is not.

Concurrent refreshes from two workers are harmless: Google accepts several valid access tokens at once
and the last write wins.

## Encryption key

`CALENDAR_TOKEN_KEY` is 32 random bytes, as 64 hex or 44 base64 characters:

```bash
openssl rand -hex 32
```

Payloads are prefixed `v1:` so a second key can be introduced later: decrypt by prefix, re-encrypt on
the next token refresh.

## Code

```ts
// calendar-sync/crypto.ts
import {
  createCipheriv,
  createDecipheriv,
  createHash,
  createHmac,
  randomBytes,
  timingSafeEqual,
} from 'node:crypto';

const ALGORITHM = 'aes-256-gcm';
const IV_LENGTH = 12;
const TAG_LENGTH = 16;
const FORMAT_VERSION = 'v1';

/** 32 bytes, given as 64 hex characters or 44 base64 characters. */
function parseKey(key: string): Buffer {
  if (key.length === 64 && /^[0-9a-fA-F]+$/.test(key)) return Buffer.from(key, 'hex');
  const buffer = Buffer.from(key, 'base64');
  if (buffer.length === 32) return buffer;
  throw new Error('Encryption key must be 32 bytes (64 hex or 44 base64 characters)');
}

/** `v1:base64(iv):base64(ciphertext):base64(tag)`. The version prefix leaves room for key rotation. */
export function encrypt(plaintext: string, key: string): string {
  const iv = randomBytes(IV_LENGTH);
  const cipher = createCipheriv(ALGORITHM, parseKey(key), iv, { authTagLength: TAG_LENGTH });
  const ciphertext = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  return [FORMAT_VERSION, iv, ciphertext, cipher.getAuthTag()]
    .map(part => (typeof part === 'string' ? part : part.toString('base64')))
    .join(':');
}

export function decrypt(payload: string, key: string): string {
  const [version, iv, ciphertext, tag] = payload.split(':');
  if (version !== FORMAT_VERSION || !iv || !ciphertext || !tag) {
    throw new Error('Unrecognised encrypted payload');
  }
  const decipher = createDecipheriv(ALGORITHM, parseKey(key), Buffer.from(iv, 'base64'), {
    authTagLength: TAG_LENGTH,
  });
  decipher.setAuthTag(Buffer.from(tag, 'base64'));
  return Buffer.concat([decipher.update(Buffer.from(ciphertext, 'base64')), decipher.final()]).toString(
    'utf8'
  );
}

export function randomToken(bytes = 32): string {
  return randomBytes(bytes).toString('base64url');
}

/** PKCE S256 challenge for a verifier. */
export function pkceChallenge(verifier: string): string {
  return createHash('sha256').update(verifier).digest('base64url');
}

export function sha256Hex(input: string): string {
  return createHash('sha256').update(input).digest('hex');
}

/** Deterministic row id: listing the same calendar twice, concurrently, writes one row. */
export function calendarRowId(userId: string, externalId: string): string {
  return sha256Hex(`calendar\n${userId}\n${externalId}`).slice(0, 40);
}

/** Deterministic busy-block id. Google event ids may contain `_`; hashing also keeps ids path-safe. */
export function busyBlockId(calendarId: string, eventId: string): string {
  return sha256Hex(`busy\n${calendarId}\n${eventId}`).slice(0, 40);
}

function hmac(data: string, secret: string): string {
  return createHmac('sha256', secret).update(data).digest('base64url');
}

/**
 * The channel token Google echoes back in `X-Goog-Channel-Token` (256 characters at most).
 * It carries only the channel id: everything else is looked up from the stored channel, so a long
 * calendar id cannot push it past the limit and nothing in it can go stale.
 */
export function signChannelToken(channelId: string, secret: string): string {
  if (!secret) throw new Error('Channel secret is not configured');
  return `${channelId}.${hmac(`channel:${channelId}`, secret)}`;
}

/** The channel id the token was issued for, or null. */
export function verifyChannelToken(token: string, secret: string): string | null {
  if (!secret) return null;
  const dot = token.lastIndexOf('.');
  if (dot <= 0) return null;
  const channelId = token.slice(0, dot);
  const received = Buffer.from(token.slice(dot + 1));
  const expected = Buffer.from(hmac(`channel:${channelId}`, secret));
  // Both are base64url SHA-256 digests (43 chars); a length mismatch is already a forgery.
  if (received.length !== expected.length || !timingSafeEqual(received, expected)) return null;
  return channelId;
}

/** Constant-time comparison for bearer secrets such as the cron secret. */
export function safeEqual(a: string, b: string): boolean {
  const left = createHash('sha256').update(a).digest();
  const right = createHash('sha256').update(b).digest();
  return timingSafeEqual(left, right) && a.length === b.length;
}
```

```ts
// calendar-sync/tokens.ts
import { decrypt, encrypt } from './crypto';
import { GoogleApiError, type TokenResponse } from './googleApi';
import { CalendarSyncError, type CalendarSyncDeps, type StoredTokens } from './types';

/** Refresh this long before Google's expiry, so a token never expires between check and use. */
const EXPIRY_SKEW_MS = 60_000;

export function sealTokens(deps: CalendarSyncDeps, tokens: StoredTokens): string {
  return encrypt(JSON.stringify(tokens), deps.config.encryptionKey);
}

export function openTokens(deps: CalendarSyncDeps, encryptedTokens: string): StoredTokens {
  return JSON.parse(decrypt(encryptedTokens, deps.config.encryptionKey)) as StoredTokens;
}

export function toStoredTokens(
  response: TokenResponse,
  now: Date,
  previous?: StoredTokens
): StoredTokens | null {
  // Google does not resend the refresh token on refresh; keep the one we have.
  const refreshToken = response.refresh_token ?? previous?.refreshToken;
  if (!refreshToken) return null;
  return {
    accessToken: response.access_token,
    refreshToken,
    expiresAt: now.getTime() + response.expires_in * 1000,
    scope: response.scope ?? previous?.scope ?? '',
  };
}

async function refreshTokens(
  deps: CalendarSyncDeps,
  userId: string,
  current: StoredTokens
): Promise<StoredTokens> {
  let response: TokenResponse;
  try {
    response = await deps.google.refresh(current.refreshToken);
  } catch (error) {
    // invalid_grant: revoked by the user or an admin, unused for six months, seven days old while
    // the OAuth app is in "Testing", or pushed out by newer refresh tokens for the same account and
    // client. Only a new consent fixes it.
    if (error instanceof GoogleApiError && error.reason === 'invalid_grant') {
      await deps.store.patchConnection(userId, {
        status: 'reauth_required',
        lastError: error.message,
        updatedAt: deps.now(),
      });
      deps.log.warn('google-calendar-reauth-required', { userId });
      throw new CalendarSyncError('reauth-required');
    }
    throw error;
  }
  const next = toStoredTokens(response, deps.now(), current);
  if (!next) throw new CalendarSyncError('no-refresh-token');
  await deps.store.patchConnection(userId, {
    encryptedTokens: sealTokens(deps, next),
    updatedAt: deps.now(),
  });
  return next;
}

/**
 * Run `fn` with a valid access token for the user's Google account. Refreshes ahead of expiry and
 * once more on a 401 (a token revoked early, or a clock skew), then gives up.
 */
export async function withAccessToken<T>(
  deps: CalendarSyncDeps,
  userId: string,
  fn: (accessToken: string) => Promise<T>
): Promise<T> {
  const connection = await deps.store.getConnection(userId);
  if (!connection) throw new CalendarSyncError('not-connected');
  if (connection.status === 'reauth_required') throw new CalendarSyncError('reauth-required');

  let tokens = openTokens(deps, connection.encryptedTokens);
  if (tokens.expiresAt - EXPIRY_SKEW_MS <= deps.now().getTime()) {
    tokens = await refreshTokens(deps, userId, tokens);
  }
  try {
    return await fn(tokens.accessToken);
  } catch (error) {
    if (!(error instanceof GoogleApiError) || error.status !== 401) throw error;
    tokens = await refreshTokens(deps, userId, tokens);
    return fn(tokens.accessToken);
  }
}
```

```ts
// calendar-sync/connect.ts
import { syncBusy } from './busySync';
import { refreshCalendarList } from './calendars';
import { startChannel, stopChannel, teardownCalendar } from './channels';
import { pkceChallenge, randomToken } from './crypto';
import { SCOPE_EVENTS, SCOPE_READ } from './googleApi';
import { openTokens, sealTokens, toStoredTokens } from './tokens';
import { CalendarSyncError, type CalendarConnection, type CalendarSyncDeps } from './types';

const AUTH_REQUEST_TTL_MS = 10 * 60 * 1000;
// randomToken(32) is 43 base64url characters. Checked before the lookup: the state arrives in the
// query string and becomes a document id, which must not contain '/'.
const STATE_PATTERN = /^[A-Za-z0-9_-]{43}$/;

/**
 * Begin a connection for the signed-in user. The state is an unguessable key of a server-side
 * record holding the user id and the PKCE verifier; the callback never reads a user id from the URL.
 */
export async function startConnect(deps: CalendarSyncDeps, userId: string): Promise<string> {
  const state = randomToken(32);
  const codeVerifier = randomToken(48);
  await deps.store.saveAuthRequest({
    state,
    userId,
    codeVerifier,
    expiresAt: new Date(deps.now().getTime() + AUTH_REQUEST_TTL_MS),
  });
  return deps.google.authUrl({ state, codeChallenge: pkceChallenge(codeVerifier) });
}

/**
 * Finish the OAuth round trip. Reconnecting the same Google account keeps every calendar's settings
 * and reopens its channels (the usual case: fixing `reauth_required`). Connecting a different
 * account tears the old one down first, because its calendar ids mean nothing to the new tokens.
 */
export async function completeConnect(
  deps: CalendarSyncDeps,
  params: { state: string; code: string }
): Promise<{ userId: string; sameAccount: boolean }> {
  const request = STATE_PATTERN.test(params.state)
    ? await deps.store.consumeAuthRequest(params.state)
    : null;
  if (!request || request.expiresAt <= deps.now()) throw new CalendarSyncError('invalid-or-expired-state');
  const { userId } = request;

  const response = await deps.google.exchangeCode(params.code, request.codeVerifier);
  const tokens = toStoredTokens(response, deps.now());
  if (!tokens) throw new CalendarSyncError('no-refresh-token');

  // Google lets the user untick scopes on the consent screen; the response says what was granted.
  const granted = new Set(tokens.scope.split(' '));
  if (!granted.has(SCOPE_READ)) throw new CalendarSyncError('missing-calendar-scope');
  const canWrite = granted.has(SCOPE_EVENTS);

  const accountEmail = await deps.google.primaryCalendarId(tokens.accessToken);
  const previous = await deps.store.getConnection(userId);
  const sameAccount = previous?.accountEmail === accountEmail;

  // Old channels can only be stopped with the old account's tokens, so this runs before they are replaced.
  if (previous && !sameAccount) {
    for (const calendar of await deps.store.listCalendars(userId)) {
      await teardownCalendar(deps, calendar);
    }
  }

  const now = deps.now();
  const connection: CalendarConnection = {
    userId,
    provider: 'google',
    accountEmail,
    status: 'connected',
    canWrite,
    encryptedTokens: sealTokens(deps, tokens),
    connectedAt: sameAccount && previous ? previous.connectedAt : now,
    updatedAt: now,
    lastError: null,
  };
  await deps.store.saveConnection(connection);

  if (sameAccount) {
    // Rows pick up the new write permission; busy calendars get a fresh channel and a resync,
    // since notifications were failing while the tokens were dead.
    const calendars = await refreshCalendarList(deps, userId);
    for (const calendar of calendars.filter(c => c.syncBusy)) {
      try {
        const old = calendar.channelId ? await deps.store.getChannel(calendar.channelId) : null;
        const fresh = await startChannel(deps, calendar);
        if (fresh && old) await stopChannel(deps, old);
        await syncBusy(deps, calendar.id);
      } catch (error) {
        deps.log.warn('google-calendar-resume-failed', {
          calendarId: calendar.id,
          message: error instanceof Error ? error.message : String(error),
        });
      }
    }
  }
  deps.log.info('google-calendar-connected', { userId, sameAccount, canWrite });
  return { userId, sameAccount };
}

/** Stop every channel, delete every busy block and row, revoke the grant at Google, forget the tokens. */
export async function disconnect(deps: CalendarSyncDeps, userId: string): Promise<void> {
  const connection = await deps.store.getConnection(userId);
  if (!connection) return;

  for (const calendar of await deps.store.listCalendars(userId)) {
    await teardownCalendar(deps, calendar);
  }
  try {
    // Revoking the refresh token revokes the whole grant, access tokens included.
    await deps.google.revoke(openTokens(deps, connection.encryptedTokens).refreshToken);
  } catch (error) {
    deps.log.warn('google-calendar-revoke-failed', {
      userId,
      message: error instanceof Error ? error.message : String(error),
    });
  }
  await deps.store.deleteConnection(userId);
  deps.log.info('google-calendar-disconnected', { userId });
}
```

## Checklist

- [ ] Consent screen *In production* before real users connect
- [ ] Redirect URI identical in the console and in config, per environment
- [ ] User id read from the consumed `AuthRequest`, never from the callback URL
- [ ] Token response's `scope` checked; `canWrite` stored
- [ ] `CALENDAR_TOKEN_KEY` set; no clear-text token fields anywhere
- [ ] `reauth_required` shown in the UI with a reconnect button
- [ ] Disconnect revokes at Google, not only locally
