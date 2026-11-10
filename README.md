# pg-couch

Framework-independent local-first relational data. PGlite provides durable local
PostgreSQL queries, PouchDB/CouchDB transports transactions, and PostgreSQL makes
authoritative decisions.

- Offline CRUD and reactive RxJS queries.
- Durable SQL outbox and automatic retry through PouchDB replication.
- Atomic multi-table transactions, including local callbacks that read their writes.
- Server authorization, constraint checks and idempotent transaction receipts.
- Optimistic reconciliation and rejection reporting.
- SharedWorker support for one database owner across browser tabs.

This project extracts the PGCouch implementation from
`@jcbuisson/express-x-plugins`, retaining its transaction document protocol.

## Install

Node.js 20+ is required

```sh
npm install pg-couch @electric-sql/pglite pouchdb-browser
# Server application:
npm install pg-couch pg pouchdb
# SharedWorker entry also needs:
npm install pouchdb-adapter-indexeddb
```

## Client

```js
import { PGlite } from '@electric-sql/pglite'
import PouchDB from 'pouchdb-browser'
import { createPGCouchClient } from 'pg-couch/client'

const models = [{ name: 'tasks', primaryKey: 'id' }]
const localDb = new PGlite('idb://my-app')
await localDb.exec(`CREATE TABLE IF NOT EXISTS tasks (
  id UUID PRIMARY KEY, title TEXT NOT NULL
)`)
const couch = new PouchDB('my-app-transactions')
const client = createPGCouchClient({
  models, localDb, couch,
  remote: '/couchdb/my-app',
  onError: console.error,
})
await client.ready
const tasks = client.model('tasks')
await tasks.create({ title: 'Available offline' })
const subscription = tasks.getObservable({}).subscribe(console.log)
await client.transaction(async tx => {
  const task = await tx.tasks.create({ title: 'Atomic write' })
  await tx.tasks.update(task.id, { title: 'Updated in the same transaction' })
})
console.log(await client.getStatus()) // pending, failed, blocked, online

// On application shutdown:
subscription.unsubscribe()
await client.dispose()
await couch.close()
await localDb.close()
```

`createPGCouchClient` returns `ready`, `model(name)`, `transaction(operationsOrCallback)`,
`start()`, `stop()`, `getStatus()`, `flushQueue()`, `waitForIdle()` and `dispose()`.
Models expose CRUD, reactive `getObservable`, and lifecycle methods. Mutations
resolve after the durable local commit; server acceptance arrives later.
`flushQueue()` performs local publication/reconciliation and does not promise
remote synchronization. `online` reports local loop/network status, not verified
server reachability. Caller-owned databases remain open after disposal.

Use matching application schemas locally and on the server. String model names
default to primary key `uid`; objects can specify `name`, `table` and `primaryKey`.
UUID primary keys are generated locally when omitted.

## Server

```js
import { Pool } from 'pg'
import PouchDB from 'pouchdb'
import { createPGCouchServer, pgcouchValidationDocument } from 'pg-couch/server'

const db = new Pool({ connectionString: process.env.DATABASE_URL })
// Create/migrate business tables before starting the server.
const couch = new PouchDB(process.env.COUCH_DATABASE_URL)
// Provision once with an admin connection before clients replicate:
// await couch.put(pgcouchValidationDocument())
const server = createPGCouchServer({
  db, couch,
  models: [{ name: 'tasks', primaryKey: 'id' }],
  authorize(context, operation) {
    return policy(context, operation) // your application's authorization policy
  },
  contextForDocument(doc) {
    return trustedContextForDatabase() // use your trusted database/account mapping
  },
  onError: console.error,
})
await server.ready

// In a route/RPC handler, pass the authenticated request context explicitly:
const tasks = server.model('tasks', { userId: 'authenticated-user' })
await tasks.create({ title: 'Created on server' })
await server.submit([
  { action: 'update', table: 'tasks', where: {}, data: { title: 'Updated' } },
], { userId: 'authenticated-user' }, 'transaction:unique-request-id')

// On shutdown:
await server.dispose()
await couch.close()
await db.end()
```

`createPGCouchServer` returns `ready`, `models`, `model(name, context)`,
`submit(operations, context, optionalId)`, `transaction` (alias of `submit`),
`flushQueue()` and `dispose()`. Server model methods are `findMany`, `findUnique`,
`create`, `update`, and `delete`. Server submissions resolve after the PostgreSQL
commit. Reuse the same transaction ID and operations for idempotent retries.

Authorization defaults to allow. A replicated document does not carry verified
request identity: derive its context using a trusted mapping. CouchDB database
membership defines a shared read scope; use separate scopes for separate tenants.
Provision the validator, CouchDB `_security`, authentication and browser CORS before
allowing client replication. Give only the server account the `pgcouch-server` role.

Both factories are also available from `pg-couch`. Use `/client` and `/server`
entry points to keep browser and server bundles focused on their respective code.

## Shared browser worker

```js
// sync.worker.js — call synchronously, before top-level await
import { startPGCouchWorker } from 'pg-couch/worker'
import schema from './schema.sql?raw'
startPGCouchWorker({
  schema, models: [{ name: 'tasks', primaryKey: 'id' }],
  remote: '/couchdb/my-app', storagePrefix: 'my-app',
})
```

```js
// Application (Vite example)
import workerUrl from './sync.worker.js?sharedworker&url'
import { storageForRemote } from 'pg-couch/storage'
const storage = await storageForRemote('/couchdb/my-app', location.origin, 'my-app')
const worker = new SharedWorker(workerUrl, { type: 'module', name: storage.worker })
worker.port.start()
```

The worker exposes a message protocol for calls, subscriptions, array transactions
and network control. See [the protocol and worker reference](docs/protocol.md).
The default storage prefix is `pg-couch`. When migrating an existing application,
pass its previous prefix explicitly to preserve its local database names. A direct
client requires the application to coordinate multi-tab database ownership.

## Constraints and tests

Route all replicated PostgreSQL writes through this package; external SQL changes
are not captured. Keep related tables in one scope and use deferrable foreign keys.
Reconciliation currently rebuilds application tables from the authoritative local
copy and pending transactions. History grows over time; pruning is not implemented.
There is no automatic semantic merge of incompatible relational transactions.

```sh
npm test
PGCOUCH_TEST_DATABASE_URL=postgresql://localhost/postgres npm run test:integration
# Add a disposable-test-capable CouchDB admin URL for real transport/role checks:
PGCOUCH_TEST_DATABASE_URL=postgresql://localhost/postgres \
PGCOUCH_TEST_COUCH_URL=http://admin:password@localhost:5984 npm run test:integration
```

The default suite uses PGlite and in-memory PouchDB. The external integration test
skips unless its PostgreSQL URL is configured. It creates and removes isolated test
resources. See [the protocol reference](docs/protocol.md) for schema, recovery,
replication, authorization and operational requirements.
# pg-couch
