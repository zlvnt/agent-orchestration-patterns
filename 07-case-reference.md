# 7. Case Reference: Health Intelligence Agent

The patterns, pitfalls, and evaluation framework in the previous sections come from a real project. This section points to it for readers who want to see how the abstract guidance maps to a working implementation, including the parts that did not work.

## 7.1 The project

A multi-agent system for health and nutrition tracking. One supervisor (orchestrator), four specialists:

- `assessment_agent` — collects user health data (age, weight, activity, goals)
- `planning_agent` — generates structured diet and activity plans
- `tracking_agent` — logs meals, computes daily summaries, looks up nutrition data via RAG
- `intervention_agent` — detects when adherence is failing and suggests adjustments

Stack: Python, FastAPI, LangGraph supervisor, Claude / Minimax-M2.7 (multi-provider), PostgreSQL (user data + checkpointer), Qdrant + fastembed (nutrition RAG), Next.js frontend.

This guide grew out of that project. The full source, including every file referenced in Section 7.3, lives in the project repository: https://github.com/zlvnt/Health-Intelligence-Agent

## 7.2 What it demonstrates

This project is a working example of every section in this guide.

**Section 1 (What and Why).** The project sits in the awkward zone: health tracking is sequential and the routing is largely keyword-matchable, so it is closer to a single-agent + classifier workload than a multi-agent one. The repository is therefore a useful counterexample: a complete, polished supervisor implementation for a domain that does not need supervisor.

**Section 2 (Supervisor Pattern).** `app/agent/graph.py` shows the full wiring with `create_supervisor`, four ReAct specialists, AsyncPostgresSaver checkpointer, output mode, and lifespan management. The handoff mechanics in Section 2.2 are exactly what runs.

**Section 3 (Minimal Implementation).** `app/agent/tools.py` is the tool definitions, `app/agent/prompts.py` is the prompts, `app/agent/model.py` is the multi-provider LLM factory. The skeleton in Section 3 is a stripped-down version of these files.

**Section 4 (Common Pitfalls).** Every pitfall in Section 4 was encountered during the project:

- *Handoff loops* — early prompts triggered supervisor → assessment → supervisor → assessment cycles. STOP RULE in `ORCHESTRATOR_PROMPT` is the mitigation.
- *Prompt rule accumulation* — `ORCHESTRATOR_PROMPT` grew from one paragraph to eight blocks of rules over the project. Each block was added in response to a specific observed failure.
- *State implicit via message history* — the planning specialist initially generated generic plans because its prompt said "base on assessment data" and the model interpreted this literally, ignoring data the user had typed in their message.
- *Anti-fabrication and over-step* — the supervisor would ask onboarding questions itself instead of routing; the assessment specialist would generate plans instead of stopping. Multiple iterations of `ANTI-OVERSTEP` and `SCOPE RULE` were the response.
- *Tool-use reliability* — Minimax-M2.7 reasoning correctly concluded "I should call collect_health_data" but did not emit the tool call. Cross-model testing against Haiku confirmed the issue was model-level, not prompt-level.

**Section 5 (Pattern Selection).** The project's `LESSONS_LEARNED.md` is a written argument that this domain does not need supervisor multi-agent. The eval results are the quantitative version of that argument.

**Section 6 (Evaluation).** The `eval/` directory contains the runner, the test cases (15 cases across four tiers), and the DeepEval-based judges. `eval/runner.py` is a working implementation of the eval loop in Section 6.5.

## 7.3 Where to read what

Suggested reading order if you want to trace the guide back to the implementation:

1. `LESSONS_LEARNED.md` — the project's own retrospective, including the eval results table and the architectural diagnosis
2. `docs/agent-orchestration.md` — the architecture as documented during the project (before the retrospective)
3. `app/agent/graph.py` — the supervisor wiring
4. `app/agent/prompts.py` — the prompts after all iterations
5. `eval/cases/routing_cases.py` — the test case definitions
6. `eval/runner.py` — the eval execution loop
7. `eval/docs/cases-5-7-13-analysis.md` — a worked example of trace inspection and root-cause analysis

The repository was finalized at the point where the architectural mismatch was clearly visible in the metrics. It is not maintained as a product, only as a reference for what the supervisor pattern looks like in a complete implementation.
