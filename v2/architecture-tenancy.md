# Tenancy architecture (CDP app, platform DB, tenant DBs, AI engine)

This document explains how **brand / tenant** context flows through the system: the **modular monolith (`cdp-app`)**, the **platform Postgres** registry, **tenant Postgres** (dedicated DB per tenant or a shared database with `brand_id` on rows), **ClickHouse** (event analytics, keyed by `brand_id`), and the **AI engine** (Python service that scores players using **`brand_id`** in API and queries).

---

## 1. Big picture

```mermaid
flowchart TB
  subgraph clients["Clients & workers"]
    AdminUI["Admin UI"]
    APIs["API clients / X-API-Key"]
    Rabbit["RabbitMQ consumers"]
  end

  subgraph cdp["cdp-app — NestJS modular monolith"]
    direction TB
    AA["Admin Auth"]
    TA["Tenant Admin"]
    EI["Event Ingestion"]
    GC["Game Catalog"]
    CE["Campaign Engine"]
    CD["Channel Delivery"]
    IE["Identity Engine"]
    LE["Lifecycle Engine"]
    PP["Player Profile"]
    AN["Analytics"]
    DC["Data Connector"]
    TDS["TenantDataSourceService<br/>(global)"]
  end

  subgraph platform["Platform Postgres — connection name: PLATFORM_CONNECTION"]
    TenantsTable["tenants<br/>(brand_id, database_name, …)"]
    AdminUsers["admin users, roles,<br/>API keys (platform)"]
  end

  subgraph tenantA["Tenant data — Brand A"]
    PGA[("Postgres<br/>brand_A_db or shared + brand_id")]
  end

  subgraph tenantB["Tenant data — Brand B"]
    PGB[("Postgres<br/>brand_B_db or shared + brand_id")]
  end

  subgraph ch["ClickHouse — shared cluster"]
    CHDB[("Database e.g. gammaengage<br/>events_raw …<br/>filter: brand_id")]
  end

  subgraph ai["AI engine — Python (separate process)"]
    AISvc["Score / train APIs"]
  end

  clients --> cdp
  TDS --> TenantsTable
  TDS --> PGA
  TDS --> PGB
  CE -.->|"brand_id"| TDS
  CD -.->|"brand_id"| TDS
  IE -.->|"brand_id"| TDS
  PP -.->|"brand_id"| TDS
  GC -.->|"brand_id"| TDS
  LE -.->|"brand_id"| TDS
  AN -.->|"brand_id"| TDS
  EI -.->|"brand_id"| TDS
  DC -.->|"brand_id"| TDS
  AA --> AdminUsers
  TA --> AdminUsers
  TA --> TenantsTable
  CD -.->|"brand_id in payload"| CHDB
  CE -.->|"brand_id in payload"| CHDB
  PP -.->|"brand_id in payload"| CHDB
  AN -.->|"brand_id in payload"| CHDB
  AISvc -->|"POST … brand_id + player_id"| CHDB
  AISvc -->|"SQL WHERE brand_id = …"| PGA
  AISvc -->|"SQL WHERE brand_id = …"| PGB
```

**Legend**

| Piece | Role |
|--------|------|
| **TenantDataSourceService** | Looks up `tenants` by **`brand_id`**, opens the matching **Postgres** (`database_name`) or **shared fallback** DB (`TENANT_FALLBACK_DATABASE` / `DB_NAME`) where all tables share **`brand_id`** columns. |
| **Platform Postgres** | **Not** tenant business data: registry of tenants, admin-auth tables, etc. |
| **AI engine** | Not inside `cdp-app`. Uses **`brand_id`** on each request and in SQL/CH queries to isolate data logically (single DSN + `brand_id` filters is typical; dedicated DBs per brand depend on deployment). |

---

## 2. `cdp-app` feature modules (all in one process)

These are the **feature modules** imported by `AppModule` (see `cdp-app/src/app.module.ts`):

```mermaid
flowchart LR
  subgraph cdp_app["cdp-app"]
    M1["AdminAuthFeatureModule"]
    M2["TenantAdminFeatureModule"]
    M3["EventIngestionFeatureModule"]
    M4["GameCatalogFeatureModule"]
    M5["CampaignEngineFeatureModule"]
    M6["ChannelDeliveryFeatureModule"]
    M7["IdentityEngineFeatureModule"]
    M8["LifecycleEngineFeatureModule"]
    M9["PlayerProfileFeatureModule"]
    M10["AnalyticsFeatureModule"]
    M11["DataConnectorFeatureModule"]
  end
```

Shared infrastructure inside the same app:

- **TypeORM default connection** — legacy/shared tenant schema on `TENANT_FALLBACK_DATABASE` (entities listed in `TENANT_ENTITIES`).
- **`PLATFORM_CONNECTION`** — second TypeORM connection to platform DB (`PLATFORM_ENTITIES`).
- **`TenantDatabaseModule`** — provides **`TenantDataSourceService`** for **tenant-scoped** Postgres access by **`brand_id`**.

---

## 3. Who uses “tenant DB” routing (`brand_id`)?

Conceptually, services that persist or read **per-brand business data** resolve a **tenant `DataSource`** (or repository) using **`brand_id`** — either from the HTTP body/query, API key metadata, or message envelope (e.g. RabbitMQ).

```mermaid
flowchart TB
  TDS["TenantDataSourceService"]

  TDS --> R1["getDataSourceForBrand(brand_id)"]
  TDS --> R2["getRepository(Entity, brand_id)"]

  subgraph examples["Examples of tenant-scoped usage in cdp-app"]
    E1["Campaign / Channel / Identity /<br/>Player Profile / Game Catalog /<br/>Lifecycle / Analytics / …"]
  end

  R2 -.-> E1
```

**Admin Auth** and parts of **Tenant Admin** primarily use the **platform** connection (users, tenant registry), not the per-tenant data store for gameplay data.

---

## 4. Multiple tenants (illustration)

```mermaid
flowchart LR
  subgraph platform["Platform DB"]
    T[("tenants")]
  end

  T -->|brand_id = casino_a<br/>database_name = pg_casino_a| DB_A[("Postgres: pg_casino_a")]
  T -->|brand_id = casino_b<br/>database_name = pg_casino_b| DB_B[("Postgres: pg_casino_b")]
  T -->|brand_id = demo<br/>database_name = null → fallback| DB_S[("Postgres: shared<br/>rows keyed by brand_id")]

  subgraph brands["brand_id in requests / events"]
    BA["casino_a"]
    BB["casino_b"]
    BD["demo"]
  end

  BA --> DB_A
  BB --> DB_B
  BD --> DB_S
```

---

## 5. AI engine vs `cdp-app`

| | **cdp-app** | **AI engine** (Python) |
|---|-------------|-------------------------|
| **Deployment** | Single NestJS app, multiple feature modules | Separate HTTP service (e.g. Docker `ai-engine`) |
| **Postgres** | **`TenantDataSourceService`** + platform TypeORM | **`POSTGRES_DSN`**; queries use **`brand_id`** in SQL |
| **ClickHouse** | Various services insert/query with **`brand_id`** | Single CH database; **`brand_id`** in tables / filters |
| **Isolation** | Physical DB per tenant **or** shared DB + **`brand_id`** | Logical isolation by **`brand_id`** in queries |

---

## 6. Related code paths

- Tenant entity list: `cdp-app/src/database/tenant-entities.ts`
- Platform entity list: `cdp-app/src/database/platform-entities.ts`
- Tenant routing: `cdp-app/src/database/tenant-data-source.service.ts`
- Monolith rule: `.cursor/rules/cdp-app-monolith.mdc`

---

*Diagrams are descriptive; exact deployment (dedicated DB vs shared fallback) depends on `tenants.database_name` and environment variables.*
