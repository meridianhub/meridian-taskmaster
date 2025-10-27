# Platform Services PRD

**Tag**: `platform`
**Folder**: `internal/platform/`, `migrations/`
**Dependencies**: `infra`, `api-contracts`

## Overview

Build the shared platform services that all BIAN service domains depend on: database layer, event streaming, observability, authentication, and idempotency guarantees.

## Scope

### In Scope
- Database connection pooling and health checks (CockroachDB/YugabyteDB)
- Migration system for schema versioning
- Kafka producer/consumer framework
- OpenTelemetry tracing, Prometheus metrics, structured logging
- OAuth 2.0 / JWT authentication middleware
- RBAC authorization framework
- Redis-based idempotency layer with distributed locking
- Health check endpoints (/health/live, /health/ready)

### Out of Scope
- Domain-specific business logic (handled by service domain tags)
- gRPC service implementations (handled by service domain tags)
- Proto definitions (handled by api-contracts tag)

## Technical Requirements

### TR-1: Database Layer

File: `internal/platform/db/db.go`

**Requirements**:
- Connection pooling with configurable limits (default: 50 max connections)
- Support for both CockroachDB and YugabyteDB via standard `database/sql`
- Connection health checks with automatic retry
- Graceful shutdown with connection draining
- Context-aware query execution with timeouts
- Transaction management helpers

**Interface**:
```go
type DB interface {
    Ping(ctx context.Context) error
    BeginTx(ctx context.Context, opts *sql.TxOptions) (*sql.Tx, error)
    ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
    Close() error
}
```

### TR-2: Migration System

Tool: golang-migrate or goose
Location: `migrations/`

**Requirements**:
- Version-based migration files (up/down)
- Support for both SQL and Go migrations
- Atomic migration execution with rollback
- Migration status tracking in database
- CLI tool for applying migrations
- Integration with CI/CD for automated migrations

**Example migration**:
```sql
-- migrations/000001_initial_schema.up.sql
CREATE TABLE financial_booking_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    financial_account_type TEXT NOT NULL,
    base_currency CHAR(3) NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_financial_booking_logs_status ON financial_booking_logs(status);
```

### TR-3: Event Streaming (Kafka)

File: `internal/platform/events/kafka.go`

**Requirements**:
- Kafka producer with:
  - At-least-once delivery semantics
  - Idempotent producer configuration
  - Partitioning strategy (by account ID)
  - Serialization (JSON or Protobuf)
  - Error handling and retry logic
- Kafka consumer framework with:
  - Consumer group management
  - Offset commit strategies
  - Dead letter queue for failed messages
  - Graceful shutdown
- Event schema definitions

**Interface**:
```go
type EventPublisher interface {
    Publish(ctx context.Context, topic string, key string, event proto.Message) error
    Close() error
}

type EventHandler func(ctx context.Context, key string, value []byte) error

type EventConsumer interface {
    Subscribe(topic string, handler EventHandler) error
    Start(ctx context.Context) error
    Close() error
}
```

### TR-4: Observability

**OpenTelemetry Tracing** (`internal/platform/observability/tracing.go`):
- Initialize tracer with OTLP exporter
- gRPC interceptors for automatic span creation
- Context propagation across service boundaries
- Trace sampling configuration

**Prometheus Metrics** (`internal/platform/observability/metrics.go`):
- Standard metrics:
  - `http_requests_total` (counter)
  - `http_request_duration_seconds` (histogram)
  - `grpc_server_handled_total` (counter)
  - `grpc_server_handling_seconds` (histogram)
  - `db_query_duration_seconds` (histogram)
  - `kafka_messages_published_total` (counter)
- Custom business metrics registration

**Structured Logging** (`internal/platform/observability/logging.go`):
- JSON-formatted logs with correlation IDs
- Log levels: debug, info, warn, error
- Context-aware logging with trace/span IDs
- No sensitive data in logs (PII redaction)

### TR-5: Authentication & Authorization

**JWT Middleware** (`internal/platform/auth/jwt.go`):
- JWT token validation with RS256
- Public key rotation support
- Claims extraction (user ID, roles, scopes)
- Token expiration checking
- gRPC interceptor for authentication

**RBAC System** (`internal/platform/auth/rbac.go`):
- Role definitions:
  - `admin` - Full access
  - `operator` - Read/write to all accounts
  - `auditor` - Read-only access
  - `service` - Service-to-service communication
- Permission checking middleware
- Service-level authorization rules

**OAuth 2.0 Integration** (`internal/platform/auth/oauth.go`):
- Support for external identity providers (Auth0, Okta, etc.)
- Token introspection endpoint
- Client credentials flow for service accounts

### TR-6: Idempotency Layer

File: `internal/platform/idempotency/idempotency.go`

**Requirements**:
- Redis-based idempotency key storage
- Idempotency key format: `{service}:{operation}:{user_provided_key}`
- TTL-based expiration (configurable, default 24 hours)
- Distributed locking for concurrent requests (Redis SETNX)
- Store operation result for duplicate request responses
- Automatic cleanup of expired keys

**Interface**:
```go
type IdempotencyChecker interface {
    Check(ctx context.Context, key string) (exists bool, result []byte, err error)
    Store(ctx context.Context, key string, result []byte, ttl time.Duration) error
    AcquireLock(ctx context.Context, key string, ttl time.Duration) (acquired bool, err error)
    ReleaseLock(ctx context.Context, key string) error
}
```

### TR-7: Health Checks

File: `internal/platform/health/health.go`

**Liveness Probe** (`/health/live`):
- HTTP 200 if process is running
- No external dependency checks
- Kubernetes uses this for restart decisions

**Readiness Probe** (`/health/ready`):
- Check database connectivity (ping with timeout)
- Check Kafka connectivity (list topics)
- Check Redis connectivity (ping)
- HTTP 200 only if all dependencies healthy
- Kubernetes uses this for traffic routing

**Dependency Status** (`/health/status`):
- Detailed health of each dependency
- Response time metrics
- JSON response format

## Acceptance Criteria

- [ ] Database connections pool correctly with health checks
- [ ] Migrations apply successfully to fresh database
- [ ] Kafka producer publishes events reliably
- [ ] Kafka consumer processes events with error handling
- [ ] OpenTelemetry traces propagate across services
- [ ] Prometheus metrics endpoint exposes all standard metrics
- [ ] Structured logs include correlation IDs
- [ ] JWT authentication validates tokens correctly
- [ ] RBAC denies unauthorized access
- [ ] Idempotency prevents duplicate transaction processing
- [ ] Health checks accurately reflect dependency status
- [ ] All platform code has >90% test coverage

## Success Metrics

- Database connection pool utilization: < 70% under load
- Event publishing latency: P99 < 10ms
- Trace overhead: < 1% performance impact
- Idempotency check latency: P99 < 5ms
- Health check response time: < 100ms
- Zero false negatives on health checks

## Folder Structure

```
internal/
└── platform/
    ├── db/
    │   ├── db.go
    │   ├── db_test.go
    │   └── health.go
    ├── events/
    │   ├── kafka.go
    │   ├── kafka_test.go
    │   ├── producer.go
    │   └── consumer.go
    ├── observability/
    │   ├── tracing.go
    │   ├── metrics.go
    │   └── logging.go
    ├── auth/
    │   ├── jwt.go
    │   ├── rbac.go
    │   └── oauth.go
    ├── idempotency/
    │   ├── idempotency.go
    │   └── redis.go
    └── health/
        └── health.go

migrations/
├── 000001_initial_schema.up.sql
├── 000001_initial_schema.down.sql
└── README.md
```

## Testing Strategy

- Unit tests for all platform components with mocks
- Integration tests using testcontainers:
  - PostgreSQL/CockroachDB container for database tests
  - Kafka container for event streaming tests
  - Redis container for idempotency tests
- Performance benchmarks for critical paths
- Chaos testing for failure scenarios

## Notes

- Platform code must be highly reusable and generic
- No domain-specific logic in platform layer
- All platform components must support graceful shutdown
- Observability is mandatory for production debugging
