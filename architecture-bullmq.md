# Enhancement: BullMQ for scheduled campaigns & journey step execution

This document describes how we intend to integrate **BullMQ** (on **Redis**) into GammaEngage for:

1. **Scheduled campaigns** — replace in-process `node-cron` timers and `setTimeout` in **`SchedulerService`** with repeatable / delayed jobs.
2. **Journeys** — replace the **minute-wide** `JourneyExecutorService` cron that scans **`journey_enrollments`** with **per-enrollment** (or per-step) **`journey-advance`** jobs so each **step** runs when due, with retries and horizontal workers.

**Status:** enhancement / target architecture.

| Area | Code today (reference) |
|------|-------------------------|
| Scheduled campaigns | `cdp-app/src/services/campaign-engine/src/scheduler/scheduler.service.ts` — `reloadAllCronJobs`, `registerCronJob`, `scheduleOneShot`, `executeSchedule` |
| Journey steps | `cdp-app/src/services/campaign-engine/src/journey/journey-executor.service.ts` — `@Cron(EVERY_MINUTE)` → `JourneyService.processDueEnrollments` → `processEnrollment` |
| Journey enrollment | `JourneyService.enrollFromEvent` (e.g. from `EventConsumerService`) |

BullMQ wiring details: `docs/journey-scheduled-campaign-architecture-changes.md`, `docs/background-jobs-bullmq.md`.

---

## Why this change

| Today | With BullMQ |
|--------|-------------|
| Each API replica can run the same cron unless operations constrain replicas | **Repeat schedulers** and **job instances** are coordinated via **Redis**; workers scale horizontally |
| Timers are **lost on restart** until `reloadAllCronJobs` runs; no built-in retries | Jobs **persist** until completed/failed; **retries**, backoff, and stall handling are first-class |
| Long `executeSchedule` (segment fetch + AMQP) runs inside a timer callback | Thin **@Process** handler can **fan out** to dedicated queues for heavy or parallel work |
| Journey **`next_step_at`** only checked **once per minute**; `isRunning` skips overlapping ticks | **Delayed `add()`** aligns wake-up with `next_step_at`; no global “whole executor” lock for all tenants |

---

## Mental model

1. **Source of truth** — Tenant Postgres **`scheduled_campaigns`** (`cron_expr`, `run_at`, `is_active`, …) remains authoritative for *what* should run.
2. **Registration** — On create/update/activate/deactivate (and on deploy **reconcile**), the app or a **CLI / deploy script** updates BullMQ so Redis reflects the DB:
   - **Recurring:** stable job key per schedule, **repeat** pattern from `cron_expr` (BullMQ 5+ **job scheduler** APIs such as **`upsertJobScheduler`** on the campaign cron queue).
   - **One-shot:** delayed **`add()`** with `delay = run_at - now` and a stable **`jobId`** (e.g. `sched-once:{id}`), not a repeat scheduler.
3. **Execution** — A **worker** bound to the **cron / scheduler queue** runs the **`@Process`** handler, which loads the schedule, marks state, and either runs dispatch inline or **`add()`**s work to **other queues** (e.g. **`scheduled-campaign-run`**, **`journey-advance`**, workflow-style queues) for backpressure and observability.

### Journeys (step execution)

1. **Source of truth** — **`journey_enrollments`** (`current_step`, `next_step_at`, `status`), **`journey_steps`** (`step_type`: send / wait / condition, `delay_hours`, …).
2. **Wake-ups in Redis** — When an enrollment is created or a **send** / **condition** step finishes, compute the next eligible time. Instead of relying only on the next **cron** scan, **`Queue.add('journey-advance', { brand_id, enrollment_id }, { delay, jobId })`** (stable **`jobId`** per “next fire”, e.g. `journey:{enrollment_id}:{step}`) so BullMQ delivers the job when **`next_step_at`** is reached.
3. **Handler** — One **`@Process`** invocation runs **at most one step** for that enrollment: load enrollment + step definition, execute **send** (templates + AMQP to `ge.campaigns.outbound`), **wait** (persist `next_step_at`, enqueue delayed follow-up), or **condition** (branch / exit). On success, if the journey continues, **`add()`** the next **`journey-advance`** with the appropriate **`delay`** (or immediate if the next step is due now).
4. **Optional thin cron** — A low-frequency **reconcile** job (e.g. daily or every 15 minutes) can scan for **orphan** enrollments (Redis lost, missed delay) and re-`add()` — same idea as DB↔BullMQ sync for schedules.

---

## BullMQ: Job Scheduler vs Queue

Both are needed, but for different responsibilities:

- **Job Scheduler** (`upsertJobScheduler`, BullMQ 5+) is for **time pattern registration** (e.g. cron-like `cron_expr`). It stores scheduler metadata in Redis and creates new job instances at each tick.
- **Queue jobs** (`queue.add`) are for **actual executable work**. They can be immediate or delayed (`delay`), retried (`attempts` / backoff), and consumed by workers.

Recommended usage in GammaEngage:

| Need | BullMQ primitive | Example |
|------|------------------|---------|
| Repeating schedule definition | `upsertJobScheduler` on scheduler queue | Scheduled campaign with `cron_expr` |
| One-shot future execution | `queue.add` with `delay` and stable `jobId` | `run_at` campaign send |
| Journey step wake-up | `queue.add` with `delay` from `next_step_at` | `journey-advance` per enrollment |
| Heavy fan-out / parallel work | `queue.add` to worker queues | batch dispatch, workflow queue |

Practical rule:

- Use **Job Scheduler** to answer: *“When should jobs be created repeatedly?”*
- Use **Queue jobs** to answer: *“What unit of work should a worker execute now/later?”*

---

## Shared `bullmqProvider` (cross-service)

To avoid each feature creating its own Redis/queue wiring, define a shared BullMQ provider module used by campaign scheduler and journeys.

### What should be shared

- **Single Redis connection factory** (host, TLS, db, prefix, retries).
- **Queue factory / registry** for known queues (`campaign-cron`, `scheduled-campaign-run`, `journey-advance`, optional workflow queues).
- **Common default job options** (`removeOnComplete`, `attempts`, backoff, `stackTraceLimit`).
- **Helper APIs** for scheduler upsert/remove and delayed add with deterministic IDs.
- **Lifecycle hooks** to close queues/workers cleanly on shutdown.

### Suggested shape

- `cdp-app/src/shared/bullmq/bullmq.module.ts`
- `cdp-app/src/shared/bullmq/bullmq.provider.ts`
- `cdp-app/src/shared/bullmq/bullmq.types.ts`

### Provider contract (example)

```ts
export interface BullmqProvider {
  queues: {
    campaignCron: Queue;
    scheduledCampaignRun: Queue;
    journeyAdvance: Queue;
  };
  upsertRecurring(input: {
    queue: Queue;
    schedulerId: string;
    pattern: string;
    payload: Record<string, unknown>;
  }): Promise<void>;
  addDelayed(input: {
    queue: Queue;
    name: string;
    jobId: string;
    delayMs: number;
    payload: Record<string, unknown>;
  }): Promise<void>;
  removeRecurring(input: { queue: Queue; schedulerId: string }): Promise<void>;
}
```

### How features consume it

- **`SchedulerService`** injects shared provider, then:
  - recurring schedule → `upsertRecurring(...)`
  - one-shot `run_at` → `addDelayed(...)`
- **`JourneyService` / journey worker** injects the same provider:
  - first enrollment wake-up → `addDelayed(...)`
  - post-step continuation → `addDelayed(...)` for next step

This keeps all BullMQ operational behavior (Redis options, retries, prefixes, queue names, shutdown) centralized and consistent across features.

---

## Sequence: scheduler registration → each tick → handler → optional fan-out

The flow below matches BullMQ’s model: something **registers** a repeatable scheduler on a queue; **Redis** stores scheduler metadata; on each match BullMQ **materializes a new job**; a **worker** runs the **Nest `@Process` (or equivalent) handler**, which may enqueue downstream jobs.

```mermaid
sequenceDiagram
  participant CLI as CLI / deploy script
  participant CQ as BullMQ cron-queue
  participant Redis as Redis
  participant W as Worker (cron-queue)
  participant H as Nest processor handler
  participant OQ as Other queues (e.g. workflow-queue)

  CLI->>CQ: upsertJobScheduler (repeat pattern)
  CQ->>Redis: persist scheduler + metadata
  loop Each tick
    Redis->>CQ: new job instance
    CQ->>W: deliver job
    W->>H: run handler
    H->>OQ: optional add() fan-out work
  end
```

**Mapping to scheduled campaigns**

- **CLI** — Admin API or bootstrap / **reconcile** job after deploy: sync DB rows to BullMQ (not only a literal shell script).
- **cron-queue** — A dedicated queue used for **repeat / scheduler** traffic (name TBD; e.g. align with `scheduled-campaign-run` or split a thin **`scheduled-campaign-tick`** from a heavier execution queue).
- **Handler** — Invokes the same business logic as today’s **`executeSchedule`** (load `ScheduledCampaignEntity` + campaign, segment resolution, RabbitMQ publish to `ge.campaigns.outbound`), with **idempotency** per fire window where needed.
- **Other queues** — Optional **`add()`** fan-out: per-tenant batches, journey steps, or workflow jobs so the cron handler stays small and failures are isolated.

---

## Flow: journey step execution (target)

High-level control flow from **enrollment** through **queued steps** to **completion** or **exit**. Solid lines are the BullMQ-backed path; the dashed line is an optional safety **reconcile** tick.

```mermaid
flowchart TB
  subgraph Entry
    EV[Event / API]
    EN[JourneyService enroll / create enrollment]
  end

  subgraph Persistence
    DB[(Tenant Postgres journey_enrollments journey_steps)]
  end

  subgraph BullMQ
    JQ[journey-advance queue]
    RD[(Redis)]
    WK[Worker]
    PR["Processor handler (one step)"]
  end

  subgraph outbound["Side effects"]
    MQ["RabbitMQ ge.campaigns.outbound"]
  end

  EV --> EN
  EN --> DB
  EN -->|add job payload brand_id enrollment_id| JQ
  JQ --> RD
  RD -->|due| WK
  WK --> PR
  PR --> DB
  PR -->|send step| MQ
  PR -->|more steps: add with delay or immediate| JQ
  PR -->|completed or exited| DB

  RC[Optional reconcile cron] -.->|re-add if drift| JQ
  RC -.-> DB
```

---

## Sequence: journey advance job (per step)

Each **job** represents “run the current step for this enrollment once”. After a **wait** step, the handler updates **`next_step_at`** in Postgres and enqueues the **next** job with **`delay`**. Retries apply to transient failures (AMQP, HTTP) without blocking other enrollments.

```mermaid
sequenceDiagram
  participant Src as Event / Admin API
  participant JS as JourneyService
  participant DB as Tenant Postgres
  participant JQ as journey-advance queue
  participant Redis as Redis
  participant W as Worker journey-advance
  participant H as Nest processor handler
  participant AMQP as RabbitMQ outbound

  Src->>JS: enrollFromEvent or manual enroll
  JS->>DB: INSERT journey_enrollments set next_step_at
  JS->>JQ: add( brand_id enrollment_id delay jobId )

  JQ->>Redis: persist job + delay
  Redis->>JQ: deliver when due
  JQ->>W: job instance
  W->>H: run single step

  H->>DB: load enrollment + journey_steps row
  alt send step
    H->>AMQP: publish outbound templates
    H->>DB: advance current_step set next_step_at
  else wait step
    H->>DB: set next_step_at from delay_hours
  else condition step
    H->>DB: branch or exit enrollment
  end

  opt Journey still active
    H->>JQ: add( next journey-advance delay or 0 )
  end
```

**Idempotency note:** use a deterministic **`jobId`** (or DB claim row) so a duplicate **`add()`** after retry does not double-send the same step; document the chosen keying in runbooks (see `docs/journey-scheduled-campaign-architecture-changes.md` §4.4 pattern for schedules).

---

## Related documentation

- `docs/journey-scheduled-campaign-architecture-changes.md` — DB → BullMQ mapping table, queue names, feature flags.
- `docs/background-jobs-bullmq.md` — Rationale vs Nest `@Cron`.
- `docs/reliable-background-work-evolution.md` — Broader “scheduler vs executor” pattern.
