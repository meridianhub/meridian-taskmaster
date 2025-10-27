# Position Keeping Service Domain PRD

**Tag**: `position-keeping`
**Folder**: `internal/position-keeping/`
**Dependencies**: `infra`, `api-contracts`, `platform`

## Overview

Implement the BIAN PositionKeeping service domain - the pre-ledger transaction log that captures financial transactions before reconciliation. This is the staging area before transactions are posted to the general ledger.

## BIAN Reference

Specification: `../bian/bian-public-main/release13.0.0/semantic-apis/oas3/yamls/PositionKeeping.yaml`

**Purpose**: "Maintains a financial transaction log to support production. Reconciled financial transactions are subsequently posted to the accounting systems"

## Scope

### In Scope
- FinancialPositionLog domain model (Control Record)
- Transaction capture and staging
- Transaction lineage and audit trail
- Transaction status tracking (pending, reconciled, posted, rejected)
- Transaction amendments before posting
- Bulk transaction import
- Repository implementation with database persistence
- gRPC service implementation
- Domain events for transaction lifecycle

### Out of Scope
- Reconciliation logic (handled by AccountReconciliation service domain)
- Ledger posting (handled by FinancialAccounting service domain)
- Event streaming infrastructure (handled by platform)
- Authentication/authorization (handled by platform)

## Data Flow

```
Transaction Source → PositionKeeping → AccountReconciliation → FinancialAccounting
(Payment systems)    (Pre-ledger log)   (Verification)        (General ledger)
```

## Domain Model

### FinancialPositionLog (Aggregate Root)

**Properties**:
- ID: UUID (primary key)
- Transaction Reference: String (external reference)
- Transaction Source: Enum (payment, card, transfer, manual)
- Transaction Type: String (debit, credit, reversal)
- Amount: Money (value, currency, decimal precision)
- Source Account: String (account identifier)
- Destination Account: String (account identifier)
- Transaction Date: Timestamp (when transaction occurred)
- Status: Enum (pending, reconciled, posted, rejected, amended)
- Reconciliation Status: Enum (not_reconciled, matched, discrepancy)
- Posted Ledger Reference: UUID (link to FinancialBookingLog after posting)
- Metadata: JSON (extensible transaction details)
- Idempotency Key: String (unique)
- Created At: Timestamp
- Updated At: Timestamp

**Invariants**:
- Transaction reference must be unique per source
- Amount must be > 0
- Status transitions: pending → reconciled → posted OR pending → rejected
- Cannot amend after status is posted or rejected
- Idempotency key must be unique

**Methods**:
```go
type FinancialPositionLog struct {
    ID                     uuid.UUID
    TransactionReference   string
    TransactionSource      TransactionSource
    TransactionType        string
    Amount                 Money
    SourceAccount          string
    DestinationAccount     string
    TransactionDate        time.Time
    Status                 PositionLogStatus
    ReconciliationStatus   ReconciliationStatus
    PostedLedgerReference  *uuid.UUID
    Metadata               map[string]interface{}
    IdempotencyKey         string
    CreatedAt              time.Time
    UpdatedAt              time.Time
}

func (f *FinancialPositionLog) MarkReconciled() error
func (f *FinancialPositionLog) MarkPosted(ledgerRef uuid.UUID) error
func (f *FinancialPositionLog) Reject(reason string) error
func (f *FinancialPositionLog) Amend(updates map[string]interface{}) error
func (f *FinancialPositionLog) CanBeAmended() bool
```

## Business Logic Requirements

### BL-1: Transaction Capture

**Rule**: Capture all financial transactions with complete audit trail.

**Implementation**:
- Accept transaction from source system
- Validate required fields (reference, amount, accounts)
- Check idempotency to prevent duplicates
- Store with status = pending
- Generate unique ID for internal tracking
- Preserve original transaction data in metadata

### BL-2: Transaction Status Lifecycle

**Valid transitions**:
```
pending → reconciled → posted (success path)
pending → rejected (failure path)
pending → amended → reconciled → posted (correction path)
```

**Implementation**:
- Enforce state machine with validation
- Track status change history
- Prevent invalid transitions
- Record user/system that triggered transition

### BL-3: Transaction Amendments

**Rule**: Allow amendments only for pending or reconciled transactions.

**Implementation**:
```go
func (f *FinancialPositionLog) Amend(updates map[string]interface{}) error {
    if f.Status != StatusPending && f.Status != StatusReconciled {
        return ErrCannotAmendTransaction
    }

    // Store original values in audit trail
    // Apply updates
    // Reset status to pending if was reconciled
    // Update timestamp

    return nil
}
```

### BL-4: Bulk Transaction Import

**Rule**: Support importing large batches of transactions efficiently.

**Implementation**:
- Accept batch of transactions (up to 10,000 per request)
- Validate each transaction
- Insert in database transaction for atomicity
- Return success/failure status per transaction
- Publish bulk import event

### BL-5: Transaction Lineage

**Rule**: Maintain complete audit trail from capture to posting.

**Implementation**:
- Link to source system transaction ID
- Link to reconciliation record
- Link to ledger posting reference
- Store all amendments with timestamps
- Never delete transactions (soft delete if needed)

### BL-6: Idempotency

**Rule**: Duplicate transaction captures return original result.

**Implementation**:
- Use idempotency key from source system
- Check before insertion
- Return existing transaction if duplicate
- Handle concurrent duplicate requests safely

## Repository Interface

```go
type FinancialPositionLogRepository interface {
    Create(ctx context.Context, log *FinancialPositionLog) error
    CreateBatch(ctx context.Context, logs []*FinancialPositionLog) error
    FindByID(ctx context.Context, id uuid.UUID) (*FinancialPositionLog, error)
    FindByIdempotencyKey(ctx context.Context, key string) (*FinancialPositionLog, error)
    FindByTransactionReference(ctx context.Context, ref string) (*FinancialPositionLog, error)
    Update(ctx context.Context, log *FinancialPositionLog) error
    List(ctx context.Context, filter PositionLogFilter) ([]*FinancialPositionLog, error)
    FindPendingForReconciliation(ctx context.Context, limit int) ([]*FinancialPositionLog, error)
}
```

## gRPC Service Implementation

Implement all operations from `api/proto/position_keeping/v1/position_keeping.proto`:

1. **InitiateFinancialPositionLog** - Capture transaction
2. **InitiateFinancialPositionLogBatch** - Capture bulk transactions
3. **UpdateFinancialPositionLog** - Amend transaction
4. **ControlFinancialPositionLog** - Change status (reconcile, post, reject)
5. **RetrieveFinancialPositionLog** - Get transaction details
6. **RetrieveFinancialPositionLogBatch** - Get multiple transactions

**Service structure**:
```go
type positionKeepingService struct {
    positionLogRepo FinancialPositionLogRepository
    eventPublisher  platform.EventPublisher
    idempotency     platform.IdempotencyChecker
}

func (s *positionKeepingService) InitiateFinancialPositionLog(
    ctx context.Context,
    req *pb.InitiateFinancialPositionLogRequest,
) (*pb.InitiateFinancialPositionLogResponse, error) {
    // 1. Check idempotency
    // 2. Validate transaction fields
    // 3. Create position log with status = pending
    // 4. Store in database
    // 5. Publish TransactionCaptured event
    // 6. Return result
}

func (s *positionKeepingService) InitiateFinancialPositionLogBatch(
    ctx context.Context,
    req *pb.InitiateFinancialPositionLogBatchRequest,
) (*pb.InitiateFinancialPositionLogBatchResponse, error) {
    // 1. Validate batch (max 10,000 transactions)
    // 2. Check idempotency for each transaction
    // 3. Insert batch in database transaction
    // 4. Publish BulkTransactionCaptured event
    // 5. Return per-transaction results
}

func (s *positionKeepingService) ControlFinancialPositionLog(
    ctx context.Context,
    req *pb.ControlFinancialPositionLogRequest,
) (*pb.ControlFinancialPositionLogResponse, error) {
    // 1. Load position log
    // 2. Apply status transition (reconciled, posted, rejected)
    // 3. Update in database
    // 4. Publish status change event
    // 5. Return result
}
```

## Domain Events

Publish events for downstream systems:

1. **TransactionCaptured**
   - Position log ID, transaction reference, amount, accounts
2. **TransactionAmended**
   - Position log ID, changed fields, amendment reason
3. **TransactionReconciled**
   - Position log ID, reconciliation status
4. **TransactionPosted**
   - Position log ID, ledger reference
5. **TransactionRejected**
   - Position log ID, rejection reason
6. **BulkTransactionCaptured**
   - Count, source, batch ID

Event schema in `api/proto/position_keeping/v1/events.proto`

## Database Schema

```sql
CREATE TABLE financial_position_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_reference TEXT NOT NULL,
    transaction_source TEXT NOT NULL,
    transaction_type TEXT NOT NULL,
    amount_value BIGINT NOT NULL CHECK (amount_value > 0),
    amount_currency CHAR(3) NOT NULL,
    amount_decimal_places INT NOT NULL,
    source_account TEXT NOT NULL,
    destination_account TEXT NOT NULL,
    transaction_date TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('pending', 'reconciled', 'posted', 'rejected', 'amended')),
    reconciliation_status TEXT NOT NULL CHECK (reconciliation_status IN ('not_reconciled', 'matched', 'discrepancy')),
    posted_ledger_reference UUID,
    metadata JSONB,
    idempotency_key TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(transaction_source, transaction_reference)
);

CREATE INDEX idx_financial_position_logs_status ON financial_position_logs(status);
CREATE INDEX idx_financial_position_logs_transaction_ref ON financial_position_logs(transaction_reference);
CREATE INDEX idx_financial_position_logs_idempotency_key ON financial_position_logs(idempotency_key);
CREATE INDEX idx_financial_position_logs_transaction_date ON financial_position_logs(transaction_date);
CREATE INDEX idx_financial_position_logs_reconciliation_status ON financial_position_logs(reconciliation_status);

-- Audit trail table
CREATE TABLE financial_position_log_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_log_id UUID NOT NULL REFERENCES financial_position_logs(id),
    change_type TEXT NOT NULL,
    old_values JSONB,
    new_values JSONB,
    changed_by TEXT,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_financial_position_log_audit_position_log_id ON financial_position_log_audit(position_log_id);
```

## Acceptance Criteria

- [ ] FinancialPositionLog domain model implemented with all invariants
- [ ] Transaction capture validates required fields
- [ ] Status lifecycle enforces valid transitions
- [ ] Transaction amendments update audit trail
- [ ] Bulk import handles 10,000+ transactions efficiently
- [ ] Idempotency prevents duplicate transaction capture
- [ ] Transaction lineage maintained from capture to posting
- [ ] Repository layer persists to database correctly
- [ ] gRPC service implements all BIAN operations
- [ ] Domain events published on state changes
- [ ] Integration tests cover all business rules
- [ ] Test coverage > 95% for domain logic

## Success Metrics

- Transaction capture latency: P99 < 20ms
- Bulk import throughput: > 10,000 transactions/second
- Idempotency effectiveness: 100% (no duplicate captures)
- Status transition accuracy: 100% (no invalid transitions)
- Test coverage: > 95%

## Folder Structure

```
internal/position-keeping/
├── domain/
│   ├── financial_position_log.go
│   ├── status_transitions.go
│   ├── validation.go
│   └── events.go
├── repository/
│   ├── position_log_repository.go
│   └── repository_test.go
└── service/
    ├── position_keeping_service.go
    └── service_test.go
```

## Testing Strategy

- Unit tests for domain logic (status transitions, amendments)
- Repository integration tests with testcontainers
- Service integration tests with full gRPC stack
- Bulk import performance tests
- Idempotency concurrency tests
- Status transition state machine verification
