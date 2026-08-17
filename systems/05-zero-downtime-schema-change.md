# Calibration 05：Zero-Downtime Schema Change｜不中斷服務的 Schema 變更

這筆紀錄檢驗 rolling deployment 期間，如何安全讓舊版與新版 application 同時面對 schema evolution。

- 初次測試｜Initial test: 2026-08
- 重測｜Retest: 尚未完成 / Not completed
- 領域｜Domain: Database migration / deployment

## 情境｜Scenario

Production backend 使用 PostgreSQL。`users` table 約 500 萬筆，需要新增 `email_verified` 欄位。

條件：

- PostgreSQL 是單一 primary。
- Backend 有多個 instance。
- Deployment 採 rolling deployment，因此一段時間內舊版與新版會同時存在。
- 舊版完全不知道這個欄位。
- 新版會讀寫這個欄位。
- 不能停機維護。

## 限制｜Constraints

可以修改 application code 與 migration，但不能要求先把所有舊 instance 停掉，也不能把 migration 當成瞬間完成。

## 第一次回答｜My First Response

> 我不知道 這問題我沒碰過肯定先問ai 但我試試看 如果新增欄位 那代表有新增的功能或是服務需要他 同時因為是新增的 所以舊資料一定沒有 我有可能不動舊table 開一個新的 新功能去找新table 舊功能不吃這欄位沒差 必要時新功能需要用舊表的資料當fallback

## 做對的部分｜What I Got Right

回答沒有把 schema change 當成單純的 `ALTER TABLE` 指令，而是先考慮新舊資料與新舊 application version 的 coexistence。

也自然想到 backward compatibility：舊版不能因為新功能出現而立刻失效，新版則可能需要處理舊資料缺少新狀態的情況。

## 遺漏的部分｜What I Missed

第一次回答沒有自然展開 production schema migration 常見的幾個 concern：

- schema change 本身是否會造成 lock 或長時間阻塞
- 500 萬筆舊資料是否需要 backfill，以及如何控制 batch size / DB load
- rolling deployment 期間 schema 必須同時相容 old code 與 new code
- constraint（例如 `NOT NULL`）應在什麼時點加入
- rollback 時 code/schema compatibility 如何維持

另外，為單一新增欄位建立新 table 雖能隔離新功能，但也可能增加 join、同步與 lifecycle complexity；是否值得需要額外 trade-off。

## 更新後的心智模型｜Updated Mental Model

安全 schema evolution 應先問：

1. 變更是否 backward-compatible？
2. migration 是否可能造成 lock / resource spike？
3. old code / new code coexist 時，兩者都能否正常工作？
4. 舊資料如何 backfill？是否需要分批？
5. stricter constraint 應在資料完成轉換後再加入嗎？
6. rollout 或 migration 失敗時如何 rollback？

一個常見方向是 expand → deploy → backfill/migrate → contract：先加入向後相容的 schema，再部署能處理新舊狀態的 application，之後才逐步填資料與收緊 constraint。

這不是唯一解法；真正要保證的是 migration 過程中的 compatibility 與 operability。

## 重測｜Retest

換成欄位 rename、資料型別變更或拆表 migration，並加入大量資料與 rolling deployment。

題目不提示 expand/contract、backfill 或 lock。回答必須自行辨識 compatibility、resource impact、rollout order 與 rollback。
