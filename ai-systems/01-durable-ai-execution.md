# Calibration 01：Durable AI Execution｜耐久 AI 執行

這筆紀錄檢驗一個 AI architecture 決策：如何把 agent execution loop 從
綁定 client session 的生命週期，移到可持續、可恢復的後端 workflow。

This record examines an architecture decision that moved an agent execution
loop from a client-bound session to a durable backend workflow.

- 初次測試｜Initial test: 2026-08
- 重測｜Retest: 尚未完成 / Not completed
- 領域｜Domain: AI systems / workflow runtime

## 情境｜Scenario

一個 web application 透過前端啟動的 session 執行多步驟 LLM agent。
client 斷線可能讓對話中斷或消失，subagent 行為也難以獨立觀測。execution
未來還需要跨越服務中斷，並支援 human-in-the-loop。

需要決定 execution lifecycle 應由哪一層負責，以及應採用 Inngest、
Temporal，或自行用 database + queue 建立 runtime。

A web application runs a multi-step LLM agent through a frontend-initiated
session. A client disconnect can interrupt or lose the conversation, and
subagent behavior is difficult to observe. The execution may need to continue
across service interruptions and later support human interaction.

The decision is where the execution lifecycle belongs and whether to use
Inngest, Temporal, or a custom database-and-queue runtime.

## 限制｜Constraints

團隊開發速度快，但維運能量有限，希望重用既有 TypeScript stack。設計必須
支援 multi-step execution、失敗恢復、observability、debugging，以及合理的
維護成本。

不能只因產品作者提供 LLM 範例，就直接推論該產品適合 production。

The team moves quickly, has limited operational capacity, and wants to reuse the
existing TypeScript stack. The design must support multi-step execution,
recovery, observability, debugging, and reasonable maintenance cost.

The scenario does not assume that a product is suitable merely because its
author published an LLM example.

## 第一次回答｜My First Response

以下保留當時的原始回答。拼字、術語、標點與尚未驗證的假設都沒有修正。

The following is the original response. Spelling, terminology, punctuation, and
unsupported assumptions remain unchanged.

> 原始ai sdk是前端發動session 一旦斷線就會讓對話消失或中斷
> 也難以追蹤subagent的行為 因此我需要把llm loop完整拉回後端
> 前端只透過事件監聽的方式取得狀態 而不是回覆接收方
> 當初有幾個參考對象 包含claude code/openai官方自己的agent架構
> 其實也有機會 但對於服務中斷接續我覺得應該有更簡單的做法
> 於是survey了現有的流程管理套件 重新找到了tempral跟inngest
> 由於inngest較為輕量化 同時作者本人其實就有示範如何用inngest
> 實作llm lop 結合先前研究過的mastra作法 用inngest管理流程
> ai sdk 退回單步驟llm呼叫 是我們當下成本最低又能滿足需求的方案

後續追問暴露了實際選型依據。

A later clarification exposed the actual selection process.

> 我選inngest其實直覺是 他運行的服務比較少
> 另外在當時跟ai討論的過程得到的訊息是temporal是大型架構
> 例如netflix的系統層級在用的 因此就選了inngest
> 但我確實沒有比較他們實際佔用的記憶體大小 效能
> 跟開發體驗 debug難度這些問題

## 做對的部分｜What I Got Right

第一次回答辨識出 client session 不應同時承擔 execution lifetime。把
LLM loop 移回後端，並讓前端只負責啟動與觀測，處理的是 architecture
boundary，而不是只補一層 reconnect。

責任切分也合理：AI SDK 降為單步驟 model invocation primitive；workflow
runtime 負責 orchestration、recovery、state 與 observation。沒有直接自製
workflow engine，也降低了 implementation risk。

The first response identified that the client session was incorrectly carrying
the execution lifetime. Moving the LLM loop to the backend and making the
frontend an observer addressed the architectural boundary rather than only
adding reconnect behavior.

It also separated responsibilities: the AI SDK became a single-step model
invocation primitive, while the workflow runtime owned orchestration, recovery,
state, and observation. Avoiding a home-grown workflow engine reduced
implementation risk.

## 遺漏的部分｜What I Missed

「Inngest 較輕、成本較低」並沒有被證明。服務數量較少只是一個 heuristic，
不是完整的 operational-cost comparison。「Temporal 被 Netflix 使用」也
無法推導出它不適合較小的團隊。

當時沒有先明確列出 must-have、criteria 權重與 failure validation，也沒有
比較 resource use、runtime semantics、restart behavior、duplicate
execution、HITL resume、debuggability，以及各方案帶來的維運後果。

作者提供的 LLM example 可以降低 integration uncertainty，但不能證明
production suitability。

The conclusion that Inngest was lighter and cheaper was not demonstrated.
"Fewer services" was a heuristic, not a complete operational-cost comparison.
"Temporal is used by Netflix" also did not prove that Temporal was unsuitable
for a smaller team.

The decision did not start with explicit must-have requirements, weighted
criteria, or failure validation. It did not compare resource use, runtime
semantics, restart behavior, duplicate execution, human-in-the-loop resume,
debuggability, or the operational consequences of each option.

## 更新後的心智模型｜Updated Mental Model

第一個問題是 ownership：互動式 client 可以啟動與觀測工作，但 durable
execution 應屬於能跨越 client failure 與 process failure 的後端生命週期。

工具選型依照以下順序進行：

1. 定義 invariants 與 hard requirements。
2. 把 coding convenience 與 runtime comprehensibility、debuggability
   分開評估。
3. 依 team capacity、deployment constraints、recovery semantics、
   operational cost 與 project risk 比較方案。
4. 用 failure test 驗證風險最高的假設。
5. 記錄 rollback 或替換工具的條件。

最後仍可能合理地選擇 Inngest。真正的改變是：結論必須建立在團隊限制與
runtime 行為的 evidence 上，而不是產品定位、熟悉度或單一案例。

The first decision is about ownership: interactive clients initiate and observe
work, but durable execution belongs to a backend lifecycle that survives client
and process failure. Tool selection follows explicit invariants, ranked
criteria, failure validation, and a documented reversal condition.

## 重測｜Retest

重新設計一個 long-running document-review agent：它可以等待人工核准、
呼叫 non-idempotent external service、在 process restart 後恢復，並提供
完整 execution history。

在不預設答案的情況下，比較 Inngest、Temporal 與 database + queue。
必須列出 must-have、排序 criteria、設計 failure tests，並說明什麼證據會
讓原本的選擇失效。

Design a long-running document-review agent with human approval, a
non-idempotent external service, process restarts, and a complete execution
history. Compare the three options, rank the criteria, design failure tests,
and define the condition that would reverse the choice.
