# 5. Pattern Selection

  The right question isn't "multi-agent or not" but which pattern fits the workload. Most projects skip the comparison and copy the framework's README example, ending up with an architecture that doesn't match their workload.   

## 5.1 The patterns to choose between

**1. Single agent.** 
One LLM, one prompt, one toolbox. Default for any task that does not have a specific reason to fan out. Cheapest, easiest to debug, lowest latency.

**2. Single agent + intent classifier.** 
A small fast model classifies user intent first. The intent maps to a code path that may invoke a domain LLM, a deterministic function, or a rule-based response. The classifier is one extra LLM call; the rest is regular code. Used when intent space is finite and the LLM is overkill for routing.

**3. Supervisor multi-agent.** 
This guide's main subject, one supervisor LLM, multiple specialist agents, handoff mechanism. Used when domain expertise is divergent enough that prompts cannot share, or when routing genuinely needs LLM judgment.

**4. Plan-and-execute.** 
A planner LLM generates a sequence of steps, an executor LLM (or several) runs each step. Used when the work is structured enough to plan up front and the plan benefits from explicit inspection or modification before execution. Common in agentic coding tools and research assistants.

**5. Hierarchical multi-agent.** 
Supervisors of supervisors, a top-level orchestrator routes between domain orchestrators, each of which routes to specialists. Used when the system is large enough that a single supervisor's prompt would be unmanageable. Rarely needed below 8–10 specialist agents.

**6. Peer-to-peer / debate.** 
Agents communicate directly without a central coordinator, often pushing against each other (debate) or building on each other's work (collaboration). Used in evaluation, red-teaming, and multi-perspective analysis. Adversarial by design.

## 5.2 Decision criteria

Five questions narrow the choice.

**1. Can routing be expressed as keyword matching or simple rules?**

For categories with distinct vocabularies (e.g., `billing` vs `technical` in a support bot), keyword matching or regex separates them faster than an LLM, more cheaply, and more consistently across model swaps.

If routing depends on subtle signals (tone, implied intent, multi-clause messages with mixed concerns), LLM judgment is the right tool.

**2. Are the sub-tasks independent or sequential?**

Independent sub-tasks (search five sources, analyze three documents, query four databases) benefit from parallelism. Multi-agent runs them concurrently; single-agent runs them one at a time, so wall-clock time differs significantly.

Sequential sub-tasks (collect data → plan → track → adjust) do not benefit from multi-agent in the parallelism dimension. The pattern's value here is purely prompt isolation, which has to be weighed against the coordination tax (extra LLM calls and routing overhead per turn).

**3. Does each role need different prompts, models, or knowledge bases?**

Legal review and medical review on the same document need different prompts (different tone, different jargon), possibly different models (specialized fine-tunes), and definitely different RAG corpora, which multi-agent maps to naturally.

Roles that could share a prompt with a switch ("if billing, do X; if technical, do Y") do not need separate agents. The switch is what an intent classifier does, more efficiently.

**4. How much state needs to flow between agents?**

If agents work on separate parts of the request (e.g., one searches docs, another queries an API) and outputs combine at the end, state flow is minimal and multi-agent works well.

If agents need to share rich state (user profile, prior decisions, accumulated context), having every agent re-parse that state from message history is fragile, so a typed shared store (database, structured object) reduces the friction.

**5. What is the latency and cost budget?**

A multi-agent turn at 4 LLM calls instead of 1 is roughly 4× the cost and 2-3× the latency. If the workload is high-volume customer chat, this matters. If the workload is a once-an-hour analysis, it does not.

## 5.3 A decision flow

Working from the decisions a project can answer:

```
Is the task a single coherent step?
├── Yes → Single agent
└── No, multiple steps:
        Are they independent / parallelizable?
        ├── Yes → Multi-agent (supervisor or plan-and-execute, depending on planning needs)
        └── No, sequential:
                Do steps need different prompts/models/knowledge?
                ├── Yes → Multi-agent (supervisor)
                └── No:
                        Is routing simple (keyword/pattern)?
                        ├── Yes → Single agent + intent classifier
                        └── No → Multi-agent (supervisor) — judgment-based routing
```

The flow biases toward simpler patterns. Pick multi-agent when a node leads there, not because the framework makes it easy.

## 5.4 Examples

**1. Customer support bot for a SaaS product.** 
Categories: billing, technical, sales. Intent boundaries are clear. Most messages map cleanly to one category. → Single agent + intent classifier. The classifier picks the category, the domain LLM handles the conversation. Going multi-agent would add cost without solving a problem.

**2. Research assistant that synthesizes information from web search, academic papers, internal documents, and APIs.** 
Sources are independent, each with its own retrieval logic and prompt, and the final step is synthesis. → Multi-agent. Specialists handle each source in parallel; a synthesizer combines.

**3. Legal contract review.** 
Aspects: clause identification, risk assessment, citation lookup, comparison to template. Each aspect has different prompt structure and may use different models. → Multi-agent (supervisor) or plan-and-execute, depending on whether the aspects always run in the same order.

**4. Medical triage chatbot.** 
Categories: symptom intake, urgency assessment, escalation, general info. Some steps need to happen in sequence (intake before assessment). Routing has subtle dimensions (severity assessment is not a keyword match). → Multi-agent (supervisor), with explicit handoff between intake and assessment specialists.

**5. Habit tracker / fitness assistant.** 
Steps: collect user data, generate plan, log activity, adjust over time. Sequential, each step backed by a clear tool that does the actual work, and routing is keyword-matchable on most messages. → Single agent + intent classifier. Splitting into specialists buys nothing the classifier doesn't already give you.

## 5.5 The migration path

Most projects do not need to commit to a pattern up front. Single agent + classifier is a good starting point. It runs the same tools as a multi-agent setup would, just under one prompt. If the prompt becomes unmanageable or specific failure modes (tool selection drift, persona bleed) appear, splitting one role out into a specialist is mechanical: extract the relevant tools and prompt section into a new agent, route to it for the relevant intent.

Going the other direction (collapsing a multi-agent setup back to single agent) is harder. State and prompt structure assume the split. The lesson: start simple, escalate when you have evidence that simpler isn't enough.
