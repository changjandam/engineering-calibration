# Engineering Calibration

This repository is a record of how I reason through unfamiliar engineering
scenarios, where my first assumptions fail, and how my mental models evolve.

It is public evidence of engineering judgment. It is not a blog, a side
project, an interview-answer collection, or a self-rating wall.

## What each record shows

Every record uses the same structure so that the reasoning is inspectable over
time.

- **Scenario** defines a realistic engineering problem.
- **Constraints** removes convenient but unrealistic assumptions.
- **My First Response** preserves the original response without retroactive
  polishing.
- **What I Got Right** identifies useful reasoning already present.
- **What I Missed** names the specific gap exposed by the scenario.
- **Updated Mental Model** records the model I use after studying the gap.
- **Retest** defines a different future scenario that can test transfer.

## Principles

The repository follows a few constraints to keep the evidence honest.

- First responses remain in their original language and wording.
- Later understanding is separated from the initial answer.
- Records avoid seniority scores and let readers judge the reasoning.
- Work scenarios are generalized to remove company-sensitive details.
- A concept is not considered internalized until it transfers to a new
  scenario.
- Quality matters more than volume.

## Initial calibration set

The first set moves from architecture boundaries to distributed failure and
database concurrency.

1. [Durable AI Execution](ai-systems/01-durable-ai-execution.md) examines moving
   an agent loop from a client-bound session to a durable backend runtime.
2. [DB + Queue Reliability](systems/02-db-queue-reliability.md) examines a job
   committed to PostgreSQL whose queue message never appears.
3. [Last Item Race](systems/03-last-item-race.md) examines two buyers racing for
   the final inventory item.

Use [TEMPLATE.md](TEMPLATE.md) for future records.

## Reading note

The first-response sections are historical evidence, not recommended final
answers. The updated sections describe what changed, and the retest sections
state how that change will be challenged later.
