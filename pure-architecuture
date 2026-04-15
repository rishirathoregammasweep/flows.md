# Services folder, `cdp-app`, and tenant database architecture

This document describes how the repository relates **historical microservice-style deployments** (the root **`services/`** folder) to the **current modular monolith (`cdp-app`)**, why the architecture was consolidated, and how **reads and writes** reach **tenant databases**. It complements [architecture-v2.md](./architecture-v2.md), which goes deeper on tenancy and `brand_id` routing.

---

## 1. What lives where

| Location | Role |
|----------|------|
| **`services/`** | **Multiple deployable services** (NestJS and others), each historically its **own codebase and often its **own database** boundary. They reflect a **microservice-oriented** split: isolated data, separate containers, and **more moving parts** to operate and debug (many Docker services, many connection strings). |
| **`cdp-app/`** | **Single NestJS application** — the **modular monolith** that **contains the feature modules** that used to be implemented as **separate services**. One process, **unified** logging, tracing, and local debugging; modules are still separated by folder under `cdp-app/src/services/<name>/`. |
| **AI engine** | **Still a separate service** (e.g. Python). It was never folded into `cdp-app`; it talks to the CDP over HTTP and uses **`brand_id`** (and shared analytics stores such as ClickHouse) like the rest of the platform. |
| **`db-migrations/`** | **Shared migrations** for platform vs tenant schemas — one place to evolve **one logical database model** as you move toward **multiple logical services backed by coordinated tenant data** (see below). |

> **Agent / contributor note:** Default place for **new features and fixes** is **`cdp-app/src/services/<service-name>/`**. The standalone **`services/`** tree may be **legacy or out of sync** with what is deployed unless a project explicitly maintains both. See `.cursor/rules/cdp-app-monolith.mdc`.

---

## 2. Why the architecture changed (from “many services + isolated DBs” to a unified app)

**Pain points of the earlier pattern**

- **Operational bottleneck:** Many small services meant **many containers**, **many databases**, and **harder end-to-end debugging** (trace a single user journey across several repos and connection pools).
- **Data isolation trade-off:** **Separate databases per service** isolated failures and schemas but duplicated **cross-cutting concerns** (tenant context, migrations, observability) and made **consistent tenant semantics** harder.
- **Delivery friction:** Shipping a coordinated change often required **multiple deploys** and **version skew** between services.

**What we moved toward**

- **`cdp-app` as one modular monolith** so former “microservices” become **NestJS modules** in one deployable unit: **fewer Docker services to debug**, **one** primary API surface for CDP features, **shared** middleware and tenant resolution.
- **Tenant data model** centered on **platform registry + tenant Postgres** (dedicated DB per tenant and/or shared DB with **`brand_id`**, depending on environment), so **every business read/write** for a brand goes through **tenant-scoped connections** (see [§4](#4-current-request-path-tenant-db-connections)).
- **Direction:** Teams may still think in **multiple logical services** (campaign engine, channel delivery, …) but implement them **inside `cdp-app`**; where the roadmap calls for **multiple processes again**, those can share **one database** (or one migration pipeline) to avoid recreating the **N-DB operational** burden — exact split is a product/ops decision.

---

## 3. Diagram: before — many services, isolated databases (conceptual)

This is a **simplified** view of the **older** pattern: **one repo folder per service**, each with its **own DB** (or schema), **more** network hops and **more** things to run locally.

```mermaid
flowchart TB
  subgraph clients["Clients"]
    UI["Admin UI / APIs"]
  end

  subgraph legacy["Historical: many `services/*` deployments"]
    S1["Service A<br/>(e.g. campaigns)"]
    S2["Service B<br/>(e.g. channels)"]
    S3["Service C<br/>(e.g. players)"]
  end

  subgraph dbs["Isolated data stores"]
    D1[("DB A")]
    D2[("DB B")]
    D3[("DB C")]
  end

  subgraph ai_old["AI engine"]
    AI["Separate process<br/>(unchanged role: scoring / ML)"]
  end

  UI --> S1
  UI --> S2
  UI --> S3
  S1 --> D1
  S2 --> D2
  S3 --> D3
  UI -.-> AI
  AI -.-> D1
```

**Takeaway:** Strong **isolation**, but **duplication of tenant plumbing** and **heavier** operations (many services + many DBs).

---

## 4. Current request path: tenant DB connections (`cdp-app`)

Today, **business data** for a brand is accessed through **tenant-scoped** Postgres (and optional shared fallback) via **`TenantDataSourceService`** and related wiring: **reads and writes** open or reuse connections **per tenant** resolved from **`brand_id`**. The **platform** database holds **tenant registry**, admin users, API keys — not the per-brand gameplay rows.

```mermaid
flowchart TB
  subgraph ingress["Ingress"]
    Client["HTTP / workers / queue consumers"]
  end

  subgraph cdp["cdp-app — single NestJS process, many modules"]
    M["Modules: campaign-engine, channel-delivery,<br/>tenant-admin, player-profile, …"]
    TDS["TenantDataSourceService<br/>getDataSourceForBrand(brand_id)"]
  end

  subgraph platform["Platform Postgres"]
    Reg[("tenants, admin users, …")]
  end

  subgraph tenant["Tenant Postgres (per deployment)"]
    TDB[("Business tables<br/>dedicated DB and/or shared + brand_id")]
  end

  subgraph ai["AI engine — separate service"]
    AISvc["Python APIs<br/>brand_id in requests/queries"]
  end

  Client --> M
  M --> TDS
  TDS --> Reg
  TDS --> TDB
  M -.->|"analytics / events"| CH[("ClickHouse etc.")]
  AISvc --> TDB
  AISvc --> CH
```

**Takeaway:** **One** primary CDP runtime (**`cdp-app`**) routes **tenant reads/writes** through **explicit tenant connections**; **AI** stays **outside** the monolith but obeys the same **`brand_id`** boundaries.

---

## 5. Diagram: `cdp-app` internal modules (formerly separate services)

The **service name** in **`cdp-app/src/services/<name>/`** is the **module boundary** that replaced a **separate repo or deployable** in `services/`.

```mermaid
flowchart LR
  subgraph monolith["cdp-app (single deployable)"]
    direction TB
    CE["campaign-engine"]
    CD["channel-delivery"]
    TA["tenant-admin"]
    PP["player-profile"]
    AN["analytics"]
    O["… other modules"]
  end

  subgraph note["Note"]
    N["Each block ≈ a former<br/>microservice responsibility"]
  end

  CE --- note
```

---

## 6. Target direction: “multiple services, one database” (roadmap framing)

When the org **splits processes again** (for scale or team ownership), a common goal is to avoid **N unrelated databases**: **one logical tenant model** and **shared migration discipline** (`db-migrations/`), with **each** new service using **the same tenant connection strategy** (or a read replica) instead of **re-isolating** data per service. Exact topology (single cluster vs sharded) is **deployment-specific**.

```mermaid
flowchart TB
  subgraph apps["Multiple future services (example)"]
    A1["cdp-app or split workers"]
    A2["Another API service"]
  end

  subgraph data["Shared data tier (conceptual)"]
    PG[("Postgres — tenant + platform schemas<br/>coordinated migrations")]
  end

  A1 --> PG
  A2 --> PG
```

**Takeaway:** **Logical** service boundaries can multiply; **physical** database sprawl should stay **controlled** so **tenant reads/writes** stay **consistent** and **debuggable**.

---

## 7. Related reading

- [architecture-v2.md](./architecture-v2.md) — tenancy, `brand_id`, ClickHouse, AI engine comparison table.
- [database-migrations.md](./database-migrations.md) — platform vs tenant migrations.
- `.cursor/rules/cdp-app-monolith.mdc` — where to implement features in this repo.

---

*Diagrams are illustrative; exact DB topology (dedicated tenant DB vs shared fallback) follows `tenants.database_name` and environment configuration.*
