# 1. What and Why

Multi-agent orchestration splits a single user request across multiple LLM-powered agents, each with a focused role and tool set. A coordinator, usually called a *supervisor* or *orchestrator*, decides which agent handles which part of the request, and how partial results combine into a final response.

A typical setup:

```
User message
   │
   ▼
┌──────────────────┐
│   Supervisor     │  ← routing decision
└────────┬─────────┘
         │
   ┌─────┼─────┐
   ▼     ▼     ▼
┌─────┐ ┌─────┐ ┌─────┐
│ A1  │ │ A2  │ │ A3  │  ← specialists
└─────┘ └─────┘ └─────┘
```

Each specialist has its own prompt, its own subset of tools, and (sometimes) its own model. The supervisor sees the conversation history but does not invoke domain tools, its job is routing.

Supervisor is one pattern among several: plan-and-execute splits planning from execution, hierarchical stacks supervisors of supervisors, peer-to-peer skips the central coordinator entirely. This guide focuses on supervisor because it is the most common starting point and the one most multi-agent frameworks (LangGraph, CrewAI, AutoGen) put in the README example. For deeper coverage of the other patterns, see [appendix-patterns.md](appendix-patterns.md).

## 1.1 The single-agent baseline

Before reaching for multi-agent, build the single-agent version mentally. A single-agent system has:

- One LLM with one prompt
- A toolbox of functions the LLM can call (database queries, API calls, RAG lookups)
- A loop: receive input, reason, optionally call tools, reply

For most tasks this is enough. The model is competent at routing tool calls based on user intent: `lookup_invoice` for "what's my last bill?", `check_service_status` for "is the API down?". The prompt holds the rules; the tools handle structured operations.

The single-agent ceiling shows up in three places.

**1. Prompt overload.** 
When a single prompt encodes multiple personas (assistant for X, expert for Y, advisor for Z), each with different tone, scope, and rules, the prompt grows long and the model becomes inconsistent. Rules conflict. New rules cause regressions in old behavior.

**2. Tool selection error rate.** 
Selection accuracy degrades as the toolbox grows. Past 10–15 tools, models pick the wrong tool, especially when descriptions overlap. Smaller models hit this limit sooner.

**3. Context contamination.** 
A user troubleshooting an API error asks a billing question mid-conversation. The model carries technical-debug context into the billing response and references error codes the user did not bring up.

These ceilings are not theoretical. They appear in production logs once the assistant handles real conversations.

## 1.2 What multi-agent buys

Multi-agent breaks the single-agent prompt into focused units:

- A billing specialist. Its prompt knows nothing about technical issues. Tools: `lookup_invoice`, `refund_charge`, `update_payment_method`.
- A technical specialist. Its prompt is about diagnosing service issues. Tools: `check_service_status`, `create_ticket`.
- A coordinator (supervisor) that decides which specialist gets the request.

Benefits map to the single-agent ceilings:

- Smaller prompts per role — easier to debug, easier to constrain
- Smaller tool subset per role — higher selection accuracy
- Implicit context isolation — the technical agent does not see the billing conversation, only the supervisor's handoff

This is not free.

## 1.3 The cost

A multi-agent turn costs more than a single-agent turn, usually 2–4× more LLM calls and sometimes more.

The breakdown for a typical turn:

- Supervisor LLM call (routing decision)
- Specialist LLM call (actual work)
- Often a second supervisor call to finalize or forward the specialist's response
- If specialists hand off to each other, multiply by the number of handoffs

A turn that was one LLM call becomes four. Latency and token cost scale with it.

New failure modes appear that single-agent systems do not have:

**1. Routing errors.** 
The supervisor sends the request to the wrong specialist.

**2. Handoff loops.** 
Supervisor → specialist A → specialist B → specialist A → ... Each hop costs a full LLM call. Loops can run until token limits stop them.

**3. State sharing failures.** 
Specialist B does not know what specialist A learned, because state lives in the conversation history and B parses it incorrectly. Data the user already provided gets re-asked.

**4. Inconsistent persona.** 
The supervisor wraps the specialist's response in its own voice, smoothing over the specialization that motivated the split in the first place.

These failure modes are why production multi-agent systems need evaluation infrastructure that single-agent systems can usually skip.

## 1.4 When multi-agent wins

The overhead pays for itself when at least one of these is true.

**1. Sub-tasks are genuinely independent.** 
A research assistant fans out to five sources in parallel, each with its own retrieval strategy and prompt, then synthesizes. Wall-clock time drops because the work runs concurrently. Single-agent cannot do this, it is sequential by construction.

**2. Adversarial setup is part of the design.** 
Debate simulators, red-team vs blue-team, proof-by-contradiction. The agents are supposed to push against each other. Collapsing them into one prompt collapses the design.

**3. Expertise is divergent enough that prompts cannot share.** 
Legal, financial, and medical review on the same document each need different prompts, different retrieval corpora, sometimes different models. A single prompt covering all three would be unmanageable.

**4. Routing genuinely needs LLM judgment.** 
Input is ambiguous, intent cannot be classified by keyword or pattern, and the choice of handler depends on subtle signals only an LLM can read.

If none of these are true, the multi-agent overhead pays for flexibility the workload does not use.

## 1.5 The wrong reason

Going multi-agent because the architecture diagram looks more impressive is a real failure mode. It produces systems that are slower, more expensive, more brittle, and not measurably better than the single-agent version they replaced.

The decision to use multi-agent is a decision to take on a coordination tax. The workload has to justify it.
