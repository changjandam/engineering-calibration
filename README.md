# Engineering Calibration｜工程判斷校準紀錄

這個 repository 記錄我面對陌生工程情境時如何思考、第一次假設在哪裡
失效，以及 mental model 如何隨著學習與重測而改變。

它是公開的工程判斷與推理證據，不是技術部落格、side project、面試標準
答案集，也不是自我評分牆。

This repository records how I reason through unfamiliar engineering scenarios,
where my first assumptions fail, and how my mental models evolve.

It is public evidence of engineering judgment. It is not a blog, a side
project, an interview-answer collection, or a self-rating wall.

## 每筆紀錄展示什麼｜What each record shows

每筆紀錄使用相同結構，讓讀者能分辨「第一次遇到問題時的能力」與「學習後
整理出的答案」。

- **情境｜Scenario**：定義貼近真實工作的工程問題。
- **限制｜Constraints**：排除不切實際或過度方便的假設。
- **第一次回答｜My First Response**：保留原始用字、錯誤與不完整之處，
  不事後美化。
- **做對的部分｜What I Got Right**：指出第一次回答中已存在的有效判斷。
- **遺漏的部分｜What I Missed**：明確記下未辨識的 failure mode、機制、
  差異或決策條件。
- **更新後的心智模型｜Updated Mental Model**：記錄補強後的理解。
- **重測｜Retest**：用不同情境檢驗概念是否真的能遷移。

Every record uses the same structure so that the reasoning remains inspectable
over time. The first response is kept separate from the updated explanation.

## 原則｜Principles

這些限制用來維持紀錄的可信度。

- 第一次回答保留原本語言與用字，不修正成看似完美的答案。
- 後續理解與初始回答分開，避免倒果為因。
- 不公開自評的職級或分數，讓讀者自行判斷 reasoning。
- 工作案例會抽象化，移除公司與客戶敏感資訊。
- 看完內容不等於內化；必須能轉移到新的陌生情境。
- 重視資訊密度與品質，不追求大量產文。

The English records follow the same rules: first responses stay unchanged,
later understanding remains separate, sensitive details are generalized, and
transfer is tested through a new scenario.

## Calibration records｜校準紀錄

1. [Durable AI Execution｜耐久 AI 執行](ai-systems/01-durable-ai-execution.md)
   檢驗如何把 agent loop 從綁定前端 session 的生命週期，移到可恢復的
   後端 workflow runtime。
2. [DB + Queue Reliability｜資料庫與佇列可靠性](systems/02-db-queue-reliability.md)
   檢驗 PostgreSQL 已提交 job，但 queue message 消失時的 failure model。
3. [Last Item Race｜最後一件商品的競態](systems/03-last-item-race.md)
   檢驗兩位買家同時搶最後一件庫存時的 concurrency control。
4. [Stale Search Response｜搜尋結果競態](frontend/04-stale-search-response.md)
   檢驗 frontend concurrent requests 亂序回來時，如何避免舊 response 覆蓋最新使用者意圖。
5. [Zero-Downtime Schema Change｜不中斷服務的 Schema 變更](systems/05-zero-downtime-schema-change.md)
   檢驗 rolling deployment、舊資料與 schema migration 同時存在時的 compatibility 與 rollout reasoning。

後續紀錄使用 [雙語固定模板](TEMPLATE.md)。

## 閱讀提醒｜Reading note

My First Response 是歷史證據，不代表推薦答案。Updated Mental Model
說明後來理解了什麼，Retest 則定義未來如何驗證這個理解是否真的內化。

The first-response sections are historical evidence, not recommended final
answers. Updated sections describe what changed, and retest sections define how
that change will be challenged.
