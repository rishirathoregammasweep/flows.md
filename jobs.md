# Background jobs with BullMQ (instead of NestJS Cron)

This document describes how **cdp-app** could move recurring work from **`@nestjs/schedule`** (`ScheduleModule`, `@Cron`) to **BullMQ**, so you get **durable queues**, **retries**, **horizontal workers**, and **workflow-style job graphs** on top of **Redis** (the stack already uses **ioredis** for throttling and caching).

It is a design and migration guide, not an implemented change.

---

## 1. What you have today

Across **cdp-app**, several modules register **`ScheduleModule`** and run work on **`@Cron`** decorators, for example:

- **Campaign engine** — journey execution tick (`EVERY_MINUTE`)
- **Channel delivery** — delivery retry sweep (`EVERY_MINUTE`)
- **Lifecycle engine** — nightly-style windows
- **Analytics** — RFM (`EVERY_DAY_AT_2AM`)
- **Data connector** — S3 / SFTP pollers (env-driven cron expressions)
- **Tenant admin** — DLQ alerts (`EVERY_MINUTE`)

Those jobs run **inside whichever Nest process(es) are up**. They are **not** centrally deduplicated: with **multiple replicas**, **every instance** runs the same cron unless you add distributed locks or leader election yourself.

---

## 2. Why BullMQ instead of in-process cron

| Concern | Nest `@Cron` | BullMQ |
|--------|----------------|--------|
| **Multi-instance** | Each pod fires; needs extra locking | **One scheduler** (or Redis-coordinated repeat) drives work; **workers** scale out safely |
| **Persistence** | Lost if process dies mid-run | Jobs live in **Redis** until completed / failed |
| **Retries / backoff** | Manual | Built-in **attempts**, **exponential backoff**, **stall** handling |
| **Visibility** | Logs only | **Bull Board** / metrics on queue depth, failures, latency |
| **Delay & workflows** | Awkward | **Delayed jobs**, **repeatable jobs**, **flows** (parent → children) |

BullMQ is **not** a full BPMN engine, but it is enough for **“run this on a schedule”**, **“retry this unit of work”**, and **“step A then parallel B/C then D”** patterns.

---

## 3. Building blocks (mental model)

1. **Connection** — Redis URL / options (reuse the same Redis you trust for job data; consider a **logical DB index** or **key prefix** so job keys do not collide with cache keys).
2. **Queue** — named pipe of jobs (e.g. `campaign-journey-tick`, `delivery-retry`).
3. **Producer** — Nest service (or any script) that **`add()`**s jobs with optional **repeat**, **delay**, or **priority**.
4. **Worker** — process that **pulls** jobs and runs your handler. Can live **in the same Nest app** as the API (simple) or in **dedicated worker** deployments (recommended under load).
5. **Scheduler / repeatable jobs** — BullMQ can register **cron-like repeat** so Redis triggers the next enqueue; only **workers** execute handlers.

Optional: **QueueEvents** for metrics, alerting, or fan-out without blocking the worker.

---

## 4. Replacing `@Cron` with scheduled BullMQ jobs

**Pattern A — Repeatable job on a dedicated queue**

- Create a queue, e.g. `lifecycle-nightly`.
- On application bootstrap (or a one-off migration command), **register** a job with a **repeat rule** (cron pattern), same spirit as `@Cron('30 3 * * *')`.
- A **single worker** (or N workers for throughput) processes each firing; use **job id** or **deduplication** options if you must guarantee at-most-once side effects per window.

**Pattern B — External cron → enqueue**

- Keep **Kubernetes CronJob** / system cron that **HTTP POSTs** or runs **`node enqueue.js`** to add a one-off job. Useful when ops wants scheduling outside the app.

**Pattern C — “Tick” queue fed every minute**

- Replace `EVERY_MINUTE` crons with a **repeat every 60s** (or a short cron) on a **`system-tick`** queue; handlers stay small and enqueue **child jobs** per tenant or per batch (avoids one giant cron method).

Map existing env-driven crons (e.g. S3/SFTP poll intervals) to **repeat patterns** or **dynamic repeat** updated when config changes.

---

## 5. One-off and delayed jobs

Not everything is cron-shaped:

- **Send in 10 minutes** — `add` with **delay** (ms).
- **Retry this delivery in 5 minutes** — prefer a **delayed job** (or dedicated **retry queue** with backoff) over a tight `@Cron` loop that scans the database every minute.

This aligns well with **channel delivery retries** and similar patterns.

---

## 6. Workflows (multi-step, fan-out, join)

BullMQ supports **Flows** (**parent job** with **child jobs** on the same or different queues):

- **Fan-out** — parent job completes “planning”; **FlowProducer** adds children (e.g. one job per **brand_id** or per batch of players).
- **Fan-in** — parent can use **“wait for children”** semantics so a final step runs only when children finish (see BullMQ docs for the current **FlowProducer** / **parent dependency** API in your version).

**Simpler alternative** — **chain by enqueueing**: step-1 worker finishes and adds step-2 job with payload. Easier to reason about; less magic than flows.

**When to reach for something else** — Long-running **Saga** across days, human approvals, or complex compensation may fit **Temporal** / **Camunda** better; BullMQ fits **minutes-scale** background work inside your own Redis.

---

## 7. NestJS integration (sketch)

Typical packages:

- **`bullmq`** — core library.
- **`@nestjs/bullmq`** — registers queues and workers with Nest DI (check compatibility with your Nest **10.x**).

Suggested layout:

1. **`BullModule.forRootAsync`** — Redis connection from `ConfigService` (host, port, password, TLS, **prefix**).
2. **`BullModule.registerQueue({ name: 'delivery-retry' })`** per domain queue.
3. **`@Processor('delivery-retry')`** class with `@Process()` methods — replaces the body of today’s `@Cron` handler.
4. **Bootstrap** — register repeatable jobs once (guarded by env flag or **idempotent** `upsertJobScheduler` / remove-duplicates pattern depending on BullMQ version).
5. **Shutdown** — close workers and queues in **`onModuleDestroy`** so in-flight jobs are not cut off abruptly (BullMQ exposes **graceful** close options).

Keep **heavy** work off the HTTP event loop: the **worker** concurrency setting controls parallelism.

---

## 8. Operations and safety

- **Idempotency** — Scheduled ticks often overlap with restarts; design handlers so **running twice** for the same window is safe (unique keys, `ON CONFLICT`, or “lease” row in Postgres).
- **Redis memory** — Configure **TTL / removeOnComplete** / **removeOnFail** policies so finished jobs do not fill Redis.
- **Tenancy** — Pass **`brand_id`** (or tenant id) in **job data**; workers resolve `TenantDataSourceService` the same way HTTP handlers do.
- **Observability** — Structured logs with **job id**, **queue name**, **attempt**; optional **OpenTelemetry** + BullMQ instrumentation if you add it later.

---

## 9. Migration checklist (incremental)

1. Add **Redis-backed** BullMQ module to **cdp-app** (or split **worker** app later).
2. Pick **one** low-risk cron (e.g. a poller or a metrics sweep), implement **queue + worker**, run **repeat** only in **one** environment first.
3. Compare **duplication** and **missed runs** vs old `@Cron` under **multiple replicas**.
4. Turn off the old **`@Cron`** for that path behind a feature flag.
5. Repeat for **delivery retry**, **journey executor**, **DLQ alert**, **RFM**, etc., possibly **consolidating** several minute ticks into one scheduler queue that dispatches child jobs.

You can keep **`ScheduleModule`** for a while for stragglers; the goal is **no duplicate fire** across instances and **clear** job lifecycle semantics.

---

## 10. References

- [BullMQ — Guide](https://docs.bullmq.io/)
- [BullMQ — Repeatable / scheduled jobs](https://docs.bullmq.io/guide/jobs/repeatable)
- [BullMQ — Flows (parent / child)](https://docs.bullmq.io/guide/flows)
- Nest: **`@nestjs/bullmq`** module documentation for your Nest major version

---

*Last updated: internal design note for GammaEngage **cdp-app**.*
