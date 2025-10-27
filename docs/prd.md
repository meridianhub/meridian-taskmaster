# Meridian PRD: Production-Grade Open Banking Ledger

## Project Overview

**Project Name:** Meridian

**Name Rationale:**
- **Prime Meridian** represents the zero point of longitude - the reference line from which all measurements are taken. Similarly, Meridian serves as the authoritative source of truth for financial positions
- **Meridian** suggests precision, standardization, and global applicability - core attributes for a banking ledger
- **Navigation metaphor** reflects guiding financial institutions through the complexity of modern banking
- The name is memorable, pronounceable across languages, and available for open source branding

**Vision Statement:**
Meridian is a production-grade, open-source financial ledger implementing BIAN (Banking Industry Architecture Network) standards. It provides a thin, high-performance transaction processing layer for modern coreless banking architectures.

**Core Principles:**
1. **BIAN Compliance First** - Full implementation of BIAN semantic APIs
2. **Production Ready** - No proof-of-concepts; every feature is production-grade
3. **Test-Driven** - Red-green-refactor cycle mandated for all code
4. **Type Safety** - Leverage type systems to prevent financial errors
5. **Observable** - Full instrumentation for production operations
6. **Cloud Native** - Kubernetes-native, multi-region capable
7. **Open Source** - Apache 2.0 licensed, community-driven

---

## Success Criteria

### Business Metrics
- **Throughput:** Process 100,000+ transactions per second sustained
- **Latency:** P99 latency < 100ms for transaction posting
- **Availability:** 99.999% uptime (5 minutes downtime per year)
- **Accuracy:** Zero tolerance for financial calculation errors
- **Adoption:** 10+ production deployments within 18 months of v1.0

### Technical Metrics
- **Test Coverage:** >95% for all financial logic, >85% overall
- **Deployment Time:** < 5 minutes for zero-downtime rolling update
- **Recovery Time:** < 60 seconds for automatic failover
- **Data Consistency:** All transactions atomic and durable
- **Standards Compliance:** Pass BIAN conformance test suite

---

## User Personas

### P1: Platform Engineer
**Needs:**
- Deploy and operate Meridian in production
- Monitor health and performance
- Respond to incidents effectively
- Scale capacity to meet demand

### P2: Integration Developer
**Needs:**
- Integrate banking products with Meridian APIs
- Understand BIAN service domains
- Test integrations before production
- Debug integration issues

### P3: Financial Controller
**Needs:**
- Trust ledger accuracy
- Reconcile positions daily
- Generate regulatory reports
- Audit transaction history

### P4: Product Manager
**Needs:**
- Launch new banking products quickly
- Understand feature roadmap
- Assess BIAN compliance
- Evaluate total cost of ownership

---

## Functional Requirements

### FR-1: Financial Accounting (General Ledger)

**As a** financial system
**I need to** maintain double-entry accounting records
**So that** all transactions are accurately recorded and balanced

**Acceptance Criteria:**
- Implement BIAN FinancialAccounting service domain
- Support FinancialBookingLog control records
- Execute LedgerPosting with double-entry validation
- Maintain chart of accounts with booking rules
- Support multi-currency with base currency conversion
- Validate all debits equal credits before posting
- Store decimal precision metadata for all amounts
- Support value dating for effective transaction timing

### FR-2: Position Keeping (Transaction Log)

**As a** transaction processing system
**I need to** capture financial transactions before reconciliation
**So that** transaction integrity is maintained through the reconciliation process

**Acceptance Criteria:**
- Implement BIAN PositionKeeping service domain
- Capture FinancialPositionLog entries
- Support transaction staging pre-reconciliation
- Maintain transaction lineage and audit trail
- Support transaction amendments before posting
- Provide transaction status tracking
- Enable bulk transaction import

### FR-3: Account Reconciliation

**As a** financial operations team
**I need to** reconcile transactions before posting to the ledger
**So that** only verified transactions affect account balances

**Acceptance Criteria:**
- Implement BIAN AccountReconciliation service domain
- Support manual and automated reconciliation workflows
- Flag discrepancies for review
- Track reconciliation status per transaction
- Support batch reconciliation processing
- Generate reconciliation reports
- Maintain reconciliation audit trail

### FR-4: Current Account Management

**As a** banking platform
**I need to** manage customer current accounts
**So that** customers can make deposits, withdrawals, and transfers

**Acceptance Criteria:**
- Implement BIAN CurrentAccount service domain
- Support account initiation with customer identification
- Process debits and credits with balance validation
- Enforce overdraft limits
- Support account status management (active, frozen, closed)
- Maintain transaction history per account
- Calculate available balance in real-time

### FR-5: Idempotency Guarantees

**As a** payment system
**I need to** ensure duplicate transaction requests are handled safely
**So that** retry logic doesn't cause duplicate postings

**Acceptance Criteria:**
- Accept idempotency keys on all mutation operations
- Return identical responses for duplicate requests
- Support configurable idempotency window
- Handle concurrent duplicate requests safely
- Expire idempotency records after configurable TTL
- Provide idempotency status via API

### FR-6: Transaction Event Streaming

**As a** downstream system
**I need to** receive real-time transaction events
**So that** I can update dependent systems immediately

**Acceptance Criteria:**
- Publish all transaction events to event stream
- Support at-least-once delivery semantics
- Include complete transaction context in events
- Maintain event ordering per account
- Support event replay for recovery scenarios
- Provide dead letter queue for failed events

### FR-7: Multi-Currency Support

**As a** international bank
**I need to** handle transactions in multiple currencies
**So that** customers can transact globally

**Acceptance Criteria:**
- Support all ISO-4217 currency codes
- Maintain base currency per account
- Apply exchange rates to cross-currency transactions
- Store original currency and converted amounts
- Support currency-specific decimal precision
- Validate currency compatibility on transfers

### FR-8: Audit Trail

**As a** compliance officer
**I need to** review complete transaction history
**So that** I can demonstrate regulatory compliance

**Acceptance Criteria:**
- Record immutable audit log for all operations
- Include user identity and timestamp on all events
- Capture before/after state for mutations
- Support audit log querying and export
- Retain audit logs per regulatory requirements
- Provide tamper-evident audit log storage

### FR-9: API Authentication & Authorization

**As a** security officer
**I need to** control access to ledger operations
**So that** only authorized systems can modify accounts

**Acceptance Criteria:**
- Support OAuth 2.0 / OpenID Connect
- Implement role-based access control
- Enforce least-privilege per service domain
- Support API key authentication for service accounts
- Log all authentication attempts
- Provide token refresh without interruption

### FR-10: Health & Readiness Probes

**As a** Kubernetes orchestrator
**I need to** monitor service health
**So that** I can route traffic only to healthy instances

**Acceptance Criteria:**
- Expose `/health/live` endpoint for liveness
- Expose `/health/ready` endpoint for readiness
- Check database connectivity in readiness probe
- Check event stream connectivity in readiness probe
- Return appropriate HTTP status codes
- Include dependency status in health response

---

## Non-Functional Requirements

### NFR-1: Performance

- **Throughput:** 100,000+ transactions per second sustained under load
- **Latency:** P50 < 10ms, P99 < 100ms, P99.9 < 500ms for transaction posting
- **Database:** CockroachDB or YugabyteDB for distributed SQL
- **Caching:** Redis for idempotency and hot data
- **Event Streaming:** Kafka for transaction events

### NFR-2: Reliability

- **Availability:** 99.999% uptime SLA
- **Data Durability:** Zero data loss under any failure scenario
- **Failover:** Automatic failover within 60 seconds
- **Disaster Recovery:** RPO < 1 second, RTO < 5 minutes
- **Graceful Degradation:** Continue operating with degraded dependencies

### NFR-3: Scalability

- **Horizontal Scaling:** Support 10,000+ concurrent connections per instance
- **Vertical Scaling:** Utilize all available CPU and memory
- **Database Scaling:** Support multi-region active-active deployments
- **Event Streaming:** Support millions of events per second
- **Storage:** Petabyte-scale transaction history

### NFR-4: Security

- **Encryption at Rest:** All data encrypted using AES-256
- **Encryption in Transit:** TLS 1.3 for all network communication
- **Secret Management:** Integration with Vault or equivalent
- **Audit Logging:** Immutable audit trail for all operations
- **Compliance:** SOC 2 Type II, ISO 27001, PCI-DSS ready

### NFR-5: Observability

- **Metrics:** Prometheus-compatible metrics endpoint
- **Tracing:** OpenTelemetry distributed tracing
- **Logging:** Structured JSON logs with correlation IDs
- **Dashboards:** Pre-built Grafana dashboards
- **Alerting:** Runbook-driven alerting for all critical scenarios

### NFR-6: Testability

- **Unit Tests:** >95% coverage for domain logic
- **Integration Tests:** Full test suite against real dependencies
- **Contract Tests:** BIAN API contract validation
- **Load Tests:** Automated performance regression testing
- **Chaos Tests:** Regular chaos engineering exercises

### NFR-7: Operability

- **Deployment:** Zero-downtime rolling deployments
- **Configuration:** 12-factor app principles, environment-based config
- **Rollback:** Automatic rollback on health check failure
- **Debugging:** Live debugging via admin endpoints (auth required)
- **Documentation:** OpenAPI specs, runbooks, architecture diagrams

---

## Technical Stack Requirements

**Language:** Go 1.23+
**API Protocol:** Protocol Buffers 3 (Protobuf) + gRPC with REST gateway
**Event Streaming:** Apache Kafka 3.x
**Distributed Database:** CockroachDB 24.x or YugabyteDB 2024.x
**Cache:** Redis 7.x
**Orchestration:** Kubernetes 1.28+
**Development Tooling:** Tilt for local development
**Observability:** OpenTelemetry + Prometheus + Grafana
**Testing:** Go testing stdlib + testify + testcontainers

---

## BIAN Service Domain Implementation Priority

### Phase 1: Core Ledger (MVP)
1. FinancialAccounting (General Ledger)
2. PositionKeeping (Transaction Log)
3. CurrentAccount (Customer Accounts)

### Phase 2: Reconciliation & Integrity
4. AccountReconciliation
5. FinancialStatementAssessment (Basic Reporting)

### Phase 3: Additional Account Types
6. SavingsAccount
7. InternalBankAccount

### Phase 4: Advanced Features
8. PositionManagement (Consolidated Positions)
9. RegulatoryReporting
10. ComplianceReporting

---

## Quality Gates

Every pull request must pass:

### Automated Checks
- ✅ All tests pass (unit, integration, contract)
- ✅ Test coverage meets threshold (95% domain, 85% overall)
- ✅ No linter warnings (`golangci-lint`)
- ✅ Security scan passes (`gosec`, `trivy`)
- ✅ Protobuf compilation succeeds
- ✅ OpenAPI spec validation passes
- ✅ Performance benchmarks within tolerance

### Manual Review
- ✅ Two approvals from maintainers
- ✅ Architecture review for significant changes
- ✅ BIAN compliance validation
- ✅ Documentation updated

### Pre-Release
- ✅ Load test achieves SLA targets
- ✅ Chaos test demonstrates resilience
- ✅ Security audit completed
- ✅ Migration path tested

---

## Out of Scope (Explicitly Not Included)

- **Banking Products:** Loans, mortgages, credit cards (use Meridian as foundation)
- **Customer Identity Management:** Integrate with external IAM
- **Payment Networks:** SWIFT, ACH, card networks (separate integration)
- **Risk Management:** Credit scoring, fraud detection (separate services)
- **Customer-Facing UI:** APIs only; UIs are consuming applications
- **AI/ML Features:** No autonomous agents in v1.0

---

## Success Metrics Dashboard

### Operational Health
- **Uptime %** (target: 99.999%)
- **Error Rate** (target: < 0.01%)
- **P99 Latency** (target: < 100ms)
- **Throughput TPS** (target: > 100k)

### Financial Accuracy
- **Reconciliation Pass Rate** (target: 100%)
- **Balance Discrepancies** (target: 0)
- **Failed Transactions** (target: < 0.01%)

### Engineering Velocity
- **Deployment Frequency** (target: daily)
- **Lead Time for Changes** (target: < 1 day)
- **Mean Time to Recovery** (target: < 1 hour)
- **Change Failure Rate** (target: < 5%)

### Community Health
- **Active Contributors** (target: 50+)
- **Monthly Downloads** (target: 1000+)
- **GitHub Stars** (target: 5000+)
- **Production Deployments** (target: 10+)

---

## Definition of Done

A feature is "done" when:

1. ✅ Requirements are met per acceptance criteria
2. ✅ Tests written first (TDD) and all pass
3. ✅ Documentation updated (API, runbooks, architecture)
4. ✅ Observability instrumented (metrics, traces, logs)
5. ✅ Security reviewed and approved
6. ✅ Performance benchmarked and within SLA
7. ✅ Deployed to staging and verified
8. ✅ Approved by product owner
9. ✅ Merged to main branch
10. ✅ Released with changelog entry

---

## Risk Register

### Technical Risks
- **R1:** Database performance doesn't scale to 100k TPS → Mitigate: Benchmark early, optimize hot paths
- **R2:** Kafka becomes bottleneck → Mitigate: Make event streaming optional, support multiple brokers
- **R3:** Data corruption under Byzantine failure → Mitigate: Extensive chaos testing, formal verification

### Operational Risks
- **R4:** Complex deployment increases time-to-recovery → Mitigate: Automated deployment, extensive runbooks
- **R5:** Insufficient observability delays incident resolution → Mitigate: Instrument everything, clear SLIs

### Business Risks
- **R6:** Low adoption due to complexity → Mitigate: Excellent documentation, reference implementations
- **R7:** BIAN specification changes break compatibility → Mitigate: Version APIs, maintain backwards compatibility

---

## Open Source Governance

**License:** Apache 2.0
**Contribution Model:** Benevolent dictator / Technical steering committee
**Code of Conduct:** Contributor Covenant
**Communication:** GitHub Discussions, Slack workspace, monthly community calls
**Decision Making:** RFCs for major features, lazy consensus for minor changes
**Release Cadence:** Monthly minor releases, quarterly major releases

---

## Initial Milestones

**M1 (Week 4):** Core ledger APIs defined, TDD infrastructure established
**M2 (Week 8):** FinancialAccounting service domain functional
**M3 (Week 12):** PositionKeeping and CurrentAccount integrated
**M4 (Week 16):** Performance targets achieved, chaos testing passed
**M5 (Week 20):** v1.0 release candidate, documentation complete
**M6 (Week 24):** v1.0 released, first production deployment

---

## Appendix: Why Tilt?

**Tilt is the correct choice because:**

- **Fast Inner Loop:** Sub-second rebuilds and redeploys during development
- **Kubernetes Native:** Develop locally exactly as it runs in production
- **Multi-Service Orchestration:** Manages Go services, databases, Kafka, Redis simultaneously
- **Live Updates:** Code changes hot-reload without container rebuild
- **Resource Dashboard:** Single pane of glass for all services
- **Team Consistency:** Every developer has identical environment
- **CI/CD Alignment:** Tilt config translates directly to production manifests

Tilt eliminates "works on my machine" and accelerates TDD cycles.

---

## Appendix: Red-Green-Refactor Mandate

**Every feature follows:**

1. **Red:** Write failing test first
2. **Green:** Write minimal code to pass test
3. **Refactor:** Clean up code while keeping tests green
4. **Repeat:** Continue until acceptance criteria met

**Enforcement:**
- Pre-commit hooks verify tests exist for new code
- CI fails if test coverage drops
- Code review checklist includes TDD verification

**Benefits:**
- Design APIs before implementation
- Prevent regressions
- Living documentation via tests
- Confidence in refactoring

---

This PRD intentionally focuses on **what** we're building and **why**, leaving the **how** to implementation.
