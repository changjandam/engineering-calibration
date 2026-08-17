# Calibration 04：Stale Search Response｜搜尋結果競態

這筆紀錄檢驗 React 搜尋頁面中，較舊 request 的 response 晚到，覆蓋較新結果的 race condition。

- 初次測試｜Initial test: 2026-08
- 重測｜Retest: 尚未完成 / Not completed
- 領域｜Domain: Frontend async state

## 情境｜Scenario

搜尋框每輸入一個字就呼叫 API。使用者快速輸入 `apple`，所有 request 都成功回 200，但畫面偶爾最後顯示的是 `app` 的搜尋結果，而不是 `apple`。

## 限制｜Constraints

Backend 沒有 error。問題發生在 frontend 如何處理多個 concurrent requests 與 state update。

## 第一次回答｜My First Response

> 我會猜res沒按照順序 或者我們的狀態更新沒按照順序

在被問到如何避免舊 response 蓋掉新結果時，原始追答是：

> 我平常用react query我記得他會自己判斷哪個res是最新的 簡陋點就是在某個環節壓req time 讓他根據req時間而不是res時間更新

後續又補充：

> 對 我忘記abort可以用了

## 做對的部分｜What I Got Right

第一次回答直接把問題定位到 request/response ordering 與 state update ordering，而不是先懷疑 backend 或 rendering。

即使沒有先說出特定 API，也已經抓到核心 invariant：較舊 request 的 response 不應該有資格覆蓋目前最新 query 的結果。

另外，平常使用 React Query 時也知道 server state 可以依 query key/cache 管理，而不是一定要把結果複製到另一份 context state。

## 遺漏的部分｜What I Missed

第一次回答沒有立即說出具體的 browser cancellation primitive，例如 `AbortController`，也沒有一開始區分：

- request cancellation
- latest-request identity / sequence
- library cache semantics
- 自己另外 `setState` 時可能重新引入 stale-write 問題

## 更新後的心智模型｜Updated Mental Model

這個問題的核心不是「response 應該照 request 順序回來」，而是 client 必須維持一個 invariant：

> 只有目前仍然代表最新使用者意圖的 request，才可以更新目前顯示狀態。

可用的方法包括取消舊 request，或為 request 建立 monotonic sequence / identity，在 response 回來時確認它仍是 latest request。

使用 React Query 等 server-state library 時，也需要理解 query key、cache entry、cancellation 與 local state duplication 的邊界，而不是假設 library 會在所有寫法下自動消除 race。

## 重測｜Retest

換成 autocomplete、route change 或 tab switch 情境，讓多個 request 的 response 亂序回來。

題目不提示 race、abort、sequence 或 React Query。回答必須自行辨識 stale write，並提出能保證「最新使用者意圖勝出」的機制。
