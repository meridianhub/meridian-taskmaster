# Current Account Service Domain PRD

**Tag**: `current-account`
**Folder**: `internal/current-account/`
**Dependencies**: `infra`, `api-contracts`, `platform`, `financial-accounting`, `position-keeping`

## Overview

Implement the BIAN CurrentAccount service domain - customer-facing current accounts that support deposits, withdrawals, transfers, and balance inquiries. This service domain integrates with PositionKeeping (for transaction capture) and FinancialAccounting (for ledger posting).

## BIAN Reference

Specification: `../bian/bian-public-main/release13.0.0/semantic-apis/oas3/yamls/CurrentAccount.yaml`

**Purpose**: "Fulfill a range of current account fulfillment routines and product and service features"

## Scope

### In Scope
- CurrentAccountFacility domain model (Control Record)
- Account initiation with customer identification
- Debit and credit operations with balance validation
- Overdraft limit enforcement
- Account status management (active, frozen, closed)
- Real-time balance calculation
- Transaction history per account
- Integration with PositionKeeping for transaction logging
- Integration with FinancialAccounting for ledger posting
- Repository implementation with database persistence
- gRPC service implementation
- Domain events for account lifecycle

### Out of Scope
- Customer identity management (external IAM integration)
- Interest calculation (future enhancement)
- Statement generation (future enhancement)
- Cheque processing (future enhancement)
- Authentication/authorization (handled by platform)

## Domain Model

### CurrentAccountFacility (Aggregate Root)

**Properties**:
- ID: UUID (primary key)
- Account Number: String (unique customer-facing identifier)
- Customer Reference: UUID (external customer ID)
- Account Currency: ISO-4217 currency code
- Account Status: Enum (active, frozen, closed)
- Current Balance: Money (real-time balance)
- Available Balance: Money (balance minus holds/reserves)
- Overdraft Limit: Money (maximum negative balance allowed)
- Account Opening Date: Date
- Account Closed Date: Date (nullable)
- Product Reference: UUID (account product configuration)
- Branch Reference: UUID (owning branch)
- Created At: Timestamp
- Updated At: Timestamp

**Invariants**:
- Account number must be unique and immutable
- Balance cannot go below negative overdraft limit
- Cannot perform transactions on frozen or closed accounts
- Overdraft limit must be >= 0
- Available balance = current balance - holds - reserved amounts
- Status transitions: active → frozen → active OR active → closed (terminal)

**Methods**:
```go
type CurrentAccountFacility struct {
    ID                 uuid.UUID
    AccountNumber      string
    CustomerReference  uuid.UUID
    AccountCurrency    Currency
    AccountStatus      AccountStatus
    CurrentBalance     Money
    AvailableBalance   Money
    OverdraftLimit     Money
    AccountOpeningDate time.Time
    AccountClosedDate  *time.Time
    ProductReference   uuid.UUID
    BranchReference    uuid.UUID
    CreatedAt          time.Time
    UpdatedAt          time.Time
}

func (c *CurrentAccountFacility) Debit(amount Money, description string) error
func (c *CurrentAccountFacility) Credit(amount Money, description string) error
func (c *CurrentAccountFacility) CanDebit(amount Money) bool
func (c *CurrentAccountFacility) Freeze() error
func (c *CurrentAccountFacility) Unfreeze() error
func (c *CurrentAccountFacility) Close() error
func (c *CurrentAccountFacility) IsActive() bool
```

### AccountTransaction (Entity)

**Properties**:
- ID: UUID
- Account ID: UUID (foreign key)
- Transaction Type: Enum (debit, credit, reversal)
- Amount: Money
- Running Balance: Money (balance after transaction)
- Description: String
- Transaction Date: Timestamp
- Value Date: Timestamp (effective date)
- Position Log Reference: UUID (link to PositionKeeping)
- Ledger Posting Reference: UUID (link to FinancialAccounting)
- Status: Enum (pending, completed, failed, reversed)
- Idempotency Key: String (unique)
- Created At: Timestamp

**Methods**:
```go
type AccountTransaction struct {
    ID                      uuid.UUID
    AccountID               uuid.UUID
    TransactionType         TransactionType
    Amount                  Money
    RunningBalance          Money
    Description             string
    TransactionDate         time.Time
    ValueDate               time.Time
    PositionLogReference    *uuid.UUID
    LedgerPostingReference  *uuid.UUID
    Status                  TransactionStatus
    IdempotencyKey          string
    CreatedAt               time.Time
}

func (t *AccountTransaction) MarkCompleted(ledgerRef uuid.UUID) error
func (t *AccountTransaction) MarkFailed(reason string) error
func (t *AccountTransaction) Reverse() (*AccountTransaction, error)
```

## Business Logic Requirements

### BL-1: Account Initiation

**Rule**: New accounts start with zero balance and active status.

**Implementation**:
```go
func InitiateAccount(
    customerRef uuid.UUID,
    currency Currency,
    overdraftLimit Money,
    productRef uuid.UUID,
) (*CurrentAccountFacility, error) {
    // 1. Generate unique account number
    // 2. Validate customer exists (external check)
    // 3. Validate currency is supported
    // 4. Create account with status = active, balance = 0
    // 5. Create corresponding FinancialBookingLog
    // 6. Publish AccountInitiated event
}
```

### BL-2: Debit Operation

**Rule**: Debits reduce balance and must not exceed available balance + overdraft.

**Implementation**:
```go
func (c *CurrentAccountFacility) Debit(amount Money, description string) error {
    // 1. Check account is active
    if !c.IsActive() {
        return ErrAccountNotActive
    }

    // 2. Check sufficient funds
    if !c.CanDebit(amount) {
        return ErrInsufficientFunds
    }

    // 3. Reduce balance
    c.CurrentBalance = c.CurrentBalance.Subtract(amount)
    c.AvailableBalance = c.AvailableBalance.Subtract(amount)

    // 4. Create transaction record
    // 5. Create PositionKeeping entry
    // 6. After reconciliation, create LedgerPosting

    return nil
}

func (c *CurrentAccountFacility) CanDebit(amount Money) bool {
    minAllowedBalance := c.OverdraftLimit.Negate()
    return c.AvailableBalance.Subtract(amount).GreaterThanOrEqual(minAllowedBalance)
}
```

### BL-3: Credit Operation

**Rule**: Credits increase balance with no limit.

**Implementation**:
```go
func (c *CurrentAccountFacility) Credit(amount Money, description string) error {
    // 1. Check account is active
    if !c.IsActive() {
        return ErrAccountNotActive
    }

    // 2. Increase balance
    c.CurrentBalance = c.CurrentBalance.Add(amount)
    c.AvailableBalance = c.AvailableBalance.Add(amount)

    // 3. Create transaction record
    // 4. Create PositionKeeping entry
    // 5. After reconciliation, create LedgerPosting

    return nil
}
```

### BL-4: Overdraft Enforcement

**Rule**: Balance cannot go below negative overdraft limit.

**Implementation**:
- Check before every debit
- Reject transaction if would exceed overdraft
- Overdraft limit is per-account configuration
- Zero overdraft = no negative balance allowed

### BL-5: Account Status Management

**Valid transitions**:
```
active → frozen (temporary suspension)
frozen → active (restore operations)
active → closed (terminal state, no transactions allowed)
```

**Implementation**:
```go
func (c *CurrentAccountFacility) Freeze() error {
    if c.AccountStatus != AccountStatusActive {
        return ErrInvalidStatusTransition
    }
    c.AccountStatus = AccountStatusFrozen
    // Publish AccountFrozen event
    return nil
}

func (c *CurrentAccountFacility) Close() error {
    if c.AccountStatus != AccountStatusActive {
        return ErrInvalidStatusTransition
    }
    // Check balance is zero (or handle closure with balance)
    c.AccountStatus = AccountStatusClosed
    c.AccountClosedDate = &now
    // Publish AccountClosed event
    return nil
}
```

### BL-6: Transaction Integration

**Flow**: CurrentAccount → PositionKeeping → (Reconciliation) → FinancialAccounting

**Implementation**:
1. **Capture transaction in CurrentAccount** (update balance)
2. **Create PositionKeeping entry** (transaction log)
3. **Reconciliation process verifies** (external system)
4. **Create LedgerPosting** (double-entry accounting)
5. **Update transaction status** (link to ledger reference)

### BL-7: Real-Time Balance

**Rule**: Balance is calculated in real-time, not derived from transactions.

**Implementation**:
- Store running balance in account record
- Update atomically with each transaction
- Transaction history provides audit trail
- Balance can be recalculated from transaction history for verification

### BL-8: Idempotency

**Rule**: Duplicate transaction requests return original result.

**Implementation**:
- Accept idempotency key on all mutation operations
- Check before processing transaction
- Return existing transaction if duplicate
- Handle concurrent duplicate requests safely

## Repository Interface

```go
type CurrentAccountRepository interface {
    Create(ctx context.Context, account *CurrentAccountFacility) error
    FindByID(ctx context.Context, id uuid.UUID) (*CurrentAccountFacility, error)
    FindByAccountNumber(ctx context.Context, accountNumber string) (*CurrentAccountFacility, error)
    FindByCustomerReference(ctx context.Context, customerRef uuid.UUID) ([]*CurrentAccountFacility, error)
    Update(ctx context.Context, account *CurrentAccountFacility) error
    List(ctx context.Context, filter AccountFilter) ([]*CurrentAccountFacility, error)
}

type AccountTransactionRepository interface {
    Create(ctx context.Context, txn *AccountTransaction) error
    FindByID(ctx context.Context, id uuid.UUID) (*AccountTransaction, error)
    FindByAccountID(ctx context.Context, accountID uuid.UUID, limit int) ([]*AccountTransaction, error)
    FindByIdempotencyKey(ctx context.Context, key string) (*AccountTransaction, error)
    Update(ctx context.Context, txn *AccountTransaction) error
}
```

## gRPC Service Implementation

Implement all operations from `api/proto/current_account/v1/current_account.proto`:

1. **InitiateCurrentAccount** - Open new account
2. **UpdateCurrentAccount** - Modify account properties (overdraft, status)
3. **ControlCurrentAccount** - Freeze/unfreeze/close account
4. **RetrieveCurrentAccount** - Get account details
5. **DebitCurrentAccount** - Withdraw funds
6. **CreditCurrentAccount** - Deposit funds
7. **RetrieveAccountTransactions** - Get transaction history

**Service structure**:
```go
type currentAccountService struct {
    accountRepo            CurrentAccountRepository
    transactionRepo        AccountTransactionRepository
    positionKeepingClient  pb.PositionKeepingServiceClient
    financialAccountingClient pb.FinancialAccountingServiceClient
    eventPublisher         platform.EventPublisher
    idempotency            platform.IdempotencyChecker
}

func (s *currentAccountService) DebitCurrentAccount(
    ctx context.Context,
    req *pb.DebitCurrentAccountRequest,
) (*pb.DebitCurrentAccountResponse, error) {
    // 1. Check idempotency
    // 2. Load account
    // 3. Validate account is active
    // 4. Check sufficient funds (including overdraft)
    // 5. Update balance atomically
    // 6. Create transaction record
    // 7. Call PositionKeeping to log transaction
    // 8. Publish AccountDebited event
    // 9. Return result
}
```

## Domain Events

Publish events for downstream systems:

1. **CurrentAccountInitiated**
   - Account ID, account number, customer reference
2. **AccountDebited**
   - Account ID, amount, running balance
3. **AccountCredited**
   - Account ID, amount, running balance
4. **AccountFrozen**
   - Account ID, freeze timestamp
5. **AccountUnfrozen**
   - Account ID, unfreeze timestamp
6. **AccountClosed**
   - Account ID, close timestamp, final balance
7. **OverdraftLimitExceeded**
   - Account ID, attempted amount (alert event)

Event schema in `api/proto/current_account/v1/events.proto`

## Database Schema

```sql
CREATE TABLE current_account_facilities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number TEXT NOT NULL UNIQUE,
    customer_reference UUID NOT NULL,
    account_currency CHAR(3) NOT NULL,
    account_status TEXT NOT NULL CHECK (account_status IN ('active', 'frozen', 'closed')),
    current_balance_value BIGINT NOT NULL,
    current_balance_currency CHAR(3) NOT NULL,
    current_balance_decimal_places INT NOT NULL,
    available_balance_value BIGINT NOT NULL,
    available_balance_currency CHAR(3) NOT NULL,
    available_balance_decimal_places INT NOT NULL,
    overdraft_limit_value BIGINT NOT NULL CHECK (overdraft_limit_value >= 0),
    overdraft_limit_currency CHAR(3) NOT NULL,
    overdraft_limit_decimal_places INT NOT NULL,
    account_opening_date DATE NOT NULL,
    account_closed_date DATE,
    product_reference UUID NOT NULL,
    branch_reference UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_current_account_facilities_account_number ON current_account_facilities(account_number);
CREATE INDEX idx_current_account_facilities_customer_ref ON current_account_facilities(customer_reference);
CREATE INDEX idx_current_account_facilities_status ON current_account_facilities(account_status);

CREATE TABLE account_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES current_account_facilities(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('debit', 'credit', 'reversal')),
    amount_value BIGINT NOT NULL CHECK (amount_value > 0),
    amount_currency CHAR(3) NOT NULL,
    amount_decimal_places INT NOT NULL,
    running_balance_value BIGINT NOT NULL,
    running_balance_currency CHAR(3) NOT NULL,
    running_balance_decimal_places INT NOT NULL,
    description TEXT NOT NULL,
    transaction_date TIMESTAMPTZ NOT NULL,
    value_date TIMESTAMPTZ NOT NULL,
    position_log_reference UUID,
    ledger_posting_reference UUID,
    status TEXT NOT NULL CHECK (status IN ('pending', 'completed', 'failed', 'reversed')),
    idempotency_key TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_account_transactions_account_id ON account_transactions(account_id);
CREATE INDEX idx_account_transactions_idempotency_key ON account_transactions(idempotency_key);
CREATE INDEX idx_account_transactions_transaction_date ON account_transactions(transaction_date);
```

## Acceptance Criteria

- [ ] CurrentAccountFacility domain model implemented with all invariants
- [ ] Account initiation creates account with zero balance
- [ ] Debit operation validates sufficient funds and overdraft
- [ ] Credit operation increases balance correctly
- [ ] Overdraft limit enforcement prevents excessive debits
- [ ] Account status transitions enforce valid states
- [ ] Real-time balance updates atomically with transactions
- [ ] Transaction history maintained per account
- [ ] Integration with PositionKeeping captures transactions
- [ ] Integration with FinancialAccounting posts to ledger
- [ ] Idempotency prevents duplicate transactions
- [ ] Repository layer persists to database correctly
- [ ] gRPC service implements all BIAN operations
- [ ] Domain events published on state changes
- [ ] Integration tests cover all business rules
- [ ] Test coverage > 95% for domain logic

## Success Metrics

- Account transaction latency: P99 < 30ms
- Balance calculation accuracy: 100% (zero discrepancies)
- Overdraft enforcement: 100% (no violations)
- Idempotency effectiveness: 100% (no duplicate transactions)
- Concurrent transaction handling: > 1,000 TPS per account
- Test coverage: > 95%

## Folder Structure

```
internal/current-account/
├── domain/
│   ├── current_account_facility.go
│   ├── account_transaction.go
│   ├── validation.go
│   └── events.go
├── repository/
│   ├── account_repository.go
│   ├── transaction_repository.go
│   └── repository_test.go
└── service/
    ├── current_account_service.go
    └── service_test.go
```

## Testing Strategy

- Unit tests for domain logic (balance, overdraft, status transitions)
- Repository integration tests with testcontainers
- Service integration tests with full gRPC stack
- Concurrent transaction tests (race conditions, atomicity)
- Overdraft limit enforcement tests
- Integration tests with PositionKeeping and FinancialAccounting
- Property-based testing for balance invariants
