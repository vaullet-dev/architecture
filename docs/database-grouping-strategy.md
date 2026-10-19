# Database Grouping Strategy

## Overview

**Decision**: Database-per-domain with strategic grouping to balance isolation with operational overhead.

**Services**: 14 backend services + Admin UI
**Databases**: 6 operational PostgreSQL + 1 analytics PostgreSQL + Redis Cluster + Elasticsearch
= **9 infrastructure components** in a full deployment

**Provisioning is conditional.** Per [ADR-005](adr/005-module-composition-and-deployment-topology.md),
each customer runs their own cluster containing only the modules their contract includes. `rewards_db`
and `datawarehouse_db` exist only when something needs them; some `support_db` schemas likewise. A
core-only deployment runs **7 components** (5 PostgreSQL + Redis + Elasticsearch).

---

## Database Groups

### 1. Vaullet Database (Isolated) 🔐

**Service**: Vaullet (Ledger Service)

**Why isolated**:
- Critical financial source of truth
- Append-only, immutable
- Different backup/retention (7 years)
- MUST be completely isolated

```sql
vaullet_db (PostgreSQL)
├─ ledger_entries
├─ balances (materialized view)
└─ audit_log
```

---

### 2. Transactions Database (Isolated)

**Service**: Transaction Service

**Why isolated**:
- Highest volume (>1000 TPS)
- Needs independent scaling (read replicas, partitioning)
- Mutable transaction state

```sql
transactions_db (PostgreSQL)
├─ transactions
├─ refunds
└─ transaction_metadata
```

**Note**: Refund Service could be part of Transaction Service or separate. If separate, it shares this database.

---

### 3. Fraud Database (Isolated)

**Services**: Fraud Detection Service, Risk Management Service

**Why isolated**:
- Different technology needs (PostgreSQL + Elasticsearch)
- Sensitive investigation data
- ML workloads

```sql
fraud_db (PostgreSQL)
├─ fraud_rules
├─ fraud_cases
├─ fraud_analysis_history
└─ risk_investigations

fraud_search (Elasticsearch)
└─ fraud_patterns
```

---

### 4. Rewards Database (Shared) — CONDITIONAL

**Services**: Bonus Service, Loyalty Service, Referral Service

**Why shared**:
- Closely related domain (all rewards/incentives)
- Often query together (e.g., "total rewards earned")
- Similar data patterns
- Schema-per-service isolation

```sql
rewards_db (PostgreSQL)

-- Bonus Service schema
bonus_schema
├─ bonuses
├─ bonus_campaigns
└─ bonus_rules

-- Loyalty Service schema
loyalty_schema
├─ loyalty_points
├─ loyalty_tiers
├─ redemptions
└─ tier_configurations

-- Referral Service schema
referral_schema
├─ referral_links
├─ referrals (who referred whom)
├─ referral_rewards
└─ referral_campaigns
```

---

### 5. Support Database (Shared)

**Services**: Limits Service, Subscription Service, Notification Service, Audit Service, Scheduler Service

**Why shared**:
- Lower volume services
- Supporting infrastructure
- Schema-per-service isolation

```sql
support_db (PostgreSQL)

-- Limits Service schema
limits_schema
├─ user_limits
└─ limit_configurations

-- Subscription Service schema
subscription_schema
├─ subscriptions
├─ subscription_plans
└─ billing_history

-- Notification Service schema
notification_schema
├─ notification_templates
├─ notification_log
└─ user_preferences

-- Audit Service schema
audit_schema
└─ audit_events (partitioned by month)

-- Scheduler Service schema
scheduler_schema
├─ scheduled_jobs
└─ job_executions
```

---

### 6. Auth Database (Isolated) 🔐 — DECIDED

**Service**: Auth Service

**Decision**: separate `auth_db`. Credentials and MFA secrets warrant their own blast radius,
credentials, and backup/retention policy. It is also the one database whose contents differ
structurally between the two Auth adapters (local credential store vs. federated OIDC — ADR-005
Category 4), so coupling it to support services would have been wrong on two counts.

Single-tenant deployment means no `tenant_id`: every row belongs to the one customer owning the
cluster.

```sql
auth_db (PostgreSQL)
├─ users
├─ credentials
├─ mfa_configs
├─ permissions
└─ roles

redis_cluster (sessions)
└─ session:* (shared namespace)
```

---

### 7. Data Warehouse (Analytics) — CONDITIONAL

**Service**: Reporting ETL Service

**Purpose**: Read-optimized analytics database

```sql
datawarehouse_db (PostgreSQL → ClickHouse in Phase 2)
├─ transaction_analytics (denormalized)
├─ user_summaries
├─ daily_metrics
├─ rewards_analytics
├─ merchant_analytics
└─ fraud_analytics
```

**Populated by**: Kafka events → ETL service → Data warehouse

---

## Summary Table

| Database | Services | Type | Volume | Critical |
|----------|----------|------|--------|----------|
| **vaullet_db** | Vaullet | PostgreSQL | High | ⭐⭐⭐ Critical |
| **transactions_db** | Transaction, Refund | PostgreSQL | Very High | ⭐⭐⭐ Critical |
| **fraud_db** | Fraud Detection, Risk Mgmt | PostgreSQL + ES | Medium | ⭐⭐ High |
| **rewards_db** | Bonus, Loyalty, Referral | PostgreSQL | Medium | ⭐ Medium |
| **support_db** | Limits, Subscription, Notification, Audit, Scheduler | PostgreSQL | Low-Medium | ⭐ Medium |
| **auth_db** | Auth | PostgreSQL | Medium | ⭐⭐ High |
| **datawarehouse_db** | Reporting ETL | PostgreSQL/ClickHouse | High (read) | - (Rebuildable) |
| **redis_cluster** | All (cache, locks, sessions) | Redis | High | ⭐⭐ High |

**Total (full deployment)**: 6 operational PostgreSQL + 1 analytics PostgreSQL + 1 Elasticsearch
+ 1 Redis = **9 infrastructure components**

**Total (core-only contract)**: 5 PostgreSQL + 1 Elasticsearch + 1 Redis = **7 components**

vs. database-per-service: 15 PostgreSQL + 1 ES + 1 Redis + 1 DW = **18 components**

**Savings**: 9 fewer databases to operate while keeping isolation where it matters. This compounds
under single-tenant deployment — the saving is per customer cluster, not once.

---

## Service-to-Database Mapping

```
Vaullet Service           → vaullet_db
Transaction Service       → transactions_db
Refund (in Transaction Svc) → transactions_db (same deployment, not a separate service)
Fraud Detection Service   → fraud_db
Risk Management Service   → fraud_db
Bonus Service             → rewards_db (bonus_schema)
Loyalty Service           → rewards_db (loyalty_schema)
Referral Service          → rewards_db (referral_schema)
Limits Service            → support_db (limits_schema)
Subscription Service      → support_db (subscription_schema)
Notification Service      → support_db (notification_schema)
Audit Service             → support_db (audit_schema)
Scheduler Service         → support_db (scheduler_schema)
Auth Service              → auth_db
Reporting ETL Service     → datawarehouse_db
Admin UI                  → No database (aggregates via APIs)
```

The two that are deployed, `auth_db` and `vaullet_db`, are drawn table by table in [`docs/schema/README.md`](schema/README.md).

---

## Access Control

Each service gets dedicated database credentials with restricted access:

```sql
-- Example: Bonus Service can only access bonus_schema in rewards_db
CREATE USER bonus_service_user WITH PASSWORD 'secure_password';
GRANT USAGE ON SCHEMA bonus_schema TO bonus_service_user;
GRANT ALL ON ALL TABLES IN SCHEMA bonus_schema TO bonus_service_user;
REVOKE ALL ON SCHEMA loyalty_schema FROM bonus_service_user;
REVOKE ALL ON SCHEMA referral_schema FROM bonus_service_user;
```

---

## Evolution Path

### If a service outgrows shared database:

**Example: Loyalty Service becomes high volume**

1. Create dedicated `loyalty_db`
2. Export `loyalty_schema` from `rewards_db`
3. Import to `loyalty_db`
4. Update Loyalty Service config
5. Drop `loyalty_schema` from `rewards_db`

**Timeline**: Can be done in hours with minimal downtime

---

## Finalized Decisions

All three open questions are resolved:

1. **Refund Service** → **part of Transaction Service.** Same database, same deployment. Splitting it
   would have put two services on one database with no independent scaling or release benefit.

2. **Auth Database** → **separate `auth_db`.** Security isolation, and the adapter difference
   described in §6.

3. **Risk Management Service** → **shares `fraud_db`.** Same domain, same data, and Risk Management
   is a sellable module whose only dependency (Fraud Detection) is core — so the dependency is always
   satisfied and the ADR-005 star invariant holds.

This gives **6 operational databases** plus `datawarehouse_db`.

## Open Items

- Retention and archival policy per database (`vaullet_db` needs 7 years; `datawarehouse_db` is
  rebuildable from events and needs almost none)
- Whether Elasticsearch can degrade to Postgres-only for small deployments, making it a Category 4
  adapter rather than core-by-inheritance (flagged in ADR-005)

## References

- [ADR-003: Hybrid Database Strategy](adr/003-hybrid-database-strategy-with-analytics.md)
- [ADR-004: Atomic Balance Reservations](adr/004-atomic-balance-reservations.md) — `vaullet_db` schema
- [ADR-005: Module Composition](adr/005-module-composition-and-deployment-topology.md) — conditional provisioning
