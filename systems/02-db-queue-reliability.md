# Calibration 02: DB + Queue Reliability

This record examines a failure boundary between a PostgreSQL commit and a queue
publish.

- Initial test: 2026-08
- Retest: Not completed
- Domain: Distributed systems / reliability

## Scenario

An API inserts a job into PostgreSQL, publishes a queue message, and returns.
A worker consumes the message, performs deterministic work, and updates the job.

Production occasionally contains a job stuck in `pending`, but the queue has
no corresponding message. The database and queue cannot be replaced, and a
distributed transaction is not allowed.

## Constraints

The system must eventually repair a committed job whose message is absent.
Retries and duplicate delivery are possible. The worker may crash before or
after producing its business side effect.

The design must distinguish detection, repair, execution ownership, and
protection against duplicated side effects.

## My First Response

The following is the original first response.

> 第一個會懷疑發送行為有沒有發生 要證明發生在哪步驟
> 我應該要先能復現 之後關掉清queue的行為 確定queue真的有收到
> 往下就是檢查worker是不是真的有收到 收到後看到底有沒有反應

After the atomicity gap was pointed out, the next response proposed repair.

> 我的直覺是做一個job status check cronjob
> 沒完成的job總要在queue或是worker出現

When duplicate delivery was introduced, the original follow-up was:

> 我剛好最近經常看到exactly-once sementic
> 應該可以讓worker方收到任務後 根據job id判斷是否多送

## What I Got Right

The response did not immediately blame the worker. It tried to establish
evidence across the API, queue, and worker boundaries.

After the failure model became explicit, it independently moved from
"guarantee the first attempt" to a reconciliation loop that makes missing work
eventually reappear. Using the job ID as stable identity also pointed toward
deduplication and idempotent processing.

## What I Missed

The initial response treated reproduction as a prerequisite. Rare distributed
failures often need to be reconstructed from timestamps, publish
acknowledgements, message IDs, logs, and traces instead.

More importantly, it did not initially model the crash window between two
independent side effects:

1. PostgreSQL commits the job.
2. The process crashes or queue publish fails.
3. The queue never receives the message.

Reordering those steps only creates the opposite invalid state. The response
also conflated detecting the same job ID with exactly-once behavior. A normal
"check, then act" worker can race, and a crash after an external side effect but
before recording completion can still duplicate the effect.

## Updated Mental Model

A database commit and a queue publish are not one atomic transition. Without a
distributed transaction, the design must accept at-least-once attempts and make
the workflow converge.

Two common repair structures are:

- Store an outbox record in the same database transaction as the job, then
  publish and mark the outbox record separately.
- Periodically reconcile stale jobs and republish work whose execution is not
  progressing.

Both approaches require idempotent or deduplicated processing. Acquiring
execution rights must be atomic, such as a conditional state transition, rather
than a separate status read and update.

A processing state also needs recovery semantics such as a lease, timeout,
heartbeat, or reconciliation rule. For non-idempotent external effects, the
idempotency boundary must include the external system or use a durable business
key. Queue delivery alone cannot provide the business meaning of exactly once.

## Retest

Use a payment-settlement scenario in which PostgreSQL records a settlement
request and a worker calls an external payment provider. Introduce crashes
before publish, after publish, after charging, and before marking completion.

Without naming outbox or idempotency in the prompt, design the state machine,
repair loop, execution claim, observability evidence, and duplicate-side-effect
protection.
