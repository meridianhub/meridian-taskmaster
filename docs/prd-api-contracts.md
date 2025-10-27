# API Contracts PRD (Protocol Buffers & gRPC)

**Tag**: `api-contracts`
**Folder**: `api/proto/`
**Dependencies**: `infra` (for build tooling)

## Overview

Define all BIAN service domain APIs using Protocol Buffers 3 and gRPC. These contracts are the source of truth for all service implementations and ensure type-safe communication.

## Scope

### In Scope
- Protobuf definitions for BIAN service domains:
  - FinancialAccounting (FinancialBookingLog, LedgerPosting)
  - PositionKeeping (FinancialPositionLog)
  - CurrentAccount
- Common types (Money, Currency, AccountType, etc.)
- gRPC service definitions with proper RPC methods
- Protobuf validation rules using buf or protoc-gen-validate
- Build pipeline integration for code generation
- OpenAPI/Swagger generation from protos

### Out of Scope
- Service implementation (handled by service domain tags)
- Database schema (handled by platform tag)
- Business logic (handled by domain tags)

## BIAN Service Domains to Model

### Financial Accounting
Based on: `../bian/bian-public-main/release13.0.0/semantic-apis/oas3/yamls/FinancialAccounting.yaml`

**Control Record**: FinancialBookingLog
- Financial account type (journal, sub-ledger)
- Product/service reference
- Business unit reference
- Chart of accounts booking rules
- Base currency
- Status

**Behavior Qualifier**: LedgerPosting
- Posting direction (debit/credit)
- Posting amount (credit-debit pair)
- Value date (effective date)
- Posting result

### Position Keeping
Based on: `../bian/bian-public-main/release13.0.0/semantic-apis/oas3/yamls/PositionKeeping.yaml`

**Control Record**: FinancialPositionLog
- Transaction log entries before reconciliation
- Transaction lineage and audit trail
- Status tracking

### Current Account
Based on: `../bian/bian-public-main/release13.0.0/semantic-apis/oas3/yamls/CurrentAccount.yaml`

**Control Record**: CurrentAccountFacility
- Account identification
- Account status (active, frozen, closed)
- Balance tracking
- Overdraft limits
- Transaction history

## Technical Requirements

### TR-1: Common Proto Definitions

File: `api/proto/common/v1/types.proto`

```protobuf
// Money representation with currency and decimal precision
message Money {
  string currency = 1;  // ISO-4217 code
  int64 amount = 2;     // Amount in smallest currency unit
  int32 decimal_places = 3;  // Decimal precision (e.g., 2 for USD)
}

// Account types from BIAN standard
enum AccountType {
  ACCOUNT_TYPE_UNSPECIFIED = 0;
  ACCOUNT_TYPE_DEBIT = 1;
  ACCOUNT_TYPE_CREDIT = 2;
  ACCOUNT_TYPE_VOSTRO = 3;
  ACCOUNT_TYPE_NOSTRO = 4;
  ACCOUNT_TYPE_CURRENT = 5;
  ACCOUNT_TYPE_SAVINGS = 6;
  // ... (see BIAN guide for full list)
}

// Posting direction
enum PostingDirection {
  POSTING_DIRECTION_UNSPECIFIED = 0;
  POSTING_DIRECTION_DEBIT = 1;
  POSTING_DIRECTION_CREDIT = 2;
}

// Standard timestamps
message Timestamp {
  int64 seconds = 1;
  int32 nanos = 2;
}
```

### TR-2: FinancialAccounting Proto

File: `api/proto/financial_accounting/v1/financial_accounting.proto`

Must include:
- `InitiateFinancialBookingLogRequest/Response`
- `UpdateFinancialBookingLogRequest/Response`
- `RetrieveFinancialBookingLogRequest/Response`
- `CaptureLedgerPostingRequest/Response`
- `RetrieveLedgerPostingRequest/Response`

Service definition:
```protobuf
service FinancialAccountingService {
  rpc InitiateFinancialBookingLog(InitiateFinancialBookingLogRequest) returns (InitiateFinancialBookingLogResponse);
  rpc UpdateFinancialBookingLog(UpdateFinancialBookingLogRequest) returns (UpdateFinancialBookingLogResponse);
  rpc RetrieveFinancialBookingLog(RetrieveFinancialBookingLogRequest) returns (RetrieveFinancialBookingLogResponse);
  rpc CaptureLedgerPosting(CaptureLedgerPostingRequest) returns (CaptureLedgerPostingResponse);
  rpc RetrieveLedgerPosting(RetrieveLedgerPostingRequest) returns (RetrieveLedgerPostingResponse);
}
```

### TR-3: PositionKeeping Proto

File: `api/proto/position_keeping/v1/position_keeping.proto`

Must include:
- Transaction log operations (Initiate, Update, Retrieve)
- Transaction status tracking
- Bulk transaction import support

### TR-4: CurrentAccount Proto

File: `api/proto/current_account/v1/current_account.proto`

Must include:
- Account initiation
- Debit/credit operations
- Balance retrieval
- Account status management

### TR-5: Validation Rules

Use `buf` or `protoc-gen-validate` for:
- Required field validation
- String format validation (ISO codes, UUIDs)
- Numeric range validation (amounts > 0)
- Enum validation (no UNSPECIFIED in requests)

Example:
```protobuf
import "validate/validate.proto";

message Money {
  string currency = 1 [(validate.rules).string = {
    len: 3,
    pattern: "^[A-Z]{3}$"
  }];
  int64 amount = 2 [(validate.rules).int64.gt = 0];
  int32 decimal_places = 3 [(validate.rules).int32 = {
    gte: 0,
    lte: 10
  }];
}
```

### TR-6: Build Pipeline Integration

- `buf.gen.yaml` configuration for code generation
- Generate Go code: `make proto` target
- Generate OpenAPI specs: `protoc-gen-openapiv2`
- Version proto packages (v1, v2, etc.)
- Proto breaking change detection in CI

### TR-7: Documentation

- Comments on all messages and fields (godoc-style)
- Service method documentation with examples
- README explaining proto structure and conventions
- Migration guide for proto changes

## Acceptance Criteria

- [ ] All BIAN operations mapped to gRPC methods
- [ ] Common types defined and reusable
- [ ] Validation rules prevent invalid requests
- [ ] Proto compilation succeeds with no errors
- [ ] Generated Go code is idiomatic
- [ ] OpenAPI specs generated successfully
- [ ] Documentation complete for all services
- [ ] Breaking change detection enabled in CI

## Success Metrics

- Proto compilation time: < 5 seconds
- Zero manual type conversions needed in service code
- 100% of BIAN required fields represented
- OpenAPI spec validates with no warnings

## Folder Structure

```
api/
└── proto/
    ├── common/
    │   └── v1/
    │       ├── types.proto
    │       └── errors.proto
    ├── financial_accounting/
    │   └── v1/
    │       └── financial_accounting.proto
    ├── position_keeping/
    │   └── v1/
    │       └── position_keeping.proto
    ├── current_account/
    │   └── v1/
    │       └── current_account.proto
    ├── buf.yaml
    ├── buf.gen.yaml
    └── README.md
```

## Reference

BIAN specifications located at: `../bian/bian-public-main/ledger-interfaces-guide.md`
