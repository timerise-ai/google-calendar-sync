# Firestore backend

`CalendarStore` on `firebase-admin`. Five top-level collections, all server-only.

| Collection | Doc id | Fields |
|---|---|---|
| `calendarAuthRequests` | `state` | `userId`, `codeVerifier`, `expiresAt` |
| `calendarConnections` | `userId` | `accountEmail`, `status`, `canWrite`, `encryptedTokens`, `connectedAt`, `updatedAt`, `lastError` |
| `calendarLinks` | `calendarRowId(userId, externalId)` | the `LinkedCalendar` fields |
| `calendarChannels` | channel UUID | `userId`, `calendarId`, `resourceId`, `expiresAt`, `createdAt` |
| `calendarBusyBlocks` | `busyBlockId(calendarId, eventId)` | `userId`, `calendarId`, `eventId`, `from`, `to`, `allDay` |

Top-level rather than `users/{uid}/...` subcollections: the cron pages across all users' calendars, the
webhook resolves a channel without knowing its user, and the availability query reads blocks by user
id across calendars. All three are one query on a top-level collection.

## Indexes

| Collection | Fields | Used by |
|---|---|---|
| `calendarBusyBlocks` | `userId` ASC, `to` ASC | `listHostBusy` (availability) |

Every other query uses one equality filter, or one range filter ordered by the same field, which
Firestore's automatic single-field indexes serve. The `firestore.indexes.json` entry:

```json
{
  "indexes": [
    {
      "collectionGroup": "calendarBusyBlocks",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "to", "order": "ASCENDING" }
      ]
    }
  ]
}
```

## Security rules

Nothing here is for clients. The Admin SDK bypasses rules; deny everything else, inside the existing
`match /databases/{database}/documents` block:

```
match /calendarAuthRequests/{id} { allow read, write: if false; }
match /calendarConnections/{id} { allow read, write: if false; }
match /calendarLinks/{id} { allow read, write: if false; }
match /calendarChannels/{id} { allow read, write: if false; }
match /calendarBusyBlocks/{id} { allow read, write: if false; }
```

If a client needs to show calendars, it calls the list route, which adds the derived health.

## Notes

- **`mutateCalendar` is a transaction.** Firestore retries it on contention, which is why the function
  passed in must be pure.
- **`consumeAuthRequest` deletes inside the transaction whether or not the request expired**, so an
  expired state cannot be retried until it happens to be valid.
- **Batches cap at 500 writes**; `applyBusyDiff` and the deletes chunk accordingly. A calendar with
  thousands of events in the window commits in several batches. That is safe because the diff is
  idempotent: an interrupted run is completed by the next one.
- **Dates are `Timestamp` in Firestore and `Date` everywhere else**; `toDoc`/`fromDoc` convert at the
  boundary so no `Timestamp` leaks into the domain functions.
- **A `/` in a document id throws.** Channel ids come from verified tokens, but `getChannel` still
  refuses one rather than turning a bad request into a 500.
- **TTL policy (optional):** a Firestore TTL policy on `calendarAuthRequests.expiresAt` removes
  abandoned requests without the cron's sweep.

## Code

```ts
// calendar-sync/store/firestore.ts
import type { firestore } from 'firebase-admin';
import { FieldPath, Timestamp } from 'firebase-admin/firestore';

import type {
  AuthRequest,
  BusyBlock,
  CalendarConnection,
  CalendarStore,
  LinkedCalendar,
  WatchChannel,
} from '../types';

const C = {
  authRequests: 'calendarAuthRequests',
  connections: 'calendarConnections',
  calendars: 'calendarLinks',
  channels: 'calendarChannels',
  busy: 'calendarBusyBlocks',
} as const;

const BATCH_LIMIT = 500;

/** Timestamp to Date for the listed fields; everything else passes through. */
function fromDoc<T>(data: firestore.DocumentData, dateFields: readonly string[]): T {
  const out: Record<string, unknown> = { ...data };
  for (const field of dateFields) {
    const value = data[field];
    out[field] = value instanceof Timestamp ? value.toDate() : (value ?? null);
  }
  return out as T;
}

function toDoc(data: object): firestore.DocumentData {
  const out: firestore.DocumentData = {};
  for (const [key, value] of Object.entries(data)) {
    if (value === undefined) continue;
    out[key] = value instanceof Date ? Timestamp.fromDate(value) : value;
  }
  return out;
}

const CALENDAR_DATES = ['lastSyncedAt', 'syncLeaseUntil'] as const;
const CHANNEL_DATES = ['expiresAt', 'createdAt'] as const;
const BUSY_DATES = ['from', 'to'] as const;

async function deleteInBatches(db: firestore.Firestore, refs: firestore.DocumentReference[]): Promise<void> {
  for (let i = 0; i < refs.length; i += BATCH_LIMIT) {
    const batch = db.batch();
    for (const ref of refs.slice(i, i + BATCH_LIMIT)) batch.delete(ref);
    await batch.commit();
  }
}

export function createFirestoreCalendarStore(db: firestore.Firestore): CalendarStore {
  const calendars = db.collection(C.calendars);
  const channels = db.collection(C.channels);
  const busy = db.collection(C.busy);

  return {
    async saveAuthRequest(request) {
      await db.collection(C.authRequests).doc(request.state).set(toDoc(request));
    },

    consumeAuthRequest(state) {
      const ref = db.collection(C.authRequests).doc(state);
      return db.runTransaction(async tx => {
        const snapshot = await tx.get(ref);
        if (!snapshot.exists) return null;
        // Deleted whether or not it is still valid, so a replay finds nothing.
        tx.delete(ref);
        return fromDoc<AuthRequest>(snapshot.data() ?? {}, ['expiresAt']);
      });
    },

    async deleteExpiredAuthRequests(before) {
      const snapshot = await db
        .collection(C.authRequests)
        .where('expiresAt', '<', Timestamp.fromDate(before))
        .limit(BATCH_LIMIT)
        .get();
      await deleteInBatches(db, snapshot.docs.map(doc => doc.ref));
      return snapshot.size;
    },

    async getConnection(userId) {
      const snapshot = await db.collection(C.connections).doc(userId).get();
      return snapshot.exists
        ? fromDoc<CalendarConnection>(snapshot.data() ?? {}, ['connectedAt', 'updatedAt'])
        : null;
    },

    async saveConnection(connection) {
      await db.collection(C.connections).doc(connection.userId).set(toDoc(connection));
    },

    async patchConnection(userId, patch) {
      await db.collection(C.connections).doc(userId).update(toDoc(patch));
    },

    async deleteConnection(userId) {
      await db.collection(C.connections).doc(userId).delete();
    },

    async listCalendars(userId) {
      const snapshot = await calendars.where('userId', '==', userId).get();
      return snapshot.docs.map(doc => fromDoc<LinkedCalendar>(doc.data(), CALENDAR_DATES));
    },

    async getCalendar(calendarId) {
      const snapshot = await calendars.doc(calendarId).get();
      return snapshot.exists ? fromDoc<LinkedCalendar>(snapshot.data() ?? {}, CALENDAR_DATES) : null;
    },

    mutateCalendar(calendarId, fn) {
      const ref = calendars.doc(calendarId);
      // Firestore retries the transaction on contention, which is why `fn` must be pure.
      return db.runTransaction(async tx => {
        const snapshot = await tx.get(ref);
        const before = snapshot.exists ? fromDoc<LinkedCalendar>(snapshot.data() ?? {}, CALENDAR_DATES) : null;
        const next = fn(before);
        if (!next) return { before, after: before };
        const after: LinkedCalendar = { ...next, id: calendarId, version: (before?.version ?? 0) + 1 };
        tx.set(ref, toDoc(after));
        return { before, after };
      });
    },

    async deleteCalendar(calendarId) {
      await calendars.doc(calendarId).delete();
    },

    async listBusySyncCalendars(cursor, limit) {
      let query = calendars.where('syncBusy', '==', true).orderBy(FieldPath.documentId()).limit(limit);
      if (cursor) query = query.startAfter(cursor);
      const snapshot = await query.get();
      return snapshot.docs.map(doc => fromDoc<LinkedCalendar>(doc.data(), CALENDAR_DATES));
    },

    async saveChannel(channel) {
      await channels.doc(channel.id).set(toDoc(channel));
    },

    async getChannel(channelId) {
      // Channel ids come from verified tokens, but a '/' would still make doc() throw.
      if (!channelId || channelId.includes('/')) return null;
      const snapshot = await channels.doc(channelId).get();
      return snapshot.exists ? fromDoc<WatchChannel>(snapshot.data() ?? {}, CHANNEL_DATES) : null;
    },

    async deleteChannel(channelId) {
      await channels.doc(channelId).delete();
    },

    async listChannelsExpiringBefore(before, limit) {
      const snapshot = await channels
        .where('expiresAt', '<', Timestamp.fromDate(before))
        .orderBy('expiresAt')
        .limit(limit)
        .get();
      return snapshot.docs.map(doc => fromDoc<WatchChannel>(doc.data(), CHANNEL_DATES));
    },

    async listBusyBlocks(calendarId) {
      const snapshot = await busy.where('calendarId', '==', calendarId).get();
      return snapshot.docs.map(doc => fromDoc<BusyBlock>(doc.data(), BUSY_DATES));
    },

    async applyBusyDiff(_calendarId, upserts, deleteIds) {
      const writes: ((batch: firestore.WriteBatch) => void)[] = [
        ...upserts.map(block => (batch: firestore.WriteBatch) => batch.set(busy.doc(block.id), toDoc(block))),
        ...deleteIds.map(id => (batch: firestore.WriteBatch) => batch.delete(busy.doc(id))),
      ];
      for (let i = 0; i < writes.length; i += BATCH_LIMIT) {
        const batch = db.batch();
        for (const write of writes.slice(i, i + BATCH_LIMIT)) write(batch);
        await batch.commit();
      }
    },

    async deleteBusyBlocks(calendarId) {
      const snapshot = await busy.where('calendarId', '==', calendarId).select().get();
      await deleteInBatches(db, snapshot.docs.map(doc => doc.ref));
    },
  };
}

/**
 * Host busy time for the availability engine: every block of the user overlapping [from, to).
 * Needs the composite index `calendarBusyBlocks (userId ASC, to ASC)`.
 */
export async function listHostBusy(
  db: firestore.Firestore,
  userId: string,
  range: { from: Date; to: Date }
): Promise<{ from: Date; to: Date }[]> {
  const snapshot = await db
    .collection(C.busy)
    .where('userId', '==', userId)
    .where('to', '>', Timestamp.fromDate(range.from))
    .get();
  return snapshot.docs
    .map(doc => fromDoc<BusyBlock>(doc.data(), BUSY_DATES))
    .filter(block => block.from < range.to)
    .map(block => ({ from: block.from, to: block.to }));
}
```

## Checklist

- [ ] Composite index `calendarBusyBlocks (userId, to)` deployed
- [ ] Rules deny client access to all five collections
- [ ] The availability engine reads host busy time through `listHostBusy` or an equivalent query
