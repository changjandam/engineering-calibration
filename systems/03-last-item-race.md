# Calibration 03: Last Item Race

This record examines a check-then-act race when two buyers attempt to purchase
the final inventory item.

- Initial test: 2026-08
- Retest: Not completed
- Domain: Database concurrency

## Scenario

An inventory row contains `stock = 1`. Two requests execute nearly
simultaneously:

1. Read the current stock.
2. If stock is positive, create an order.
3. Decrement the stock.

Production occasionally ends with `stock = -1`.

## Constraints

The system must prevent overselling. Creating the order and consuming inventory
must also succeed or fail together.

The answer must distinguish transaction atomicity from concurrency control. It
cannot assume that placing the existing read and update inside a transaction
automatically removes the race.

## My First Response

The following is the original first response.

> 我對這個也不熟 只能用猜測的
> 但我直覺我們把兩段query拆開讓他有狀態過期的窗口
> 不知道有沒有關閉窗口的辦法

When asked how order creation and inventory deduction should relate, the
original follow-up was:

> transaction? 我記得sql本來就有這方面的語法

## What I Got Right

The response located the bug in the time window between reading state and
acting on it. It described stale state without needing the standard term
"check-then-act race."

It also recognized that order creation and inventory deduction require an
all-or-nothing relationship.

## What I Missed

The response could identify the failure window but did not yet have concrete
tools for closing it. It did not distinguish:

- a transaction that makes order creation and inventory deduction atomic,
- an atomic conditional update that combines the stock check and decrement,
- a row lock that serializes access to the inventory row, and
- isolation behavior that determines what concurrent transactions can observe.

A transaction containing an ordinary read followed by an update is not, by
itself, proof that overselling is impossible.

## Updated Mental Model

There are two different invariants:

1. Only one buyer may successfully transition the final available unit to sold.
2. The successful stock transition and order creation must commit together.

A conditional update can make the stock transition atomic:

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = $1
  AND stock > 0;
```

The affected-row count determines whether the request acquired inventory.
Order creation and that state transition can then share one database
transaction.

A row-locking approach can also work by locking the inventory row before
reading and updating it. The important calibration change is to ask separately:
"What makes these writes all-or-nothing?" and "What prevents concurrent actors
from both winning?"

## Retest

Present a seat-reservation, coupon, or quota-allocation scenario with an
expiration rule and multiple concurrent claimants.

Without using the terms transaction, lock, or atomic update in the prompt,
explain the invariant, choose a concurrency-control mechanism, handle the
losing claimant, and state what the surrounding transaction guarantees.
