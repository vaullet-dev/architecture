# ADR-003: Hybrid Database Strategy with Separate Analytics Database

## Status

**Accepted** (2026-08-27)
**Revised** (2026-08-27) — database-per-domain with strategic grouping: `rewards_db` and `auth_db`
split out, Bonus moved off `support_db`, Scheduler added, the analytics store renamed to `datawarehouse_db`,
and provisioning made conditional on deployed modules (see [ADR-005](005-module-composition-and-deployment-topology.md)).

## Context

The Vaullet walleting system uses a polyrepo microservices architecture with 14 services plus an Admin UI. Each service needs persistent storage for its domain data. We must decide how to structure databases across services.

### Key Requirements

1. **Data isolation**: Critical financial data (ledger) must be completely isolated
2. **Scalability**: High-volume services (transactions) need independent scaling
3. **Operational simplicity**: Minimize overhead for small team
4. **Cost efficiency**: Balance isolation with infrastructure costs
5. **Reporting capability**: Support complex analytics across multiple domains
6. **Technology flexibility**: Allow different databases for different needs

### Reporting Challenge

Financial systems require complex reporting that spans multiple services:
- Transaction reports with user info, bonuses, merchant data
- Fraud analytics across transactions, user behavior, patterns
- Business metrics: daily volume, bonus payouts, refund rates
- User lifetime value calculations

**Key insight**: Even with a shared database, cross-service reporting should not query operational databases directly. This would violate service boundaries and impact operational performance.

### The Trade-off

**Full separation** (database per service):
- ✅ Maximum isolation, independent scaling
- ❌ 15 databases to manage, higher cost, operational overhead

**Shared database** (all services):
- ✅ Simple, low cost, easy JOINs
- ❌ Tight coupling, schema coordination nightmare, no isolation

**Hybrid** (strategic separation):
- ✅ Isolation where critical, simplicity where acceptable
- ⚠️ Need clear criteria for what gets separated

## Decision

We will use a **hybrid database strategy** with clear separation criteria, plus a **dedicated analytics database** for reporting.

### Operational Databases (6 Databases)

#### 1. Vaullet Database (Isolated) 🔐

**Service**: Vaullet (Ledger Service) only

**Technology**: PostgreSQL with append-only constraints

**Schema**:
```sql
vaullet_db
├─ ledger_entries (immutable, append-only)
│   ├─ entry_id (PK)
│   ├─ transaction_id (FK)
│   ├─ user_id (indexed)
│   ├─ amount (NOT NULL)
│   ├─ type (DEBIT/CREDIT)
│   ├─ category (TRANSACTION/REFUND/BONUS/etc)
│   ├─ created_at (immutable)
│   └─ metadata (JSONB)
│
├─ balances (materialized view)
│   ├─ user_id (PK)
│   ├─ balance (computed from ledger_entries)
│   └─ last_updated_at
│
└─ audit_log (compliance)
    ├─ audit_id (PK)
    ├─ entry_id (FK)
    ├─ action (INSERT only)
    └─ timestamp
```

**Why isolated**:
- ✅ Source of truth for all money movements
- ✅ Immutable data (append-only) requires strict controls
- ✅ Compliance and audit requirements
- ✅ Different backup/retention policies (7+ years)
- ✅ No risk of accidental modification by other services

**Access control**: ONLY Vaullet Service can write. Other services call Vaullet REST API.

**Backup strategy**: Continuous backup, point-in-time recovery, 7-year retention

---

#### 2. Transactions Database (Isolated)

**Service**: Transaction Service only

**Technology**: PostgreSQL

**Schema**:
```sql
transactions_db
├─ transactions (mutable, workflow state)
│   ├─ transaction_id (PK)
│   ├─ user_id (indexed)
│   ├─ amount (NOT NULL)
│   ├─ currency
│   ├─ type (WITHDRAWAL/DEPOSIT/TRANSFER)
│   ├─ status (PENDING/APPROVED/COMPLETED/FAILED)
│   ├─ merchant_id
│   ├─ merchant_category
│   ├─ fraud_analysis_id
│   ├─ created_at
│   ├─ updated_at
│   └─ metadata (JSONB)
│
├─ refunds
│   ├─ refund_id (PK)
│   ├─ original_transaction_id (FK)
│   ├─ refund_amount
│   ├─ reason
│   ├─ status
│   └─ created_at
│
└─ transaction_links (bonuses, refunds, etc)
    ├─ transaction_id (FK)
    ├─ linked_entity_type (BONUS/REFUND)
    ├─ linked_entity_id
    └─ created_at
```

**Why isolated**:
- ✅ High transaction volume (>1000 TPS target)
- ✅ Needs independent scaling (read replicas, partitioning)
- ✅ Mutable state (status updates) separate from immutable ledger
- ✅ Complex indexes for different query patterns

**Access control**: ONLY Transaction Service writes

---

#### 3. Fraud Detection Database (Isolated)

**Service**: Fraud Detection Service, Risk Management Service

**Technology**: PostgreSQL + Elasticsearch

> **Elasticsearch is out, decided 2026-09-17. The replacement is deferred.**
>
> Elasticsearch has not been open source since 2021: SSPL / Elastic License 2.0, with AGPLv3 added as
> a third option in 2024. **None of the three is permissive.** Vaullet ships into customer clusters
> under contract — the same reasoning that rejected BSL-licensed HashiCorp Vault in favour of OpenBao.
> Paying for, or asking a customer to accept, a licence of that shape is ruled out.
>
> **What replaces it is an open question, to be settled when Fraud Detection is built.** Two
> candidates:
>
> - **PostgreSQL alone** — `tsvector` + GIN for search, `pg_trgm` for fuzzy matching, JSONB for
>   case documents. Removes a core component outright: ~4 GiB, one technology, and a visible cut to
>   ADR-005's per-customer floor. Viable only if "pattern detection" here does not actually need a
>   search engine, which this ADR asserts but has never tested.
> - **OpenSearch** — Apache 2.0, Linux Foundation since 2024. Query DSL and mappings port nearly
>   unchanged from Elasticsearch; **client libraries do not.** Elastic's official clients check a
>   product header and refuse to connect to anything else, so this is `opensearch-java` and
>   `spring-data-opensearch`, not a drop-in dependency swap.
>
> **This is a core decision, not an optional one.** Fraud Detection is Category 1 in ADR-005, so
> whatever is chosen ships to every customer.

**Schema**:
```sql
fraud_db (PostgreSQL)
├─ fraud_rules
│   ├─ rule_id (PK)
│   ├─ rule_name
│   ├─ rule_type (VELOCITY/AMOUNT/LOCATION)
│   ├─ threshold_value
│   ├─ is_active
│   └─ created_at
│
├─ fraud_cases
│   ├─ case_id (PK)
│   ├─ user_id (indexed)
│   ├─ transaction_id (indexed)
│   ├─ risk_score
│   ├─ status (OPEN/INVESTIGATING/CLOSED)
│   ├─ assigned_to
│   ├─ resolution
│   └─ created_at
│
└─ fraud_analysis_history
    ├─ analysis_id (PK)
    ├─ transaction_id
    ├─ risk_score
    ├─ ml_model_score
    ├─ decision (APPROVE/REVIEW/BLOCK)
    ├─ rules_triggered (JSONB array)
    └─ created_at

fraud_search (Elasticsearch)
└─ fraud_patterns (full-text search, pattern matching)
    ├─ pattern_id
    ├─ pattern_type
    ├─ description (searchable)
    ├─ affected_transactions
    └─ detected_at
```

**Why isolated**:
- ✅ Different technology needs (Elasticsearch for pattern detection)
- ✅ Sensitive data (fraud cases, investigations)
- ✅ Complex ML workloads (separate from transactional DBs)
- ✅ Independent scaling for fraud detection spike

**Access control**: Fraud Detection Service and Risk Management Service share this DB

---

#### 4. Support Services Database (Shared)

**Services**: Limits, Subscription, Notification, Audit, Scheduler (share this DB)

> **Revised**: Bonus moved to `rewards_db` — see #5. Scheduler added; it is platform infrastructure
> (ADR-005) and always deployed.

**Technology**: PostgreSQL with schema-per-service

**Schema**:
```sql
support_db

-- Limits Service schema
limits_schema
├─ user_limits
│   ├─ user_id (PK)
│   ├─ daily_limit
│   ├─ monthly_limit
│   ├─ daily_used (updated from events)
│   ├─ monthly_used
│   └─ last_reset_date
│
└─ limit_configurations
    ├─ config_id (PK)
    ├─ user_type (BASIC/PREMIUM)
    ├─ default_daily_limit
    └─ default_monthly_limit

-- Subscription Service schema
subscription_schema
├─ subscriptions
│   ├─ subscription_id (PK)
│   ├─ user_id (indexed)
│   ├─ plan_id
│   ├─ status (ACTIVE/SUSPENDED/CANCELLED)
│   ├─ next_billing_date
│   ├─ retry_count
│   └─ created_at
│
└─ subscription_plans
    ├─ plan_id (PK)
    ├─ plan_name
    ├─ amount
    └─ billing_cycle

-- Notification Service schema
notification_schema
├─ notification_templates
│   ├─ template_id (PK)
│   ├─ template_name
│   ├─ template_body
│   └─ channel (EMAIL/SMS/PUSH)
│
└─ notification_log
    ├─ notification_id (PK)
    ├─ user_id (indexed)
    ├─ template_id
    ├─ sent_at
    └─ status (SENT/FAILED)

-- Audit Service schema
audit_schema
└─ audit_events
    ├─ event_id (PK)
    ├─ event_type (indexed)
    ├─ user_id (indexed)
    ├─ service_name (indexed)
    ├─ event_data (JSONB)
    └─ created_at (partitioned by month)

-- Scheduler Service schema
scheduler_schema
├─ scheduled_jobs
│   ├─ job_id (PK)
│   ├─ job_type (indexed)
│   ├─ cron_expression
│   ├─ next_run_at (indexed)
│   └─ is_active
│
└─ job_executions
    ├─ execution_id (PK)
    ├─ job_id (FK)
    ├─ started_at
    ├─ finished_at
    └─ status (SUCCESS/FAILED/RETRYING)
```

**Conditional schemas**: `subscription_schema` and `notification_schema` are created only when their
modules are sold (ADR-005). `limits_schema`, `audit_schema` and `scheduler_schema` are always
present — Limits and Audit are core, Scheduler is infrastructure.

**Why shared**:
- ✅ Lower transaction volumes (combined <100 TPS)
- ✅ Closely related domains (user preferences, configurations)
- ✅ No special technology requirements (standard PostgreSQL)
- ✅ Simpler operations (one DB to backup/monitor)
- ✅ Cost-effective for smaller services
- ✅ Can be split later if volume grows

**Access control**: Each service has its own PostgreSQL schema with dedicated credentials

**Schema isolation**:
```sql
-- Limits Service user can only access limits_schema
GRANT USAGE ON SCHEMA limits_schema TO limits_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA limits_schema TO limits_service_user;

-- Subscription Service user can only access subscription_schema
GRANT USAGE ON SCHEMA subscription_schema TO subscription_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA subscription_schema TO subscription_service_user;
```

**Future evolution**: If any service outgrows the shared DB, it can be split into its own database.

---

#### 5. Rewards Database (Shared)

**Services**: Bonus, Loyalty, Referral

**Technology**: PostgreSQL with schema-per-service

**Why grouped**: One commercial domain, similar access patterns, and frequently reported on together
("total rewards earned"). Schema-per-service keeps the boundaries enforced.

```sql
rewards_db
├─ bonus_schema      (bonuses, bonus_campaigns, bonus_rules)
├─ loyalty_schema    (loyalty_points, loyalty_tiers, redemptions, tier_configurations)
└─ referral_schema   (referral_links, referrals, referral_rewards, referral_campaigns)
```

**Conditionally provisioned**: all three services are sellable modules (ADR-005). `rewards_db` is
created only if at least one is sold, and each schema only for its own module. A core-only
deployment has no `rewards_db` at all.

**Boundary with the ledger**: these schemas hold *rules* — wagering terms, tier thresholds, campaign
eligibility. They never hold balances. Promotional *money* lives in `vaullet_db` balance buckets
(ADR-004), reached only by emitting grant events. Two services must not both believe they know what
a user's money is.

---

#### 6. Auth Database (Isolated) 🔐

**Service**: Auth Service

**Technology**: PostgreSQL (Keycloak + Vaullet identity schemas) + Redis (status cache)

**Why isolated**: Credentials and MFA secrets warrant a separate blast radius, separate credentials,
and a separate backup/retention policy from operational data. Since [ADR-006](006-authentication-and-identity.md)
adopted Keycloak, this database also hosts a third-party schema with its own migration lifecycle —
another reason not to co-locate it with services we control.

```sql
auth_db

-- Owned entirely by Keycloak; runs its own migrations. Never written to by us.
keycloak_schema
└─ (Keycloak-managed: users, credentials, MFA, roles, brokers, sessions)

-- Vaullet's own identity tables, separate credentials
identity_schema
├─ accounts        (account_id ↔ keycloak_sub, status) — the ledger's anchor
└─ api_clients     (machine clients, scopes)
```

**No foreign key crosses the schema boundary.** `accounts.keycloak_sub` is a plain `UUID`, not a
reference, so a Keycloak major upgrade migrating its own schema cannot break ours.

Single-tenant deployment (ADR-005) means no `tenant_id`: every row in this database belongs to the
one customer that owns the cluster.

---

#### 7. Redis Cluster (Shared Cache)

**Services**: Auth, Transaction, Limits, Fraud Detection (shared)

**Technology**: Redis Cluster (3 masters, 3 replicas)

**Namespaces**:
```
redis_cluster
├─ session:* (Auth Service - user sessions)
├─ balance:cache:* (Vaullet - read-through balance cache)
├─ limits:counter:* (Limits Service - real-time counters)
└─ fraud:cache:* (Fraud Detection - recent transaction cache)
```

> **Revised**: `lock:balance:*` removed. [ADR-004](004-atomic-balance-reservations.md) moved balance
> consistency into a Postgres transaction inside Vaullet, so Redis no longer holds locks and is no
> longer on the correctness path. Losing Redis now degrades latency; it cannot cause an overdraft.

**Why shared**: Redis is designed for namespaced multi-workload use

---

### Analytics Database (Separate Read-Optimized Store)

#### 8. Analytics Database

**Purpose**: Complex analytics, business intelligence, reporting

**Technology**: PostgreSQL (Phase 1) → ClickHouse/TimescaleDB (Phase 2)

**Schema**:
```sql
datawarehouse_db

-- Denormalized transaction analytics
├─ transaction_analytics
│   ├─ transaction_id (PK)
│   ├─ user_id (indexed)
│   ├─ amount
│   ├─ currency
│   ├─ merchant_id
│   ├─ merchant_category
│   ├─ transaction_type
│   ├─ transaction_status
│   ├─ bonus_amount (joined from bonus events)
│   ├─ fraud_risk_score (joined from fraud events)
│   ├─ was_refunded (boolean)
│   ├─ refund_amount
│   └─ created_at (partitioned)
│
-- Pre-aggregated daily metrics
├─ daily_metrics
│   ├─ date (PK)
│   ├─ total_transactions
│   ├─ total_volume
│   ├─ total_refunds
│   ├─ total_bonuses_paid
│   ├─ fraud_detected_count
│   ├─ fraud_blocked_count
│   └─ average_transaction_amount
│
-- User summaries
├─ user_summaries
│   ├─ user_id (PK)
│   ├─ total_transactions
│   ├─ total_spent
│   ├─ total_bonuses_earned
│   ├─ total_refunds_received
│   ├─ fraud_incidents_count
│   ├─ first_transaction_date
│   ├─ last_transaction_date
│   └─ lifetime_value (computed)
│
-- Merchant analytics
└─ merchant_analytics
    ├─ merchant_id (PK)
    ├─ merchant_category
    ├─ total_transactions
    ├─ total_volume
    ├─ fraud_rate
    └─ last_updated_at
```

**Data flow**:
```
Operational Services → Kafka Events → Reporting ETL Service → Reporting DB
```

**ETL Service**:
```java
@Service
public class ReportingETLService {

    @KafkaListener(topics = "transaction.created")
    public void handleTransactionCreated(TransactionCreatedEvent event) {
        reportingRepository.save(TransactionAnalytics.builder()
            .transactionId(event.getPayload().getTransactionId())
            .userId(event.getPayload().getUserId())
            .amount(event.getPayload().getAmount())
            .merchantCategory(event.getPayload().getMerchantCategory())
            .createdAt(event.getTimestamp())
            .build());
    }

    @KafkaListener(topics = "bonus.applied")
    public void handleBonusApplied(BonusAppliedEvent event) {
        reportingRepository.updateBonusAmount(
            event.getPayload().getTransactionId(),
            event.getPayload().getBonusAmount()
        );
    }

    @KafkaListener(topics = "fraud.analysis.completed")
    public void handleFraudAnalysis(FraudAnalysisCompletedEvent event) {
        reportingRepository.updateFraudScore(
            event.getPayload().getTransactionId(),
            event.getPayload().getRiskScore()
        );
    }

    @KafkaListener(topics = "refund.completed")
    public void handleRefundCompleted(RefundCompletedEvent event) {
        reportingRepository.markAsRefunded(
            event.getPayload().getOriginalTransactionId(),
            event.getPayload().getRefundAmount()
        );
    }

    @Scheduled(cron = "0 0 1 * * *")  // Daily at 1 AM
    public void calculateDailyMetrics() {
        // Pre-aggregate metrics for fast dashboard queries
        LocalDate yesterday = LocalDate.now().minusDays(1);
        DailyMetrics metrics = reportingRepository.calculateDailyMetrics(yesterday);
        dailyMetricsRepository.save(metrics);
    }
}
```

**Access**: Read-only for most users, BI tools, dashboards

---

## Database Summary

| Database | Services | Technology | Purpose | Volume | Always provisioned? |
|----------|----------|------------|---------|--------|---------------------|
| vaullet_db | Vaullet | PostgreSQL | Immutable ledger + balance buckets, source of truth | High (append-only) | Yes (core) |
| transactions_db | Transaction (incl. Refund) | PostgreSQL | Transaction lifecycle, mutable state | Very High (>1000 TPS) | Yes (core) |
| fraud_db | Fraud Detection, Risk Management | PostgreSQL + Elasticsearch | Fraud cases, ML, pattern detection | Medium | Yes (Fraud is core) |
| auth_db | Auth | PostgreSQL | Credentials, MFA, roles, API clients | Medium | Yes (core) |
| support_db | Limits, Subscription, Notification, Audit, Scheduler | PostgreSQL (multi-schema) | Supporting services | Low-Medium | Yes — schemas conditional |
| rewards_db | Bonus, Loyalty, Referral | PostgreSQL (multi-schema) | Reward rules and campaigns | Medium | **No** — only if a rewards module is sold |
| datawarehouse_db | Reporting ETL | PostgreSQL → ClickHouse | Analytics, BI, reporting | High (read-heavy) | **No** — only if Reporting ETL is sold |
| redis_cluster | Auth, Vaullet, Limits, Fraud | Redis Cluster | Sessions, counters, cache | High (ephemeral) | Yes (infrastructure) |

**Total infrastructure (full deployment)**: 7 PostgreSQL databases + 1 Redis Cluster + 1 Elasticsearch cluster = **9 components**

> **How many PostgreSQL *servers*? Three.** Revised 2026-09-18, superseding the "One" written here a
> day earlier. That note was right that this document counts *databases* and never says how many
> servers, and right that seven instances would be seven times the memory and seven backup
> configurations. It reached the wrong number because it weighed **cost** and never asked what
> isolation is actually *usable*.
>
> **The grouping principle is the restore boundary**: databases that must be restored to the same
> point in time belong in the same instance; databases that can be rewound independently may be
> separated.
>
> Restoring `vaullet_db` to 09:00 while `transactions_db` stays at 11:00 leaves the ledger and the
> transaction log disagreeing about money that exists. Those two can therefore *never* be restored
> independently, so isolating them from each other buys nothing that could ever be spent. `auth_db`
> has no such coupling — restoring identity without rewinding the ledger is coherent. That asymmetry,
> not the component count, is what decides the grouping:
>
> | Cluster | Databases | Why together |
> |---|---|---|
> | `money` | `vaullet_db`, `transactions_db` | one restore unit; strictest durability |
> | `platform` | `auth_db`, `support_db` | independent of money; no shared point-in-time |
> | `analytics` | `fraud_db` *(+ `rewards_db`, `datawarehouse_db` when sold)* | read-heavy, relaxed durability |
>
> The isolation is now **physical between groups and logical within them** — a role per service and no
> cross-database grants still holds inside each cluster. The measured cost of the extra two instances
> is ~2 GiB of memory and two more backup targets, against ~37 GiB free; the cost the earlier note
> feared was seven instances, not three.
>
> **As built, 2026-09-18.** CloudNativePG 1.30, PostgreSQL 18.6, one instance per cluster pinned to a
> node (`local-path` binds a volume to a node permanently, so the placement is declared rather than
> discovered). `dataChecksums` on and the **builtin** collation provider with `C.UTF-8` — both chosen
> because neither can be changed after `initdb` without a dump and restore, and because glibc
> collation changing between base images silently invalidates every text index built under the old
> rules. Single instances are deliberate: `instances: 3` later is a number change that builds standbys
> with `pg_basebackup`, not a redesign. What is accepted today is that draining the node holding a
> primary is an outage for that group.

**Minimum (core-only contract)**: 5 PostgreSQL databases (`vaullet`, `transactions`, `fraud`, `auth`,
`support`) + Redis + Elasticsearch = **7 components**

### Provisioning is a function of the contract

Per [ADR-005](005-module-composition-and-deployment-topology.md), a customer's deployment contains
only the modules they bought, and database provisioning follows: `rewards_db` and
`datawarehouse_db` exist only when something needs them, and conditional schemas inside `support_db`
are created only for deployed services. A customer without Bonus pays for neither the pod nor the
schema.

---

## Consequences

### Positive Consequences

✅ **Critical data isolated**: Vaullet ledger cannot be corrupted by other services
✅ **Independent scaling**: High-volume services (transactions, fraud) scale independently
✅ **Technology flexibility**: Can use Elasticsearch for fraud patterns, ClickHouse for analytics
✅ **Operational balance**: Not managing 12 separate databases
✅ **Cost-effective**: Shared DB for low-volume services reduces infrastructure cost
✅ **Clear boundaries**: Each service's data ownership is explicit
✅ **Reporting decoupled**: Analytical queries don't impact operational performance
✅ **Room to evolve**: Shared DB services can be split later if needed

### Negative Consequences

⚠️ **No ACID across services**: Can't use database transactions spanning Transaction + Vaullet
⚠️ **Data duplication**: Reporting DB duplicates data from operational DBs
⚠️ **ETL complexity**: Need to maintain Reporting ETL Service
⚠️ **Eventual consistency**: Reporting data lags behind operational data (seconds)
⚠️ **More databases than shared approach**: 6 databases vs 1-2 in monolith
⚠️ **Schema coordination for shared DB**: Bonus and Limits teams share one database

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Shared DB becomes bottleneck | Performance degradation | Monitor per-schema metrics, split high-volume schemas first |
| ETL lag for reporting | Stale dashboards | Monitor lag metrics, alert if >1 minute lag, add stream processing if needed |
| Vaullet DB is single point of failure | No transactions can proceed | HA setup with automatic failover, continuous backup, DR plan |
| Reporting DB sync fails | Missing data in reports | Dead letter queue for failed events, reconciliation job, alerts |
| Schema changes in shared DB | Service coordination overhead | Backward-compatible migrations only, versioned schemas |

---

## Alternatives Considered

### Alternative 1: Database Per Service (Full Separation)

**Description**: Each of the 14 services gets its own dedicated database.

**Pros**:
- Maximum isolation
- True microservices
- Independent everything

**Cons**:
- 12+ PostgreSQL instances to manage
- High infrastructure cost
- Operational overhead for small team
- Over-engineered for low-volume services (Bonus, Notification)

**Why Rejected**: Over-engineered for current needs. Support services (Bonus, Limits, etc.) don't need isolation. Can always split later.

---

### Alternative 2: Shared Database (All Services)

**Description**: All services share one database, each with their own tables.

**Pros**:
- Simple (one database)
- Easy JOINs for reports
- Low cost

**Cons**:
- Tight coupling (schema changes affect everyone)
- No isolation (one service's bad query impacts all)
- Can't scale databases independently
- Deployment coordination nightmare
- No technology flexibility (everyone uses PostgreSQL)

**Why Rejected**: Violates microservices principles. Vaullet ledger is too critical to share database with other services.

---

### Alternative 3: Query Operational Databases for Reporting

**Description**: Use existing operational databases for reporting queries instead of separate analytics DB.

**Example**:
```sql
-- Report query hitting operational DBs
SELECT t.*, b.bonus_amount, f.risk_score
FROM transactions_db.transactions t
LEFT JOIN rewards_db.bonus_schema.bonuses b ON t.transaction_id = b.transaction_id
LEFT JOIN fraud_db.fraud_analysis_history f ON t.transaction_id = f.transaction_id
WHERE t.created_at > NOW() - INTERVAL '30 days';
```

**Pros**:
- No ETL needed
- Real-time data (no lag)
- One less database

**Cons**:
- **Violates service boundaries**: Reporting directly queries other services' databases
- **Performance impact**: Heavy analytical queries slow down operational transactions
- **Can't use cross-database JOINs**: PostgreSQL doesn't support JOINs across DB instances
- **Tight coupling**: Report changes when schemas change

**Why Rejected**:
- Even if technically possible (shared DB), it violates service encapsulation
- Analytical queries would degrade operational performance (transactions, fraud detection)
- Industry best practice: CQRS pattern with separate read models

---

### Alternative 4: Event Sourcing for Everything

**Description**: Store only events, rebuild state from event log.

**Pros**:
- Full audit trail
- Time-travel queries
- Natural fit for reporting

**Cons**:
- Paradigm shift for team
- Complex to implement
- Overkill for most services
- Performance concerns for high-volume events

**Why Rejected**: Over-engineered. Only Vaullet benefits from event sourcing (ledger is already event-sourced). Other services don't need it.

---

## Implementation Notes

### Database Provisioning

**Development environment** (Docker Compose):
```yaml
version: '3.8'

services:
  vaullet-db:
    image: postgres:15
    environment:
      POSTGRES_DB: vaullet_db
      POSTGRES_USER: vaullet_user
      POSTGRES_PASSWORD: ${VAULLET_DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - vaullet-data:/var/lib/postgresql/data

  transactions-db:
    image: postgres:15
    environment:
      POSTGRES_DB: transactions_db
      POSTGRES_USER: transactions_user
      POSTGRES_PASSWORD: ${TRANSACTIONS_DB_PASSWORD}
    ports:
      - "5433:5432"

  fraud-db:
    image: postgres:15
    environment:
      POSTGRES_DB: fraud_db
      POSTGRES_USER: fraud_user
      POSTGRES_PASSWORD: ${FRAUD_DB_PASSWORD}
    ports:
      - "5434:5432"

  support-db:
    image: postgres:15
    environment:
      POSTGRES_DB: support_db
      POSTGRES_USER: support_user
      POSTGRES_PASSWORD: ${SUPPORT_DB_PASSWORD}
    ports:
      - "5435:5432"

  auth-db:
    image: postgres:15
    environment:
      POSTGRES_DB: auth_db
      POSTGRES_USER: auth_user
      POSTGRES_PASSWORD: ${AUTH_DB_PASSWORD}
    ports:
      - "5436:5432"

  # Conditional: only when a rewards module is sold (ADR-005)
  rewards-db:
    image: postgres:15
    environment:
      POSTGRES_DB: rewards_db
      POSTGRES_USER: rewards_user
      POSTGRES_PASSWORD: ${REWARDS_DB_PASSWORD}
    ports:
      - "5437:5432"

  # Conditional: only when Reporting ETL is sold (ADR-005)
  datawarehouse-db:
    image: postgres:15
    environment:
      POSTGRES_DB: datawarehouse_db
      POSTGRES_USER: reporting_user
      POSTGRES_PASSWORD: ${DATAWAREHOUSE_DB_PASSWORD}
    ports:
      - "5438:5432"

  redis-cluster:
    image: redis:7
    ports:
      - "6379:6379"

  elasticsearch:
    image: elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"
```

**Production** (Kubernetes):
- Use managed databases (AWS RDS, GCP Cloud SQL, Azure Database)
- Or PostgreSQL StatefulSets with persistent volumes
- Redis: Managed Redis (ElastiCache, MemoryStore) or Redis Operator

### Service Configuration

**Spring Boot application.yml**:
```yaml
# Transaction Service
spring:
  datasource:
    url: jdbc:postgresql://transactions-db:5432/transactions_db
    username: transactions_user
    password: ${TRANSACTIONS_DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # Never auto-create in production
  flyway:
    enabled: true
    locations: classpath:db/migration
```

**Flyway migrations** (schema versioning):
```sql
-- transactions-db/src/main/resources/db/migration/V1__create_transactions_table.sql
CREATE TABLE transactions (
    transaction_id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    type VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
```

### Shared Database Schema Isolation

```sql
-- support_db initialization (always-present schemas)
CREATE SCHEMA limits_schema;
CREATE SCHEMA audit_schema;
CREATE SCHEMA scheduler_schema;

-- Conditional: created only when the module is sold (ADR-005)
CREATE SCHEMA subscription_schema;   -- if modules.subscription.enabled
CREATE SCHEMA notification_schema;   -- if modules.notification.enabled

-- Create service-specific users
CREATE USER limits_service_user       WITH PASSWORD 'secure_password';
CREATE USER subscription_service_user WITH PASSWORD 'secure_password';

-- Grant access only to respective schemas
GRANT USAGE ON SCHEMA limits_schema TO limits_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA limits_schema TO limits_service_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA limits_schema TO limits_service_user;

GRANT USAGE ON SCHEMA subscription_schema TO subscription_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA subscription_schema TO subscription_service_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA subscription_schema TO subscription_service_user;

-- Prevent cross-schema access
REVOKE ALL ON SCHEMA limits_schema       FROM subscription_service_user;
REVOKE ALL ON SCHEMA subscription_schema FROM limits_service_user;
```

```sql
-- rewards_db initialization (provisioned only if a rewards module is sold)
CREATE SCHEMA bonus_schema;      -- if modules.bonus.enabled
CREATE SCHEMA loyalty_schema;    -- if modules.loyalty.enabled
CREATE SCHEMA referral_schema;   -- if modules.referral.enabled

CREATE USER bonus_service_user WITH PASSWORD 'secure_password';
GRANT USAGE ON SCHEMA bonus_schema TO bonus_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA bonus_schema TO bonus_service_user;

REVOKE ALL ON SCHEMA loyalty_schema  FROM bonus_service_user;
REVOKE ALL ON SCHEMA referral_schema FROM bonus_service_user;
```

### Reporting ETL Service

```java
@SpringBootApplication
public class ReportingETLApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReportingETLApplication.class, args);
    }
}

@Service
@Slf4j
public class TransactionAnalyticsETL {

    @Autowired
    private TransactionAnalyticsRepository analyticsRepo;

    // Consume transaction created event
    @KafkaListener(topics = "transaction.created", groupId = "reporting-etl")
    public void processTransactionCreated(TransactionCreatedEvent event) {
        log.info("Processing transaction.created: {}", event.getPayload().getTransactionId());

        TransactionAnalytics analytics = TransactionAnalytics.builder()
            .transactionId(event.getPayload().getTransactionId())
            .userId(event.getPayload().getUserId())
            .amount(event.getPayload().getAmount())
            .currency(event.getPayload().getCurrency())
            .merchantId(event.getPayload().getMerchantId())
            .merchantCategory(event.getPayload().getMerchantCategory())
            .transactionType(event.getPayload().getType())
            .transactionStatus(event.getPayload().getStatus())
            .createdAt(event.getTimestamp())
            .build();

        analyticsRepo.save(analytics);
    }

    // Enrich with bonus data
    @KafkaListener(topics = "bonus.applied", groupId = "reporting-etl")
    public void processBonusApplied(BonusAppliedEvent event) {
        log.info("Updating bonus for transaction: {}", event.getPayload().getTransactionId());

        analyticsRepo.findById(event.getPayload().getTransactionId())
            .ifPresent(analytics -> {
                analytics.setBonusAmount(event.getPayload().getBonusAmount());
                analyticsRepo.save(analytics);
            });
    }

    // Enrich with fraud score
    @KafkaListener(topics = "fraud.analysis.completed", groupId = "reporting-etl")
    public void processFraudAnalysis(FraudAnalysisCompletedEvent event) {
        if (event.getPayload().getTransactionId() != null) {
            analyticsRepo.findById(event.getPayload().getTransactionId())
                .ifPresent(analytics -> {
                    analytics.setFraudRiskScore(event.getPayload().getRiskScore());
                    analyticsRepo.save(analytics);
                });
        }
    }

    // Daily aggregation job
    @Scheduled(cron = "0 0 1 * * *")  // 1 AM daily
    public void calculateDailyMetrics() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        log.info("Calculating daily metrics for {}", yesterday);

        DailyMetrics metrics = analyticsRepo.calculateDailyMetrics(yesterday);
        dailyMetricsRepo.save(metrics);

        log.info("Daily metrics saved: {} transactions, {} volume",
            metrics.getTotalTransactions(), metrics.getTotalVolume());
    }
}
```

### Migration from Shared to Isolated (Future)

If a service in a shared database outgrows it — illustrated with Loyalty leaving `rewards_db`:

**Step 1**: Create new database
```bash
createdb loyalty_db
```

**Step 2**: Export schema and data
```bash
pg_dump -h rewards-db -U rewards_user -n loyalty_schema rewards_db > loyalty_export.sql
```

**Step 3**: Import to new database
```bash
psql -h loyalty-db -U loyalty_user loyalty_db < loyalty_export.sql
```

**Step 4**: Update Loyalty Service config
```yaml
spring:
  datasource:
    url: jdbc:postgresql://loyalty-db:5432/loyalty_db  # Changed
```

**Step 5**: Deploy and verify

**Step 6**: Drop from shared DB
```sql
DROP SCHEMA loyalty_schema CASCADE;
```

Because deployments are per-customer (ADR-005), this migration runs per cluster and only for
customers whose volume warrants it. Different customers may legitimately sit at different points
in this evolution.

---

## Monitoring and Observability

### Key Metrics

**Per database**:
- Connection pool utilization
- Query latency (p50, p95, p99)
- Active connections
- Lock waits
- Replication lag (for read replicas)
- Disk I/O

**Reporting ETL**:
- Event processing lag (time between event published and processed)
- Failed event count
- Reporting DB sync lag

**Alerts**:
- Vaullet DB down → P0 (critical, all transactions stop)
- Transactions DB down → P0
- Support DB down → P1 (some features degraded)
- Reporting ETL lag > 5 minutes → P2
- Connection pool exhaustion → P1

### Backup Strategy

| Database | Backup Frequency | Retention | RPO | RTO |
|----------|-----------------|-----------|-----|-----|
| vaullet_db | Continuous (WAL) | 7 years | 0 (point-in-time) | 15 min |
| transactions_db | Hourly snapshots | 30 days | 1 hour | 30 min |
| fraud_db | Daily snapshots | 90 days | 24 hours | 1 hour |
| support_db | Daily snapshots | 30 days | 24 hours | 1 hour |
| datawarehouse_db | Daily snapshots | 7 days | 24 hours | 2 hours (rebuildable from events) |

> **The table above stays as the product requirement. The reference cluster does not meet it.**
> Decided 2026-09-17, and written this way round on purpose: the requirement is not lowered to match
> what one rented machine can do.
>
> **Why it cannot be met here.** PITR needs continuous WAL archiving to an object store. CloudNativePG
> offers no volume-snapshot alternative on this cluster either — `local-path` is not a CSI driver that
> supports snapshots, so there is no `VolumeSnapshotClass` at all. An object store means either paying
> for one or running one on the same RAID1 array, and
> [ADR-015](015-cluster-topology-three-servers-on-one-machine.md) admits no spend.
>
> **What the reference cluster runs instead**: a `pg_dump` CronJob to a local PVC, per database.
>
> | | Requirement | Reference cluster |
> |---|---|---|
> | Mechanism | Continuous WAL archiving, PITR | Logical dump on a schedule |
> | `vaullet_db` RPO | 0 | **the dump interval** |
> | Recovers a bad migration or a wrong `WHERE` | ✅ | ✅ *(to the last dump)* |
> | Recovers losing the machine | ✅ | ❌ — the dump is on the same array |
>
> **The gap is one configuration change, not a redesign.** A customer deployment with object storage
> points CloudNativePG's `barmanObjectStore` at it and gets the table above as written. The reference
> cluster is a reduced configuration of the same design, which is the only form of this compromise
> worth making — and the reason to keep the requirement here rather than edit it down.
>
> **As built and DRILLED, 2026-09-18.** A `CronJob` in namespace `postgres` (`apps/db/postgres/backup.yaml`)
> runs daily at 02:30 UTC in the server's own image, so `pg_dump` can never be version-skewed from the
> server it dumps. Per cluster it writes `pg_dumpall --globals-only` plus one `-Fc` dump per database
> to a `local-path` PVC pinned to **k8s-3** — the node holding `analytics`, chosen so losing that one
> VM does not take both the money database and every backup of it. Losing the *host* still loses both;
> that is the compromise this section already states.
>
> Two properties worth keeping if this is ever rewritten. Every dump is **verified at write time**
> with `pg_restore -l`, which parses the table of contents, so a truncated file is caught the day it is
> written rather than during a restore under pressure. And **pruning only runs after a clean
> verification** — an old good dump beats a new broken one.
>
> The restore was **drilled, not assumed**, on 2026-09-18: an actual CronJob artifact restored in 38 ms,
> and a 50,000-row round trip came back with an identical MD5 of the whole table. Measured throughput
> was ~6 MB/s dumping and ~11 MB/s restoring, so a 1 GB `vaullet_db` is roughly three minutes out and
> two back. That is the number to plan an incident around.
>
> Dump volume is unmeasured. Watch it: it shares the array with everything else, and ADR-015 leaves
> roughly 220 GB for all data.

---

## Evolution Path

### Phase 1 (Current): Hybrid with PostgreSQL Reporting

- 6 operational databases + `datawarehouse_db` + Redis + Elasticsearch
- PostgreSQL for reporting
- Simple Kafka consumer for ETL

### Phase 2 (6-12 months): Optimize Reporting

When reporting queries become slow (>5 seconds):
- Migrate from PostgreSQL to **ClickHouse** or **TimescaleDB**
- Add stream processing (Kafka Streams, Flink) for real-time aggregations
- Introduce data warehouse concepts (star schema, fact tables)

### Phase 3 (12-24 months): Split Shared DBs if Needed

If a service in `rewards_db` or `support_db` grows significantly:
- Split `bonus_schema` out of `rewards_db` into `bonus_db`
- Split `limits_schema` out of `support_db` into `limits_db`
- Follow migration procedure above

### Phase 4 (Future): Cloud Data Warehouse

When data volume is massive (billions of rows):
- Migrate reporting to cloud data warehouse (BigQuery, Snowflake, Redshift)
- Keep operational databases as-is
- Stream events to cloud storage → warehouse ingestion

---

## References

- [Database per Service Pattern](https://microservices.io/patterns/data/database-per-service.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [PostgreSQL Schema Management](https://www.postgresql.org/docs/current/ddl-schemas.html)
- [ClickHouse for Analytics](https://clickhouse.com/docs/en/intro)
- Related ADRs:
  - [ADR-001: Redis Distributed Locking](001-distributed-locking-for-balance-consistency.md)
  - [ADR-002: Shared Contracts Strategy](002-shared-contracts-versioning-strategy.md)

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
**Next Review**: 2027-02-27 (6 months) - Evaluate if support DB needs to be split, assess reporting DB performance
