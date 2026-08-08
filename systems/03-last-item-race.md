# Calibration 03：Last Item Race｜最後一件商品的競態

這筆紀錄檢驗兩位買家同時購買最後一件商品時發生的 check-then-act race。

This record examines a check-then-act race when two buyers attempt to purchase
the final inventory item.

- 初次測試｜Initial test: 2026-08
- 重測｜Retest: 尚未完成 / Not completed
- 領域｜Domain: Database concurrency

## 情境｜Scenario

inventory row 的 stock 等於 1。兩個 request 幾乎同時執行以下流程：

1. 讀取目前 stock。
2. 如果 stock 大於 0，建立 order。
3. 將 stock 減 1。

Production 偶爾得到 stock 等於 -1。

An inventory row contains stock equal to 1. Two requests nearly simultaneously
read the stock, create an order when stock is positive, and decrement it.
Production occasionally ends with stock equal to -1.

## 限制｜Constraints

系統必須避免超賣；建立 order 與消耗 inventory 也必須一起成功或一起失敗。

回答需要區分 transaction atomicity 與 concurrency control，不能假設只要把
現有的 SELECT 與 UPDATE 放進 transaction，就自動消除 race。

The system must prevent overselling. Creating the order and consuming inventory
must also succeed or fail together.

The answer must distinguish transaction atomicity from concurrency control. It
cannot assume that placing the existing read and update inside a transaction
automatically removes the race.

## 第一次回答｜My First Response

以下是原始第一次回答，沒有補上後來才學到的標準術語與解法。

The following is the original first response.

> 我對這個也不熟 只能用猜測的
> 但我直覺我們把兩段query拆開讓他有狀態過期的窗口
> 不知道有沒有關閉窗口的辦法

當問題進一步要求 order 與 inventory 必須一起成功或失敗時，原始追答如下。

When asked how order creation and inventory deduction should relate, the
original follow-up was:

> transaction? 我記得sql本來就有這方面的語法

## 做對的部分｜What I Got Right

回答直接抓到「讀取狀態」與「根據狀態行動」之間存在時間窗口。即使當時不
知道 check-then-act race 這個詞，也能辨識 stale state 是 bug 的核心。

後續也想到建立 order 與扣 inventory 必須具備 all-or-nothing 關係。

The response located the bug in the time window between reading state and
acting on it. It described stale state without needing the standard term
"check-then-act race."

It also recognized that order creation and inventory deduction require an
all-or-nothing relationship.

## 遺漏的部分｜What I Missed

當時能指出問題窗口，但還沒有具體工具把窗口關掉，也沒有區分以下機制：

- transaction：讓 order creation 與 inventory deduction 一起 commit 或
  rollback。
- atomic conditional update：把 stock check 與 decrement 合成單一狀態
  轉換。
- row lock：序列化對同一筆 inventory row 的存取。
- isolation behavior：決定 concurrent transactions 可以觀察到什麼狀態。

普通的 SELECT 接 UPDATE 即使放在 transaction 中，也不能單獨證明不會
oversell。

The response identified the failure window but did not yet have concrete tools
for closing it. It did not distinguish transaction atomicity, an atomic
conditional update, row locking, and isolation behavior.

A transaction containing an ordinary read followed by an update is not, by
itself, proof that overselling is impossible.

## 更新後的心智模型｜Updated Mental Model

這個情境有兩個不同 invariant：

1. 最後一件有效庫存只能由一位買家成功取得。
2. 成功的庫存狀態轉換與 order creation 必須一起 commit。

可以用 conditional update 把 stock transition 變成 atomic operation：

~~~sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = $1
  AND stock > 0;
~~~

affected-row count 決定這個 request 是否成功取得庫存。成功後，order
creation 與這個 state transition 可以放在同一個 DB transaction。

另一個作法是在讀取與修改前先 lock inventory row。真正需要分開問的是：

- 什麼機制讓多筆 write all-or-nothing？
- 什麼機制避免兩個 concurrent actor 同時成為 winner？

There are two separate invariants: only one buyer may win the final available
unit, and the successful stock transition must commit together with the order.

A conditional update or row-locking approach can enforce the first invariant.
A surrounding transaction can enforce the second. The calibration change is to
ask which mechanism provides each guarantee.

## 重測｜Retest

改用 seat reservation、coupon 或 quota allocation 情境，加入 expiration
rule 與多個 concurrent claimant。

題目不提示 transaction、lock 或 atomic update。回答必須自行定義
invariant、選擇 concurrency-control mechanism、處理失敗 claimant，並說明
外層 transaction 究竟保證什麼。

Present a seat-reservation, coupon, or quota-allocation scenario with expiration
and concurrent claimants. Without naming the mechanisms in advance, define the
invariants, choose the concurrency control, handle losers, and state what the
surrounding transaction guarantees.
