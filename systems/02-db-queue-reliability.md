# Calibration 02：DB + Queue Reliability｜資料庫與佇列可靠性

這筆紀錄檢驗 PostgreSQL commit 與 queue publish 之間的 failure boundary。

This record examines a failure boundary between a PostgreSQL commit and a queue
publish.

- 初次測試｜Initial test: 2026-08
- 重測｜Retest: 尚未完成 / Not completed
- 領域｜Domain: Distributed systems / reliability

## 情境｜Scenario

API 先把 job 寫入 PostgreSQL，再發布 queue message。worker 收到訊息後
執行 deterministic task，最後更新 job 狀態。

Production 偶爾出現 job 永遠停在 pending，但 queue 沒有對應 message。
PostgreSQL 與 queue 都不能替換，也不能引入 distributed transaction。

An API inserts a job into PostgreSQL, publishes a queue message, and returns.
A worker consumes the message, performs deterministic work, and updates the job.

Production occasionally contains a job stuck in pending, but the queue has no
corresponding message. The database and queue cannot be replaced, and a
distributed transaction is not allowed.

## 限制｜Constraints

系統必須最終修復「DB 已 commit、message 卻不存在」的 job。retry 與
duplicate delivery 都可能發生，worker 也可能在 business side effect 前後
crash。

設計必須區分 detection、repair、execution ownership，以及如何避免重複
side effect。

The system must eventually repair a committed job whose message is absent.
Retries and duplicate delivery are possible. The worker may crash before or
after producing its business side effect.

The design must distinguish detection, repair, execution ownership, and
protection against duplicated side effects.

## 第一次回答｜My First Response

以下是原始第一次回答，沒有補上後來才理解的 atomicity gap。

The following is the original first response.

> 第一個會懷疑發送行為有沒有發生 要證明發生在哪步驟
> 我應該要先能復現 之後關掉清queue的行為 確定queue真的有收到
> 往下就是檢查worker是不是真的有收到 收到後看到底有沒有反應

當 atomicity gap 被指出後，下一個回答開始提出 repair loop。

After the atomicity gap was pointed out, the next response proposed repair.

> 我的直覺是做一個job status check cronjob
> 沒完成的job總要在queue或是worker出現

當情境加入 duplicate delivery 時，原始追答如下。

When duplicate delivery was introduced, the original follow-up was:

> 我剛好最近經常看到exactly-once sementic
> 應該可以讓worker方收到任務後 根據job id判斷是否多送

## 做對的部分｜What I Got Right

回答沒有立刻假設 worker 壞掉，而是嘗試沿著 API、queue、worker 建立
evidence，這個 debugging instinct 是合理的。

failure model 被明確指出後，思路能從「保證第一次就成功」轉向
reconciliation loop，讓消失的工作最終重新出現。用 job ID 當穩定 identity
也已經朝 deduplication 與 idempotent processing 的方向前進。

The response did not immediately blame the worker. It tried to establish
evidence across the API, queue, and worker boundaries.

After the failure model became explicit, it moved from guaranteeing the first
attempt to a reconciliation loop that makes missing work eventually reappear.
Using the job ID as stable identity also pointed toward deduplication and
idempotent processing.

## 遺漏的部分｜What I Missed

最初把「能復現」當成調查前提，但低頻 distributed failure 經常只能透過
timestamp、publish acknowledgement、message ID、log 與 trace 重建當次
timeline。

更核心的遺漏是：沒有第一時間把兩個獨立 side effects 中間的 crash window
視為設計本身允許的狀態。

1. PostgreSQL commit job。
2. process crash 或 queue publish 失敗。
3. queue 永遠收不到 message。

把 publish 移到 commit 前面，只會產生相反的不一致狀態。單純看到相同
job ID 也不等於 exactly-once。普通的 check-then-act worker 仍可能 race；
如果 external side effect 成功後、寫入 completed 前 crash，retry 仍可能
重複執行副作用。

The initial response treated reproduction as a prerequisite. Rare distributed
failures often need reconstruction from timestamps, acknowledgements, message
IDs, logs, and traces.

More importantly, it did not initially model the crash window between two
independent side effects. Reordering the operations creates the opposite invalid
state. Detecting the same job ID is not the same as exactly-once behavior.

## 更新後的心智模型｜Updated Mental Model

database commit 與 queue publish 不是同一個 atomic transition。在不使用
distributed transaction 的條件下，系統必須接受 at-least-once attempt，
再設計成最終收斂。

常見的 repair structure 有兩類：

- 在建立 job 的同一個 DB transaction 內寫入 outbox record，之後獨立
  publish 並標記 outbox 狀態。
- 定期 reconciliation stale job，把未正常推進的工作重新送出。

兩種做法都需要 idempotent 或 deduplicated processing。worker 取得執行權
也必須是 atomic conditional transition，不能先查 status、再普通 update。

processing 狀態還需要 lease、timeout、heartbeat 或 reconciliation 等恢復
語意。遇到 non-idempotent external effect 時，idempotency boundary 必須
延伸到外部系統，或使用 durable business key。queue 的 delivery semantic
本身不能保證業務副作用 exactly once。

A database commit and a queue publish are not one atomic transition. Without a
distributed transaction, the system accepts at-least-once attempts and makes
the workflow converge through outbox or reconciliation.

Execution claims must be atomic, processing states need recovery semantics, and
idempotency must include any external business side effect.

## 重測｜Retest

改用 payment settlement 情境：PostgreSQL 記錄 settlement request，worker
呼叫外部 payment provider。分別在 publish 前、publish 後、扣款後，以及
標記 completed 前注入 crash。

題目不提示 outbox 或 idempotency。回答需要自行設計 state machine、repair
loop、execution claim、observability evidence 與 duplicate-side-effect
protection。

Use a payment-settlement scenario and inject crashes before publish, after
publish, after charging, and before completion. Design the state machine,
repair loop, execution claim, evidence, and duplicate-side-effect protection
without naming the expected patterns in advance.
