# CLAUDE.md

This file provides guidance for AI assistants working with the Elephant Repository codebase.

## Project Overview

Elephant Repository is a NewsDoc document repository system written in Go. It provides full document versioning, ACL-based permissions, S3 archiving, validation schemas, workflow statuses, event streaming (EventBridge), real-time updates (WebSocket/SSE), and Prometheus metrics. The API layer uses Twirp RPC (Protocol Buffers + JSON).

## Build & Run Commands

### Prerequisites

- Go 1.25.6+
- PostgreSQL (local or Docker)
- MinIO or S3-compatible storage
- [Mage](https://magefile.org/) task runner

### Common Commands

```bash
# Run all tests (requires Docker for integration test containers)
go test ./...

# Lint (matches CI)
golangci-lint run --timeout=4m

# Start local PostgreSQL via Mage
mage sql:postgres

# Create database
mage sql:db

# Run database migrations
mage sql:migrate

# Rollback last migration
mage sql:rollback

# Start local MinIO S3
mage s3:minio

# Regenerate sqlc queries after modifying postgres/query.sql or postgres/schema.sql
mage sql:generate

# Grant reporting permissions on all tables
mage grantreporting

# Run the server (reads .env for configuration)
go run ./cmd/repository run
```

### Environment Configuration

The server reads a `.env` file at startup. Key environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `CONN_STRING` | `postgres://elephant-repository:pass@localhost/elephant-repository` | PostgreSQL connection |
| `S3_ENDPOINT` | (AWS default) | S3/MinIO endpoint |
| `S3_ACCESS_KEY_ID` | - | S3 static credentials |
| `S3_ACCESS_KEY_SECRET` | - | S3 static credentials |
| `ARCHIVE_BUCKET` | `elephant-archive` | S3 bucket for document archive |
| `ASSET_BUCKET` | `elephant-assets` | S3 bucket for uploaded assets |
| `LOG_LEVEL` | `error` | Log verbosity |
| `ADDR` | `:1080` | API listen address |
| `PROFILE_ADDR` | `:1081` | Health/metrics/pprof address |

Feature toggles: `NO_ARCHIVER`, `NO_EVENTSINK`, `NO_EVENTLOG_BUILDER`, `NO_SCHEDULER`, `NO_CHARCOUNTER`, `NO_WEBSOCKET`, `NO_SSE`.

## Architecture

```
cmd/repository/     Entry point, CLI flags, server wiring
repository/         Core business logic
  ├── documents     Document CRUD, versioning, status management
  ├── schemas       Schema validation service
  ├── workflows     Workflow rules and state management
  ├── metrics       Document metrics calculation and querying
  ├── archiver      Background S3 archiving with ECDSA signing
  ├── socket        WebSocket real-time API
  ├── sse           Server-Sent Events API
  └── store         PGDocStore (main data access layer)
postgres/           sqlc-generated query code and custom types
schema/             Database migration files (tern, 24 migrations)
sinks/              Event sink implementations (EventBridge)
internal/           Internal helpers (migration runner, CLI config, test utilities)
magefiles/          Mage build task definitions
```

### Key Layers

1. **API Layer** - Twirp RPC services mounted on httprouter
2. **Service Layer** - `DocumentsService`, `SchemasService`, `WorkflowsService`, `MetricsService`
3. **Store Layer** - `PGDocStore` handles all document persistence, versioning, ACLs
4. **Database Layer** - PostgreSQL via pgx/v5 with sqlc-generated prepared statements
5. **Archive Layer** - Background archiver writes to S3 with ECDSA-signed metadata
6. **Event Layer** - Outbox pattern → EventLog builder → EventBridge sink

### External Dependencies (ttab ecosystem)

- `elephant-api` - Protobuf/Twirp service definitions (generated code)
- `elephantine` - Shared utilities: auth, logging, metrics, health checks, HTTP helpers
- `revisor` - Document schema validation engine
- `newsdoc` - NewsDoc document format types
- `langos` - Language/locale handling
- `mage` - Build task library (SQL, S3 setup)
- `revisorschemas` - Built-in schema definitions

## Code Conventions

### Go Style

- **Formatter**: gofumpt (strict superset of gofmt)
- **Import ordering**: gci with standard → third-party → local grouping
- **Linter**: golangci-lint v2.7 with 32+ linters enabled (see `.golangci.yml`)
- Errors must be wrapped: `fmt.Errorf("operation: %w", err)`
- No `FIXME` or `BUG` comments (enforced by godox linter)
- Blank lines required before returns (nlreturn linter)
- Tests must be in `_test` packages (testpackage linter)
- No naked returns (nakedret linter)
- Max line length enforced (lll linter)
- Errors from external packages must be wrapped (wrapcheck linter)

### Naming

- Interfaces use verb-based names: `DocStore`, `DocumentValidator`, `WorkflowProvider`, `MetricCalculator`
- Services use `*Service` suffix: `DocumentsService`, `SchemasService`
- Store implementations use `PG` prefix: `PGDocStore`
- Method receivers use 1-2 character abbreviations
- Abbreviation renames in sqlc: `uuid` → `UUID`, `uri` → `URI`, `url` → `URL`
- Interface compliance guards: `var _ Interface = &Implementation{}`

### Database

- Migrations are in `schema/` using [tern](https://github.com/jackc/tern) format, numbered 001-024
- SQL queries live in `postgres/query.sql`, code generated by sqlc into `postgres/`
- After modifying `postgres/query.sql` or `postgres/schema.sql`, run `mage sql:generate`
- Never edit `postgres/query.sql.go` or `postgres/models.go` directly (they are generated)
- Custom types for sqlc column overrides are defined in `postgres/` package files

### Testing

- Integration tests use `dockertest` to spin up real PostgreSQL and MinIO containers
- Test utilities in `internal/test/` provide helpers for JWT tokens, Twirp clients, and backing services
- Tests use `t.Context()` for context management
- Both Protobuf and JSON Twirp clients are tested

## CI/CD

- **Tests**: `go test ./...` runs on every push (`.github/workflows/test.yaml`)
- **Lint**: golangci-lint v2.7 runs on every push (`.github/workflows/lint.yaml`)
- **Build**: Multi-platform Docker images (amd64/arm64) published to ghcr.io on version tags (`.github/workflows/build.yaml`)
- **Dependabot**: Weekly updates for Go modules, Docker, and GitHub Actions

## Key Files

| File | Purpose |
|---|---|
| `cmd/repository/main.go` | Application entry point and server wiring |
| `repository/documents.go` | Documents service implementation |
| `repository/store.go` | PGDocStore - main data access layer |
| `repository/archiver.go` | Background S3 archiving |
| `repository/validator.go` | Schema validation |
| `repository/workflows.go` | Workflow rule engine |
| `postgres/query.sql` | All SQL queries (sqlc source) |
| `postgres/schema.sql` | Database schema definition |
| `schema/` | Tern migration files |
| `.golangci.yml` | Linter configuration |
| `sqlc.yaml` | sqlc code generation config |
