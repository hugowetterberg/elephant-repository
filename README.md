# Elephant repository

![Image](docs/elephant.png?raw=true)

Elephant repository is a [NewsDoc](https://github.com/ttab/newsdoc) document repository with versioning, ACLs for permissions, archiving, validation schemas, workflow statuses, event output, real-time updates, and metrics for observability.

The repository depends on PostgreSQL for data storage and a S3 compatible store for archiving and assets. It can use AWS EventBridge as an event sink, but that is optional and can be disabled with `--no-eventsink`.

All operations against the repository are exposed as a [Twirp RPC API](https://twitchtv.github.io/twirp/docs/intro.html) that you can communicate with using either [Protobuf](https://protobuf.dev/) messages or standard JSON. See [Calling the API](#calling-the-api) for more details on communicating with the defined services.

## Table of contents

- [Architecture](#architecture)
  - [Project structure](#project-structure)
  - [Service layer](#service-layer)
  - [Background services](#background-services)
  - [Design patterns](#design-patterns)
- [Document lifecycle](#document-lifecycle)
  - [Versioning](#versioning)
  - [ACLs for permissions](#acls-for-permissions)
  - [Workflow statuses](#workflow-statuses)
  - [Attaching objects (files/assets)](#attaching-objects-filesassets)
  - [Document locking](#document-locking)
  - [Scheduled publishing](#scheduled-publishing)
- [Validation schemas](#validation-schemas)
- [Event system](#event-system)
  - [Eventlog](#eventlog)
  - [Event sink](#event-sink)
  - [Real-time APIs](#real-time-apis)
- [Archiving](#archiving)
  - [Signing](#signing)
  - [Deletes](#deletes)
  - [Restoring documents](#restoring-documents)
  - [Purging documents](#purging-documents)
- [Authentication and permissions](#authentication-and-permissions)
- [Observability](#observability)
- [Calling the API](#calling-the-api)
- [Running locally](#running-locally)
- [The database](#the-database)
- [Configuration reference](#configuration-reference)

## Architecture

### Project structure

```
cmd/repository/         Entry point, CLI flags, server wiring
repository/             Core business logic
  ├── documents         Document CRUD, versioning, status management
  ├── schemas           Schema validation service
  ├── workflows         Workflow rules and state management
  ├── metrics           Document metrics calculation and querying
  ├── archiver          Background S3 archiving with ECDSA signing
  ├── socket            WebSocket real-time API
  ├── sse               Server-Sent Events API
  └── store             PGDocStore (main data access layer)
postgres/               sqlc-generated query code and custom types
schema/                 Database migration files (tern, 24 migrations)
sinks/                  Event sink implementations (EventBridge)
internal/               Internal helpers (migration runner, CLI config, test utilities)
magefiles/              Mage build task definitions
```

### Service layer

The application exposes four Twirp RPC services that form the external API:

**Documents** (`elephant.repository.Documents`) is the primary service for document operations. It handles creating and updating documents, fetching documents by UUID (optionally by version or status), retrieving document history and metadata, managing ACLs and permissions, setting workflow statuses, managing file attachments, reading the eventlog, and managing document locks. It also provides operations for deleting, restoring, and purging documents.

**Schemas** (`elephant.repository.Schemas`) manages document type declarations and validation schemas. It provides methods for registering schema specifications (with versioning), listing document types and their active schemas, and configuring type-specific metadata like timespan and label expressions. These type configurations are used by downstream systems like [elephant-index](https://github.com/ttab/elephant-index) to create correct search index mappings.

**Workflows** (`elephant.repository.Workflows`) manages workflow rules for document statuses. Status rules use the [expr](https://github.com/expr-lang/expr) expression language to define constraints that are evaluated when statuses are set. This allows enforcing business rules such as requiring certain metadata before a document can be published.

**Metrics** (`elephant.repository.Metrics`) provides document metrics calculation and querying. Metrics are calculated by pluggable `MetricCalculator` implementations. The built-in calculator is a character counter that counts UTF-8 characters in text blocks (excluding vignettes). Metrics support different aggregation strategies: `REPLACE`, `INCREMENT`, and `NONE`.

### Background services

The server runs several background processes alongside the API:

- **EventLog builder** -- Polls the event outbox and converts entries into the sequential eventlog. Uses PostgreSQL advisory locks for distributed coordination so that only one instance processes events at a time.
- **Event forwarder** -- Reads the eventlog and forwards enriched events to an external event sink (currently AWS EventBridge). Tracks its position in the database for reliable delivery.
- **Archiver** -- Writes document versions, statuses, and events to S3 with ECDSA signatures forming a tamper-proof merkle chain. Also handles finalising deletes and processing restores.
- **Scheduler** -- Publishes documents that have been given a scheduled publish time by setting the appropriate status when the time arrives.

All background services can be individually disabled via feature toggles (see [Configuration reference](#configuration-reference)).

### Design patterns

**Outbox pattern** -- Document changes are written to an `event_outbox` table atomically within the same database transaction as the document change itself. The EventLog builder then asynchronously polls the outbox and converts entries into the sequential eventlog. This guarantees that events are never lost even if the event processing pipeline is temporarily unavailable.

**Pessimistic document locking** -- All updates to a document begin by acquiring a row lock on the `document` table for the duration of the transaction. This provides serialisation guarantees for writes to a single document and allows straightforward sequential numbering for document versions and status updates.

**Job locks for distribution** -- Background processes use PostgreSQL advisory locks (via the `elephantine` library) to coordinate across multiple instances. Only one instance will process events, archive documents, or run the scheduler at any given time. Stale locks are automatically cleaned up.

**FanOut notifications** -- The store layer maintains broadcast channels for internal notifications about schema updates, workflow changes, eventlog entries, and archived events. This allows real-time APIs (WebSocket, SSE) and internal caches to react immediately to changes without polling.

## Document lifecycle

A document in Elephant follows a well-defined lifecycle: creation, versioning, status management, and eventual deletion/archiving. At each stage the repository maintains a complete audit trail.

### Versioning

All updates to a document are stored as sequentially numbered versions with information about when it was created, who created it, and optional metadata for the version.

Versions are immutable once created. Old versions can always be fetched through the API, and the full history can be inspected through `Documents.GetHistory`. When fetching a document you can request a specific version number, or request the version that was last given a particular status (e.g. the version that was last marked "usable").

### ACLs for permissions

Documents can be shared with individuals or units (groups of people). By default only the entity that created a document has access to it, and other entities will have to be granted read and/or write access.

The available document-level permissions are:

| Permission | Code | Description |
|---|---|---|
| Read | `r` | View the document and its versions |
| Write | `w` | Create new versions of the document |
| Meta-write | `m` | Update document metadata |
| Set-status | `s` | Set workflow statuses on the document |

In most workflows documents will be shared with a group of people (a unit), but this model makes it possible to work with private drafts and share documents selectively with individuals. ACL changes are tracked in the eventlog.

See [docs/permissions.md](docs/permissions.md) for a detailed breakdown of which scopes and permissions are required for each API method.

### Workflow statuses

You can define and set statuses for document versions. To publish a version of a document you would typically set the status "usable" for it. Your publishing pipeline would then pick up that status event and act on it. New document versions that are created don't affect the "usable" status you set; to publish a new version you would have to create a new "usable" status update that references that version.

The last status of a given name for a document is referred to as the "head". Just like documents, statuses are versioned and have sequential IDs for a given document and status name.

Workflow rules can be defined through the `Workflows` service to enforce constraints on status transitions. Rules are written using the [expr](https://github.com/expr-lang/expr) expression language and are evaluated when a status is set. This allows you to enforce business rules such as requiring certain metadata fields to be present before a document can be marked as "usable".

### Attaching objects (files/assets)

It's possible to attach objects (files/assets) to documents. This can be done using the `Documents.CreateUpload` method to get an upload ID and URL. After making a PUT-request to the upload URL with the contents of the object the ID can be used together with a `Documents.Update` request that performs a document write to attach the object to the document.

When an object has been attached to a document that information is shown in the event for the update as `attached_objects`, conversely a detach is shown as `detached_objects`. Assets are also described in the response to `Documents.GetMeta`.

To download attachments use `Documents.GetAttachments` with `DownloadLink` set to true, and the response will include a link that the object contents can be downloaded from.

The actual attached objects are currently not being archived. They are, however, copied to the archive bucket if their document is deleted, so a document can be restored together with its attachments. Only the latest version of the currently attached objects are restored; backup of attachments has to be solved outside of the repository.

### Document locking

Documents support pessimistic locking through the `Documents.Lock`, `Documents.ExtendLock`, and `Documents.Unlock` methods. Locks have an expiration time and an owner, and prevent other users from making changes to the document while the lock is held. This is useful for editorial workflows where you want to ensure that only one person is editing a document at a time.

### Scheduled publishing

The scheduler background service monitors documents that have been given a scheduled publish time. When the scheduled time arrives the scheduler automatically sets the appropriate status on the document, effectively publishing it without manual intervention. This allows editorial teams to prepare content in advance and have it go live at a specific time. The scheduler can be disabled with `--no-scheduler`.

## Validation schemas

All document types need to be declared before they can be stored in the repository. This serves two purposes: the primary purpose is to maintain data quality by validating documents against their declared schema, and the secondary purpose is to inform automated systems about the shape of your data. This is leveraged by [elephant-index](https://github.com/ttab/elephant-index) to create correct mappings for OpenSearch/ElasticSearch.

Schema management is handled through the `Schemas` service. The repository uses the [revisor](https://github.com/ttab/revisor) validation engine with a set of built-in core schemas from [revisorschemas](https://github.com/ttab/revisorschemas) (covering core document types, metadocs, and planning documents). Custom schemas can be registered through the API with version tracking and activation control.

The validator watches for schema updates and automatically reloads when schemas change. It also tracks deprecated fields and reports deprecation counts as Prometheus metrics, so you can monitor how widely deprecated constructs are still in use.

For details on how to write specifications, see [revisor "Writing specifications"](https://github.com/ttab/revisor#writing-specifications).

## Event system

### Eventlog

All changes to a document are emitted on the eventlog, accessed through `Documents.Eventlog`. Changes can be:

* a new document version
* a new document status
* updated ACL entries
* a document delete
* a restore completion
* a workflow state change

Each event has a strictly sequential ID (enforced by a database trigger), a timestamp, the identity of the updater, and detailed information about what changed. Events also include information about attached/detached objects, timespans, and labels when applicable.

This eventlog can be used by other applications to act on changes in the repository. Consumers track their position in the eventlog by ID and can resume from where they left off.

### Event sink

The repository has the concept of event sinks where enriched events can be posted to an external event bus. Currently only AWS EventBridge is supported as a sink.

The purpose of the enriched events is to allow the construction of event-based architectures where, for example, a Lambda function could subscribe to published articles with a specific category. The enriched event format includes document metadata, so downstream consumers can filter events without having to load the full document. This avoids situations where many systems poll the repository unnecessarily just to determine if an event should be handled.

Events that exceed the EventBridge size limit (256KB) are tracked as rejected events. The event forwarder maintains its position in the eventlog and supports automatic restart with configurable retry logic.

### Real-time APIs

In addition to polling the eventlog, the repository provides two real-time push mechanisms:

**WebSocket** (`/websocket/{token}`) provides bidirectional real-time communication. Clients obtain a token through `Documents.GetSocketToken()`, which returns an ECDSA-signed token valid for 24 hours. The WebSocket API supports both Protobuf and JSON message encoding, with a 10KB maximum message size and rate limiting per connection. CORS validation is enforced based on configured allowed hosts.

**Server-Sent Events** (`/eventlog`) provides a streaming HTTP connection for event delivery. It supports topic-based filtering via query parameters and maintains a 200-event replay buffer for reconnection resilience (clients can resume from their last position using the `Last-Event-ID` header). Authentication requires the `eventlog_read` scope. As a fallback, the SSE provider polls every 5 seconds to guarantee delivery even if a notification is missed.

Both real-time APIs can be individually disabled with `--no-websocket` and `--no-sse`.

## Archiving

The repository has an archiving subsystem that records all eventlog events, and their associated document or status data, to a S3 compatible store. Archiving is tightly integrated with the document lifecycle -- a document cannot be deleted until it has been fully archived.

As part of the archiving process the archive objects are signed with an archiving key, and as the signature of the previous version is included in the object we create a tamper-proof chain of updates. Statuses also include the signature of the document version they refer to.

All archived objects (events, statuses, and documents) contain the signature of their parent, so the entire archive forms a narrow [merkle tree](https://en.wikipedia.org/wiki/Merkle_tree). This would for example allow for the publication of a transparency log, with which a trusted third party could verify what was in the repository at any given time.

The archive status and signature are fed back into the database after the object has been successfully archived.

### Signing

The repository maintains a set of ECDSA P-384 signing keys that are used to sign archived objects. The signature is an ASN1 signature of the SHA-256 hash of the marshalled data of the archive object. The format of a signature string looks like this:

```
v1.[key ID].[sha256 hash as raw URL base64].[signature as raw URL base64]
```

The signature is set as the metadata header `X-Amz-Meta-Elephant-Signature` on the S3 object itself. After the object has been archived the corresponding database row is updated with the signature of the archived object.

Event, status, and document version archive objects contain the signature of their parents to create a signature chain that can be verified. That means that everything that gets written can be verified against a log.

The reason that signing has to be done during archiving is that the jsonb data type isn't guaranteed to be byte stable. Verifying signatures on the archive objects is straightforward; verifying signatures for the database would have to be done by verifying the signature for the archive, and then verifying that the data in the database is "logically" equivalent to the archive data.

Signing keys are rotated on a 180-day cycle. A new signing key will be created and published 30 days before it's taken into use. Keys are stored in the `signing_keys` database table using a JWK-based key spec format, with the intent of exposing them through a JWKS-compatible endpoint in the future.

### Deletes

Archiving is used to support the delete functionality. A delete request will acquire a row lock for the document, and then wait for its versions and statuses to be fully archived. It then creates a `delete_record` with information about the delete, and deletes the document row to replace it with a `system_state = 'deleting'` placeholder. From the clients' standpoint the delete is now finished. But no reads of, or updates to the document are allowed until the delete has been finalised by an archiver. The reason that the archiver is responsible for finalising the delete is that we then can ensure that the database and S3 archive are consistent. Otherwise we would be forced to manage error handling and consistency across a database transaction and the object store.

The archiver looks for documents with pending deletes and then moves the objects from the `documents/[uuid]` prefix to a `deleted/[uuid]/[delete record id]` prefix in the bucket. Once the move is complete the document row is deleted, and the only thing that remains is the `delete_record` and the archived objects.

### Restoring documents

When a restore is initiated a system-locked document row is created in the `document` table (`system_state = 'restoring'`). This is not reflected in the eventlog, but all the restored document versions and status updates will be, and when the restore is finished a `restore_finished` event will be emitted. All eventlog events that result from a restore will have `system_state` set to `restoring` so that they can be ignored by event processors that don't need to react to restored data.

### Purging documents

When a document has been deleted the archived information associated with it can be purged. This will remove all objects in S3 and clear information about status heads, ACLs, and the document version from the delete record. The information that will remain is:

* UUID and URI of the document
* Type of the document
* The time the document was deleted and who deleted it
* The time the document was purged

So what remains is the bare-bones information that something existed, and an audit trail related to its removal.

## Authentication and permissions

The repository uses JWT-based authentication. Every API request must include a valid JWT in the `Authorization: Bearer` header. The JWT contains the following claims:

| Claim | Description |
|---|---|
| `sub` | Subject URI identifying the user (e.g. `user://tt/hugo`) |
| `sub_name` | Human-readable display name |
| `scope` | Space-separated list of granted scopes |
| `units` | Array of unit URIs the user belongs to (used for ACL matching) |

### Scopes

Scopes control access to API operations. The available scopes are:

| Scope | Description |
|---|---|
| `doc_admin` | Full access to all document operations |
| `doc_read_all` | Read any document regardless of ACL |
| `doc_read` | Read documents the user has ACL access to |
| `doc_write` | Create and update documents |
| `doc_delete` | Delete documents |
| `doc_restore` | Restore deleted documents |
| `doc_purge` | Purge deleted documents |
| `doc_import` | Use import directives in updates |
| `eventlog_read` | Read the eventlog and use SSE |
| `schema_admin` | Manage schemas and type configurations |
| `schema_read` | Read schema definitions |
| `workflow_admin` | Manage workflow rules |
| `metrics_admin` | Full access to metrics operations |
| `metrics_read` | Read metrics |
| `metrics_write` | Write metrics (supports subscopes, e.g. `metrics_write:word_count`) |
| `asset_upload` | Upload file attachments |

For a detailed breakdown of which scopes and ACL permissions are required for each API method, see [docs/permissions.md](docs/permissions.md).

## Observability

### Prometheus metrics

Prometheus metrics are exposed on port 1081 at `/metrics`. The metrics cover:

- API latency and error rates (via Twirp hooks)
- Archiver progress and event counts
- Event sink restarts, skipped events, and latency percentiles
- WebSocket open connections, rate limiting, and message counts
- Scheduler publication attempts
- Validation deprecation counts by label
- Go runtime metrics

### PPROF debugging

PPROF debugging endpoints are exposed on port 1081 at `/debug/pprof/`. These allow CPU and memory profiling to investigate performance issues and concurrency bugs.

### Health checks

The health check endpoints are also on port 1081:

- `/health/alive` -- Basic liveness check
- `/health/ready` -- Readiness check (verifies PostgreSQL and S3 connectivity)

### Logging

The repository uses Go's `slog` structured logging. Log entries include contextual fields such as error codes, service/method names, document UUIDs, and user identities. Error responses are automatically logged with appropriate levels: 400-class errors at INFO, 500-class errors at ERROR, and everything else at WARN. See [docs/logs.md](docs/logs.md) for details on the logging approach.

## Calling the API

The API is defined in [service.proto](https://github.com/ttab/elephant-api/blob/main/repository/service.proto). All endpoints are available as both Protobuf and JSON Twirp services at the base path `/twirp/elephant.repository.{Service}/{Method}`.

### Retrieving a token

For development purposes Elephant has an endpoint for fetching tokens. This is a password-less password grant where you can specify your own permissions and identity.

``` shell
curl http://localhost:1080/token \
    -d grant_type=password \
    -d 'username=Hugo Wetterberg <user://tt/hugo, unit://tt/unit/a, unit://tt/unit/b>' \
    -d 'scope=doc_read doc_write doc_delete'
```

This will yield a JWT with the following claims:

``` json
{
  "iss": "test",
  "sub": "user://tt/hugo",
  "exp": 1675894185,
  "sub_name": "Hugo Wetterberg",
  "scope": "doc_read doc_write doc_delete",
  "units": [
    "unit://tt/unit/a",
    "unit://tt/unit/b"
  ]
}
```

Example scripting usage:

``` shell
TOKEN=$(curl -s http://localhost:1080/token \
    -d grant_type=password \
    -d 'username=Hugo Wetterberg <user://tt/hugo, unit://tt/unit/a, unit://tt/unit/b>' \
    -d 'scope=doc_read doc_write doc_delete' | jq -r .access_token)

curl --request POST \
  --url http://localhost:1080/twirp/elephant.repository.Documents/Get \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
        "uuid": "23ba8778-36c2-417b-abc7-323db47a7472"
}'
```

### Fetching a document

``` shell
curl --request POST \
  --url http://localhost:1080/twirp/elephant.repository.Documents/Get \
  --header 'Content-Type: application/json' \
  --data '{
	"uuid": "8090ff79-030e-419b-952e-12917cfdaaac"
}'
```

Here you can specify `version` to fetch a specific version, or `status` to fetch the version that last got f.ex. the "usable" status.

### Fetching document metadata

``` shell
curl --request POST \
  --url http://localhost:1080/twirp/elephant.repository.Documents/GetMeta \
  --header 'Content-Type: application/json' \
  --data '{
	"uuid": "8090ff79-030e-419b-952e-12917cfdaaac"
}'
```

## Running locally

### Prerequisites

- Go 1.25.6+
- PostgreSQL (local via Docker, or remote)
- MinIO or S3-compatible storage
- [Mage](https://magefile.org/) task runner

### Preparing the environment

Follow [the instructions](#the-database) to get the database up and running.

Then create a `.env` file containing the following values:

```
S3_ENDPOINT=http://localhost:9000/
S3_ACCESS_KEY_ID=minioadmin
S3_ACCESS_KEY_SECRET=minioadmin
MOCK_JWT_SIGNING_KEY='MIGkAgEBBDAgdjcifmVXiJoQh7IbTnsCS81CxYHQ1r6ftXE6ykJDz1SoQJEB6LppaCLpNBJhGNugBwYFK4EEACKhZANiAAS4LqvuFUwFXUNpCPTtgeMy61hE-Pdm57OVzTaVKUz7GzzPKNoGbcTllPGDg7nzXIga9ObRNs8ytSLQMOWIO8xJW35Xko4kwPR_CVsTS5oMaoYnBCOZYEO2NXND7gU7GoM'
```

The server will generate a JWT signing key (and log a warning) if it's missing from the environment.

### Running the repository server

The repository server runs the API, archiver, and eventlog builder. If your environment has been set up correctly (env vars, postgres, and minio) you should be able to run it like this:

``` shell
go run ./cmd/repository run --no-eventsink
```

### Running tests

Tests use `dockertest` to spin up real PostgreSQL and MinIO containers, so Docker must be running:

``` shell
go test ./...
```

### Linting

``` shell
golangci-lint run --timeout=4m
```

## The database

### Running and DB schema ops

The repository uses [mage](https://magefile.org/) as a task runner. Start a local postgres instance using `mage sql:postgres pg16`. Create a database using `mage sql:db`.

The database schema is defined using numbered [tern](https://github.com/jackc/tern) migrations in `./schema/`. Initialise the schema by running `mage sql:migrate`. Set the `CONN_STRING` environment variable to run the `mage sql:*` operations against a remote database.

Start a local minio instance and the necessary buckets using `mage s3:minio s3:bucket elephant-archive s3:bucket elephant-assets`.

Queries are defined in `./postgres/query.sql` and are compiled using [sqlc](https://sqlc.dev/) to a `Queries` struct in `./postgres/query.sql.go`. Run `mage sql:generate` to compile queries. Never edit the generated files (`query.sql.go`, `models.go`) directly.

Use `mage sql:rollback 0` to undo all migrations, or to migrate to a specific version, f.ex. 7, use `mage sql:rollback 7`.

Connect to the local database using `psql $(mage connstring)` or `psql postgres://elephant-repository:pass@localhost/elephant-repository`.

### Introduction to the schema

Each document has a single row in the `document` table. New versions of the document get added to the `document_version` table, and `document(updated, updater_uri, current_version)` is updated at the same time. The same logic applies to `document_status` and `status_heads(id, updated, updater_uri)`. This relationship between the tables is formalised in the stored procedures `create_version` and `create_status`.

An update to a document always starts with getting a row lock on the `document(uuid)` table for the transaction. This gives us serialisation guarantees for writes to a single document, and lets us use straightforward numbering for document versions and status updates.

The key tables in the schema are:

| Table | Purpose |
|---|---|
| `document` | Main document rows (UUID, URI, type, timestamps, current version, workflow state) |
| `document_version` | Immutable version records (document data, metadata, archive signature) |
| `document_status` | Workflow statuses per document (versioned, with signatures) |
| `acl` | Access control lists (document UUID, URI, permissions array) |
| `status_heads` | Current status pointers for efficient head lookup |
| `eventlog` | Sequential event stream with strict ID ordering |
| `event_outbox` | Transient outbox for event building (outbox pattern) |
| `document_lock` | Pessimistic locks with expiration |
| `delete_record` | Soft delete tracking and restoration metadata |
| `document_schema` | Schema specifications (JSONB storage) |
| `signing_keys` | ECDSA signing keys (JWK format) |
| `attached_object` | File/asset attachment tracking |
| `type_configuration` | Document type metadata (timespan, label expressions) |
| `scheduled_document` | Documents awaiting scheduled publication |

### Data mining examples

#### Published article cause

`¤` is `NULL`, in other words it's the initial publication of an article.

``` sql
SELECT date(s.created), s.meta->>'cause' AS cause, COUNT(*) AS num
FROM document_status AS s
WHERE s.name='usable'
GROUP BY date(s.created), cause
ORDER BY date(s.created), cause NULLS FIRST;
```

```
    date    │    cause    │ num
════════════╪═════════════╪═════
 2023-02-07 │ ¤           │ 620
 2023-02-07 │ correction  │   4
 2023-02-07 │ development │  64
 2023-02-07 │ fix         │  10
 2023-02-08 │ ¤           │ 734
 2023-02-08 │ correction  │   3
 2023-02-08 │ development │  97
 2023-02-08 │ fix         │  14
 2023-02-09 │ ¤           │ 613
 2023-02-09 │ correction  │   5
 2023-02-09 │ development │  89
 2023-02-09 │ fix         │   8
 2023-02-10 │ ¤           │ 428
 2023-02-10 │ correction  │   2
 2023-02-10 │ development │  52
 2023-02-10 │ fix         │  12
(16 rows)
```

#### Time to correction after first publish

``` sql
SELECT s.uuid, i.created AS initially_published, s.created-i.created AS time_to_correction
FROM document_status AS s
     INNER JOIN document_status AS i
           ON i.uuid = s.uuid AND i.name = s.name AND i.id = 1
WHERE s.name='usable' AND s.meta->>'cause' = 'correction'
ORDER BY s.created;
```

```
                 uuid                 │  initially_published   │    time_to_correction
══════════════════════════════════════╪════════════════════════╪═══════════════════════════
 54123854-9303-4cc6-b98d-afa9b2656602 │ 2023-02-07 09:19:50+00 │ @ 11 mins 55 secs
 eedf4fe2-5b3a-4fa4-a2c8-cf2029ca268b │ 2023-02-07 09:20:58+00 │ @ 1 hour 59 mins 30 secs
 03d47f19-a4b5-4de5-b6e2-664d759683ec │ 2023-02-07 12:58:07+00 │ @ 4 mins 34 secs
 37041f9b-386b-47f5-a974-f054bb628292 │ 2023-02-07 13:10:55+00 │ @ 17 mins 5 secs
 f550fbce-6c8c-43cc-a31d-0cbdb464a681 │ 2023-02-08 05:15:02+00 │ @ 1 hour 13 mins 13 secs
 f550fbce-6c8c-43cc-a31d-0cbdb464a681 │ 2023-02-08 05:15:02+00 │ @ 3 hours 15 mins 2 secs
 6ee43615-2cb8-441a-9c0f-fb68a675e1f2 │ 2023-02-08 08:30:02+00 │ @ 3 mins 56 secs
 5d75600e-4d26-488e-bcd2-1c27bd05794f │ 2023-02-09 01:30:02+00 │ @ 1 hour 2 mins 31 secs
 629ddc10-47e0-46ae-b47d-6c9fbb3ad7e0 │ 2023-02-09 08:24:37+00 │ @ 1 hour 27 mins 13 secs
 44e6653b-8be7-4175-8e4c-0c24c132e774 │ 2023-02-09 10:36:31+00 │ @ 5 hours 9 mins 25 secs
 71b61828-510d-4a6b-a8fa-574101eb54f5 │ 2023-02-09 08:30:26+00 │ @ 9 hours 54 mins 52 secs
 be6c03f8-81d1-40dd-bbe1-9b0c727b39a8 │ 2023-02-09 09:54:13+00 │ @ 8 hours 40 mins 27 secs
 d6413696-d189-4ad0-9454-8f0681a3f541 │ 2023-02-10 05:00:02+00 │ @ 1 hour 32 mins 2 secs
(13 rows)
```

#### High newsvalue articles per section

``` sql
SELECT vs.section, vs.newsvalue, COUNT(*)
FROM (
     SELECT d.uuid, s.created,
            (jsonb_path_query_first(
                v.document_data,
                '$.meta[*] ? (@.type == "core/newsvalue").data'
            )->>'score')::int AS newsvalue,
            jsonb_path_query_first(
                v.document_data,
                '$.links[*] ? (@.rel == "subject" && @.type == "core/section")'
            )->>'title' AS section
     FROM document_status AS s
          INNER JOIN document AS d ON d.uuid = s.uuid
          INNER JOIN document_version AS v
                ON v.uuid = d.uuid
                   AND v.version = d.current_version
                   AND v.type = 'core/article'
     WHERE
        s.name='usable'
        AND s.id = 1
        AND date(s.created) = '2023-02-08'
) AS vs
WHERE vs.newsvalue <= 2 AND newsvalue > 0
GROUP BY vs.section, vs.newsvalue
ORDER BY vs.section, vs.newsvalue;
```

```
 section │ newsvalue │ count
═════════╪═══════════╪═══════
 Ekonomi │         1 │     2
 Ekonomi │         2 │     5
 Inrikes │         1 │     2
 Inrikes │         2 │    12
 Kultur  │         2 │     2
 Nöje    │         2 │     5
 Sport   │         1 │     4
 Sport   │         2 │     7
 Utrikes │         1 │     2
 Utrikes │         2 │     7
(10 rows)
```

## Configuration reference

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `ADDR` | `:1080` | API listen address |
| `TLS_ADDR` | `:1443` | TLS listener address |
| `PROFILE_ADDR` | `:1081` | Health/metrics/pprof address |
| `LOG_LEVEL` | `error` | Log verbosity (`debug`, `info`, `warn`, `error`) |
| `CONN_STRING` | `postgres://elephant-repository:pass@localhost/elephant-repository` | PostgreSQL connection string |
| `S3_ENDPOINT` | (AWS default) | S3/MinIO endpoint URL |
| `S3_ACCESS_KEY_ID` | - | S3 static credentials (access key) |
| `S3_ACCESS_KEY_SECRET` | - | S3 static credentials (secret key) |
| `ARCHIVE_BUCKET` | `elephant-archive` | S3 bucket for document archive |
| `ASSET_BUCKET` | `elephant-assets` | S3 bucket for uploaded assets |
| `DEFAULT_LANGUAGE` | `sv-se` | Default document language |
| `DEFAULT_TIMEZONE` | `Europe/Stockholm` | Default timezone |
| `EVENTSINK` | `aws-eventbridge` | Event sink type |
| `CORS_HOSTS` | - | Comma-separated list of allowed CORS hosts |
| `TLS_CERT` | - | Path to TLS certificate file |
| `TLS_KEY` | - | Path to TLS key file |
| `ENSURE_SCHEMA` | - | Comma-separated schema specs to load at startup (`name@version:URL`) |
| `MIGRATE_DB` | `false` | Automatically run migrations on startup (development only) |
| `TOLERATE_EVENTLOG_GAPS` | `false` | Allow gaps in eventlog IDs (for legacy data) |

### Feature toggles

| Flag / env var | Description |
|---|---|
| `--no-archiver` / `NO_ARCHIVER` | Disable S3 archiving |
| `--no-eventsink` / `NO_EVENTSINK` | Disable event forwarding to EventBridge |
| `--no-eventlog-builder` / `NO_EVENTLOG_BUILDER` | Disable eventlog building from outbox |
| `--no-scheduler` / `NO_SCHEDULER` | Disable scheduled publishing |
| `--no-charcounter` / `NO_CHARCOUNTER` | Disable built-in character count metrics |
| `--no-websocket` / `NO_WEBSOCKET` | Disable WebSocket real-time API |
| `--no-sse` / `NO_SSE` | Disable Server-Sent Events API |

## Related projects

Elephant repository is part of a larger ecosystem of packages:

| Package | Description |
|---|---|
| [elephant-api](https://github.com/ttab/elephant-api) | Protobuf/Twirp service definitions and generated code |
| [elephant-index](https://github.com/ttab/elephant-index) | Search indexing for OpenSearch/ElasticSearch |
| [elephantine](https://github.com/ttab/elephantine) | Shared utilities: auth, logging, metrics, health checks, HTTP middleware |
| [newsdoc](https://github.com/ttab/newsdoc) | NewsDoc document format type definitions |
| [revisor](https://github.com/ttab/revisor) | Document schema validation engine |
| [revisorschemas](https://github.com/ttab/revisorschemas) | Built-in schema definitions (core, metadoc, planning) |
| [langos](https://github.com/ttab/langos) | Language and locale handling |
