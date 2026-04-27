# Workflow engine: schedules, campaigns, journeys, webhooks

Conceptual flow diagrams for a **workflow engine** that coordinates **many scheduled campaigns** (cron or one-shot), a **repeatable campaign queue**, **journey steps**, and **outbound webhooks**. This aligns with ideas in **cdp-app** (`scheduled_campaigns`, `cron_expr`, journey enrollments / steps) and with a **BullMQ-style** orchestration layer described in [background-jobs-bullmq.md](./background-jobs-bullmq.md).

These diagrams are **target architecture**; wiring may use BullMQ flows, separate queues, or a mix.

---

## 1. End-to-end engine (schedules → campaigns → channels & webhooks)

High-level control plane: one **orchestrator** tick (or repeatable Redis job) fans out work without duplicating cron on every API replica.

```mermaid
flowchart TB
  subgraph inputs["Definitions (per brand_id)"]
    SC1["Scheduled campaign #1<br/>cron_expr / run_at"]
    SC2["Scheduled campaign #2"]
    SCN["Scheduled campaign #N"]
    JV["Journeys + journey_steps<br/>(send / wait / condition)"]
  end

  subgraph engine["Workflow engine"]
    ORCH["Orchestrator tick<br/>(single leader or BullMQ repeat)"]
    RQ[("Repeatable campaign queue<br/>Redis / BullMQ")]
    ORCH --> RQ
    RQ --> DUE{"What is due?"}
  end

  subgraph schedule_path["Scheduled campaigns"]
    DUE -->|"next fire for cron"| SCH_JOB["Job: executeSchedule<br/>brand_id + schedule_id"]
    SCH_JOB --> DISPATCH["Segment resolve +<br/>dispatch to players"]
  end

  subgraph journey_path["Journeys"]
    DUE -->|"next_step_at elapsed"| JR_JOB["Job: advance enrollment<br/>brand_id + enrollment_id"]
    JR_JOB --> STEP_EXEC["Execute current step"]
  end

  subgraph delivery["Outbound"]
    DISPATCH --> PUB["Campaign publisher<br/>(e.g. AMQP → channel delivery)"]
    STEP_EXEC -->|"step_type = send"| PUB
    PUB --> CH["Channels<br/>email / sms / push / …"]
    PUB --> WHQ[("Webhook out queue")]
    WHQ --> WH["HTTP webhooks<br/>delivery / analytics callbacks"]
  end

  subgraph journey_control["Journey step logic"]
    STEP_EXEC -->|"step_type = wait"| DELAY["Enqueue delayed job<br/>(delay_hours)"]
    DELAY --> RQ
    STEP_EXEC -->|"step_type = condition"| BR{"Condition met?"}
    BR -->|"yes"| NEXT["Advance step_order"]
    BR -->|"no"| EXIT["Mark enrollment exited"]
    NEXT --> RQ
  end

  SC1 & SC2 & SCN -.->|"persisted config"| DUE
  JV -.->|"enrollments + steps"| STEP_EXEC
```

**Reading the diagram**

- **Many schedules** map to **many repeatable jobs** (one job key per `schedule_id`, or one scanner job that loads all due rows from Postgres).
- **Journey** work is either the same tick scanning **`journey_enrollments`** (`status`, `next_step_at`) or **per-enrollment delayed jobs** after a **wait** step (cleaner than polling only on a minute cron).
- **Webhooks** are modeled as a **dedicated queue** so HTTP retries, backoff, and dead-lettering do not block campaign send or journey advancement.

---

## 2. Repeatable campaign queue vs one-shot

```mermaid
flowchart LR
  subgraph definitions
    CRON["cron_expr set<br/>(e.g. 0 9 * * 1)"]
    ONCE["run_at only<br/>(one-shot)"]
  end

  subgraph repeatable_campaign_queue
    REP["Repeatable job<br/>pattern = cron_expr"]
    ONESHOT["Delayed job<br/>run_at - now"]
  end

  CRON --> REP
  ONCE --> ONESHOT

  REP --> EX["executeSchedule"]
  ONESHOT --> EX
  EX -->|"cron: reset status pending"| REP
  EX -->|"one-shot"| DONE["status completed"]
```

This mirrors **`ScheduledCampaignEntity`**: `cron_expr` takes precedence over `run_at`; after a successful cron run, schedule stays **pending** for the next occurrence.

---

## 3. Journey: steps as a small state machine

```mermaid
stateDiagram-v2
  [*] --> DueCheck: enrollment active

  DueCheck --> SendStep: step_type = send
  DueCheck --> WaitStep: step_type = wait
  DueCheck --> CondStep: step_type = condition

  SendStep --> Publish: publish campaign / channels
  Publish --> WebhookNotify: optional webhook job
  WebhookNotify --> Advance

  WaitStep --> ScheduleWait: enqueue delayed job delay_hours
  ScheduleWait --> Advance

  CondStep --> Evaluate: compare condition_field
  Evaluate --> Advance: condition met
  Evaluate --> Exited: condition not met

  Advance --> Completed: no next step
  Advance --> DueCheck: more steps, update next_step_at

  Exited --> [*]
  Completed --> [*]
```

---

## 4. Optional: sequence — schedule fire to webhook

```mermaid
sequenceDiagram
  participant R as Repeatable queue (Redis)
  participant W as Worker / campaign-engine
  participant DB as Tenant Postgres
  participant Q as Message bus (AMQP)
  participant CD as Channel delivery
  participant WH as Webhook worker

  R->>W: job(schedule_id, brand_id)
  W->>DB: load scheduled_campaigns + campaign
  W->>DB: resolve segment → player list / batches
  loop batches
    W->>Q: publish send commands
    Q->>CD: deliver email/sms/…
    W->>R: enqueue webhook_out(delivery_summary)
  end
  R->>WH: job(webhook URL, payload, attempt)
  WH-->>WH: HTTP POST + retry/backoff
```

---

## 5. Implementation notes (short)

- **Idempotency** — Use stable **job ids** (`schedule_id` + fire window, or `enrollment_id` + `step_order`) so duplicates after deploy do not double-send.
- **Fan-out** — BullMQ **FlowProducer**: parent “run schedule” → children “batch 1…k” → optional “aggregate + webhook” child (see BullMQ flows docs).
- **Separation** — Keep **orchestration** (what runs when) in the workflow engine; keep **content** (templates, segments) in tenant DB; keep **HTTP webhooks** on an isolated queue with strict timeouts.

---

*See also: [background-jobs-bullmq.md](./background-jobs-bullmq.md), campaign-engine `SchedulerService`, `JourneyExecutorService`, `JourneyStepEntity`.*
