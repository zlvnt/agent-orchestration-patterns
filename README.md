# Multi-Agent Orchestration — A Practical Reference

A guide to building, debugging, and evaluating multi-agent systems with the supervisor pattern. Targets developers who have worked with single-agent LLM applications and are considering multi-agent for the first time.

The bias of this guide: prefer simpler patterns. Multi-agent is the right tool for specific workloads. For most workloads, it is overhead masquerading as architecture. The first half of the guide explains what the pattern does. The second half explains when to skip it and how to measure whether the version you built is working.

## Contents

1. [What and Why](01-what-and-why.md) — definition, single-agent baseline, costs, when multi-agent wins
2. [The Supervisor Pattern](02-supervisor-pattern.md) — components, handoff mechanics, state, variations
3. [Minimal Implementation](03-minimal-implementation.md) — runnable skeleton in LangGraph, with tools, specialists, checkpointer
4. [Common Pitfalls](04-common-pitfalls.md) — handoff loops, prompt rule accumulation, implicit state, anti-fabrication, tool-use reliability, supervisor finalize cost
5. [Pattern Selection](05-pattern-selection.md) — alternatives to supervisor multi-agent, decision criteria, decision flow
6. [Evaluation](06-evaluation.md) — what to measure, test case design, metrics, LLM-as-judge, reading results
7. [Case Reference](07-case-reference.md) — pointer to the Health Intelligence Agent project as a worked example

## Appendix

- [Pattern Variants](appendix-patterns.md) — plan-and-execute, hierarchical, peer-to-peer; when to reach for them instead of supervisor

## Prerequisites

- Python familiarity
- Some prior use of an LLM API (Anthropic Claude, OpenAI, or similar) for tool calling
- Conceptual familiarity with what an agent loop is (receive input → reason → optionally call tools → reply)

LangGraph experience is not required. The guide introduces the relevant pieces as they come up.

## What this guide is not

A framework comparison. The code examples use LangGraph because it is the most explicit framework about how the pattern works under the hood. The patterns and pitfalls translate to CrewAI, AutoGen, custom implementations, and any other framework that lets you wire a coordinator and a set of specialists.

A claim that multi-agent is the future of LLM applications. It is one tool. The guide includes the cases where it is the wrong tool, because those cases are common and underdocumented.
