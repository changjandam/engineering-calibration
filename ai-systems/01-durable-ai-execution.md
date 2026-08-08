# Calibration 01: Durable AI Execution

This record examines an architecture decision that moved an agent execution
loop from a client-bound session to a durable backend workflow.

- Initial test: 2026-08
- Retest: Not completed
- Domain: AI systems / workflow runtime

## Scenario

A web application runs a multi-step LLM agent through a frontend-initiated
session. A client disconnect can interrupt or lose the conversation, and
subagent behavior is difficult to observe. The execution may need to continue
across service interruptions and later support human interaction.

Decide where the execution lifecycle belongs and whether to use Inngest,
Temporal, or a custom database-and-queue runtime.

## Constraints

The team moves quickly, has limited operational capacity, and wants to reuse the
existing TypeScript stack. The design must support multi-step execution,
recovery, observability, debugging, and reasonable maintenance cost.

The scenario does not assume that a product is suitable merely because its
author published an LLM example.

## My First Response

The following is the original response. Spelling, terminology, and unsupported
assumptions remain unchanged.

> 原始ai sdk是前端發動session 一旦斷線就會讓對話消失或中斷
> 也難以追蹤subagent的行為 因此我需要把llm loop完整拉回後端
> 前端只透過事件監聽的方式取得狀態 而不是回覆接收方
> 當初有幾個參考對象 包含claude code/openai官方自己的agent架構
> 其實也有機會 但對於服務中斷接續我覺得應該有更簡單的做法
> 於是survey了現有的流程管理套件 重新找到了tempral跟inngest
> 由於inngest較為輕量化 同時作者本人其實就有示範如何用inngest
> 實作llm lop 結合先前研究過的mastra作法 用inngest管理流程
> ai sdk 退回單步驟llm呼叫 是我們當下成本最低又能滿足需求的方案

A later clarification exposed the actual selection process.

> 我選inngest其實直覺是 他運行的服務比較少
> 另外在當時跟ai討論的過程得到的訊息是temporal是大型架構
> 例如netflix的系統層級在用的 因此就選了inngest
> 但我確實沒有比較他們實際佔用的記憶體大小 效能
> 跟開發體驗 debug難度這些問題

## What I Got Right

The first response identified that the client session was incorrectly carrying
the execution lifetime. Moving the LLM loop to the backend and making the
frontend an observer addressed the architectural boundary rather than only
adding reconnect behavior.

It also separated responsibilities: the AI SDK became a single-step model
invocation primitive, while the workflow runtime owned orchestration, recovery,
state, and observation. Avoiding a home-grown workflow engine reduced
implementation risk.

## What I Missed

The conclusion that Inngest was lighter and cheaper was not demonstrated.
"Fewer services" was a heuristic, not a complete operational-cost comparison.
"Temporal is used by Netflix" also did not prove that Temporal was unsuitable
for a smaller team.

The decision did not start with explicit must-have requirements, weighted
criteria, or failure validation. It did not compare resource use, runtime
semantics, restart behavior, duplicate execution, human-in-the-loop resume,
debuggability, or the operational consequences of each option.

An author-provided LLM example reduced integration uncertainty but did not
establish production suitability.

## Updated Mental Model

The first decision is about ownership: interactive clients initiate and observe
work, but durable execution belongs to a backend lifecycle that survives client
and process failure.

Tool selection then follows an explicit sequence:

1. Define invariants and hard requirements.
2. Separate coding convenience from runtime comprehensibility and
   debuggability.
3. Compare alternatives against team capacity, deployment constraints,
   recovery semantics, operational cost, and project risk.
4. Validate the highest-risk assumptions with failure tests.
5. Record rollback or replacement conditions.

A reasonable conclusion may still be Inngest. The calibration change is that
the conclusion must follow evidence about the team's constraints and the
runtime's behavior, not product positioning or familiarity.

## Retest

Design a long-running document-review agent that can pause for human approval,
call a non-idempotent external service, survive process restarts, and expose a
complete execution history.

Compare Inngest, Temporal, and a database-and-queue design without being told
which one is expected. State the must-have requirements, rank the criteria,
design failure tests, and define the condition that would reverse the choice.
