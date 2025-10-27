# Financial Accounting Service Domain PRD

**Tag**: `financial-accounting`
**Folder**: `internal/financial-accounting/`
**Dependencies**: `infra`, `api-contracts`, `platform`

## Overview

Implement the BIAN FinancialAccounting service domain - the core general ledger that maintains double-entry accounting records. This is the authoritative source of truth for all financial positions.

## BIAN Reference

Specification: `../bian/bian-public-main/release13.0.0/semantic-apis/oas3/yamls/FinancialAccounting.yaml`

**Purpose**: "Takes in financial facts and creates accounting instructions that will update the general ledger and sub ledger accounts"

## Scope

### In Scope
- FinancialBookingLog domain model (Control Record)
- LedgerPosting domain model (Behavior Qualifier)
- Double-entry accounting validation
- Multi-currency support with base currency conversion
- Chart of accounts booking rules
- Value dating for effective transaction timing
- Repository implementation with database persistence
- gRPC service implementation
- Domain events for transaction posted, booking log updated

### Out of Scope
- Reconciliation logic (handled by AccountReconciliation service domain)
- Account management (handled by CurrentAccount service domain)
- Event streaming infrastructure (handled by platform)
- Authentication/authorization (handled by platform)

## Domain Model

### FinancialBookingLog (Aggregate Root)

**Properties**:
- ID: UUID (primary key)
- Financial Account Type: Enum (journal, sub-ledger)
- Product and Service Reference: UUID
- Business Unit Reference: UUID
- Chart of Accounts Booking Rules Reference: UUID
- Base Currency: ISO-4217 currency code (3 chars)
- Status: Enum (active, suspended, closed)
- Created At: Timestamp
- Updated At: Timestamp

**Invariants**:
- Base currency must be valid ISO-4217 code
- Status transitions: active → suspended → active OR active → closed (terminal)
- Cannot add postings to non-active booking logs

**Methods**:
```go
type FinancialBookingLog struct {
    ID                      uuid.UUID
    FinancialAccountType    AccountType
    ProductServiceRef       uuid.UUID
    BusinessUnitRef         uuid.UUID
    ChartOfAccountsRulesRef uuid.UUID
    BaseCurrency            Currency
    Status                  BookingLogStatus
    CreatedAt               time.Time
    UpdatedAt               time.Time
}

func (f *FinancialBookingLog) Suspend() error
func (f *FinancialBookingLog) Resume() error
func (f *FinancialBookingLog) Close() error
func (f *FinancialBookingLog) CanAcceptPosting() bool
```

### LedgerPosting (Entity)

**Properties**:
- ID: UUID
- Financial Booking Log ID: UUID (foreign key)
- Posting Direction: Enum (debit, credit)
- Account Code: String (chart of accounts reference)
- Posting Amount: Money (amount, currency, decimal precision)
- Posting Value Date: Timestamp (effective date)
- Posting Result: Enum (success, failed, pending)
- Idempotency Key: String (unique)
- Created At: Timestamp

**Invariants**:
- Debit and credit must be paired (double-entry)
- Total debits must equal total credits in a transaction
- Posting amount must be > 0
- Currency must match booking log base currency OR have conversion rate
- Value date cannot be more than 1 year in future or past
- Idempotency key must be unique per booking log

**Methods**:
```go
type LedgerPosting struct {
    ID                  uuid.UUID
    BookingLogID        uuid.UUID
    PostingDirection    PostingDirection
    AccountCode         string
    PostingAmount       Money
    PostingValueDate    time.Time
    PostingResult       PostingResult
    IdempotencyKey      string
    CreatedAt           time.Time
}

func (l *LedgerPosting) Validate() error
```

### Double-Entry Transaction

A transaction consists of at least 2 postings (1 debit, 1 credit):

```go
type Transaction struct {
    Postings       []LedgerPosting
    IdempotencyKey string
}

func (t *Transaction) Validate() error {
    // 1. Must have at least 2 postings
    // 2. Must have at least 1 debit and 1 credit
    // 3. Sum of debits must equal sum of credits
    // 4. All postings must have same currency OR valid conversion rates
}
```

## Business Logic Requirements

### BL-1: Double-Entry Validation

**Rule**: For every transaction, total debits must equal total credits.

**Implementation**:
```go
func ValidateDoubleEntry(postings []LedgerPosting) error {
    var totalDebits, totalCredits decimal.Decimal

    for _, p := range postings {
        amount := decimal.NewFromInt(p.PostingAmount.Amount)
        if p.PostingDirection == PostingDirectionDebit {
            totalDebits = totalDebits.Add(amount)
        } else {
            totalCredits = totalCredits.Add(amount)
        }
    }

    if !totalDebits.Equal(totalCredits) {
        return ErrUnbalancedTransaction
    }
    return nil
}
```

### BL-2: Multi-Currency Conversion

**Rule**: If posting currency differs from booking log base currency, convert at transaction time.

**Implementation**:
- Accept exchange rates in posting request
- Store both original amount and converted amount
- Validate exchange rate is within acceptable bounds
- Use decimal arithmetic (never floats for money)

### BL-3: Value Dating

**Rule**: Postings have an effective date that may differ from creation date.

**Implementation**:
- Support backdating up to 1 year for corrections
- Support forward dating up to 1 year for scheduled transactions
- Value date determines when posting affects account balances
- Audit trail includes both value date and creation date

### BL-4: Chart of Accounts Booking Rules

**Rule**: Posting account codes must exist in chart of accounts.

**Implementation**:
- Validate account code against chart of accounts reference
- Check account type compatibility with posting direction
- Enforce account-specific validation rules (e.g., only certain accounts allow overdrafts)

### BL-5: Idempotency Guarantees

**Rule**: Duplicate posting requests with same idempotency key return original result.

**Implementation**:
- Check idempotency key before processing
- Store operation result after successful posting
- Return cached result for duplicate requests
- Expire idempotency records after configurable TTL (default 24 hours)

## Repository Interface

```go
type FinancialBookingLogRepository interface {
    Create(ctx context.Context, log *FinancialBookingLog) error
    FindByID(ctx context.Context, id uuid.UUID) (*FinancialBookingLog, error)
    Update(ctx context.Context, log *FinancialBookingLog) error
    List(ctx context.Context, filter BookingLogFilter) ([]*FinancialBookingLog, error)
}

type LedgerPostingRepository interface {
    CreatePostings(ctx context.Context, postings []*LedgerPosting) error
    FindByID(ctx context.Context, id uuid.UUID) (*LedgerPosting, error)
    FindByBookingLogID(ctx context.Context, bookingLogID uuid.UUID) ([]*LedgerPosting, error)
    FindByIdempotencyKey(ctx context.Context, key string) (*LedgerPosting, error)
}
```

## gRPC Service Implementation

Implement all operations from `api/proto/financial_accounting/v1/financial_accounting.proto`:

1. **InitiateFinancialBookingLog** - Create new booking log
2. **UpdateFinancialBookingLog** - Update booking log properties
3. **ControlFinancialBookingLog** - Suspend/resume/close log
4. **RetrieveFinancialBookingLog** - Get booking log details
5. **CaptureLedgerPosting** - Post transaction to ledger
6. **RetrieveLedgerPosting** - Get posting details

**Service structure**:
```go
type financialAccountingService struct {
    bookingLogRepo  FinancialBookingLogRepository
    postingRepo     LedgerPostingRepository
    eventPublisher  platform.EventPublisher
    idempotency     platform.IdempotencyChecker
}

func (s *financialAccountingService) CaptureLedgerPosting(
    ctx context.Context,
    req *pb.CaptureLedgerPostingRequest,
) (*pb.CaptureLedgerPostingResponse, error) {
    // 1. Check idempotency
    // 2. Validate double-entry
    // 3. Validate against chart of accounts
    // 4. Store postings in transaction
    // 5. Publish domain event
    // 6. Return result
}
```

## Domain Events

Publish events for downstream systems:

1. **FinancialBookingLogInitiated**
   - Booking log ID, type, base currency
2. **LedgerPostingCaptured**
   - Posting details, booking log ID, account codes, amounts
3. **FinancialBookingLogClosed**
   - Booking log ID, close timestamp

Event schema in `api/proto/financial_accounting/v1/events.proto`

## Database Schema

```sql
CREATE TABLE financial_booking_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    financial_account_type TEXT NOT NULL,
    product_service_ref UUID NOT NULL,
    business_unit_ref UUID NOT NULL,
    chart_of_accounts_rules_ref UUID NOT NULL,
    base_currency CHAR(3) NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('active', 'suspended', 'closed')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ledger_postings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_log_id UUID NOT NULL REFERENCES financial_booking_logs(id),
    posting_direction TEXT NOT NULL CHECK (posting_direction IN ('debit', 'credit')),
    account_code TEXT NOT NULL,
    amount_value BIGINT NOT NULL CHECK (amount_value > 0),
    amount_currency CHAR(3) NOT NULL,
    amount_decimal_places INT NOT NULL,
    posting_value_date TIMESTAMPTZ NOT NULL,
    posting_result TEXT NOT NULL CHECK (posting_result IN ('success', 'failed', 'pending')),
    idempotency_key TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(booking_log_id, idempotency_key)
);

CREATE INDEX idx_ledger_postings_booking_log_id ON ledger_postings(booking_log_id);
CREATE INDEX idx_ledger_postings_idempotency_key ON ledger_postings(idempotency_key);
CREATE INDEX idx_ledger_postings_value_date ON ledger_postings(posting_value_date);
```

## Acceptance Criteria

- [ ] FinancialBookingLog domain model implemented with all invariants
- [ ] LedgerPosting domain model implemented with validation
- [ ] Double-entry validation prevents unbalanced transactions
- [ ] Multi-currency conversion supported with exchange rates
- [ ] Value dating allows effective date specification
- [ ] Chart of accounts validation integrated
- [ ] Idempotency guarantees prevent duplicate postings
- [ ] Repository layer persists to database correctly
- [ ] gRPC service implements all BIAN operations
- [ ] Domain events published on state changes
- [ ] Integration tests cover all business rules
- [ ] Test coverage > 95% for domain logic

## Success Metrics

- Transaction posting latency: P99 < 50ms
- Double-entry validation: 100% accuracy (zero unbalanced transactions)
- Idempotency effectiveness: 100% (no duplicate postings)
- Test coverage: > 95%

## Folder Structure

```
internal/financial-accounting/
├── domain/
│   ├── financial_booking_log.go
│   ├── ledger_posting.go
│   ├── transaction.go
│   ├── validation.go
│   └── events.go
├── repository/
│   ├── booking_log_repository.go
│   ├── posting_repository.go
│   └── repository_test.go
└── service/
    ├── financial_accounting_service.go
    └── service_test.go
```

## Testing Strategy

- Unit tests for domain logic (double-entry, validation)
- Repository integration tests with testcontainers
- Service integration tests with full gRPC stack
- Property-based testing for double-entry invariants
- Chaos testing for concurrent posting scenarios
