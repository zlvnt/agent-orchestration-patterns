# Appendix: Multi-Agent Pattern Variants

The main guide focuses on the supervisor pattern. This appendix covers the other common multi-agent patterns: what they are, when they fit, and how they differ.

## A.1 Supervisor

The pattern the main guide covers in depth, recapped here for comparison.

```
       Supervisor
       ↓  ↓  ↓
   A1   A2   A3   ← specialists
```

A central LLM coordinator routes each user request to one of several specialists, each with a focused role and tool set. The coordinator itself does not call domain tools; its job is routing.

**When to use.** Workloads where intent is clear once classified, but the right way to handle each intent differs significantly enough to warrant separate prompts and tool sets. Most "customer service router" or "domain expert dispatcher" use cases.

**Trade-off.** Coordination tax (multiple LLM calls per turn). Best when classifications are stable and routing decisions are not the bottleneck.

## A.2 Plan-and-Execute

```
   User: "Build feature X"
        ↓
   Planner LLM
        ↓
   Plan: [step1, step2, step3, step4]
        ↓
   Executor LLM (or pool)
        ↓
   step1 ✓ → step2 ✓ → step3 ✓ → step4 ✓
```

A planner agent decomposes the request into an explicit, ordered sequence of steps, which an executor (or pool of executors) then runs. The plan exists as a first-class artifact: it can be inspected, edited, or replanned mid-execution.

**Difference from supervisor:** Supervisor decides routing one step at a time, reactively. Plan-and-execute decides the whole sequence up front, proactively. Supervisor is reactive; plan-and-execute is deliberative.

**When to use.**
- Tasks with multiple dependent sub-steps where the order matters
- Workloads where you want a human or another agent to review the plan before execution
- Long-running tasks where intermediate state needs to be tracked against an explicit goal

**Concrete example.** Agentic coding tools (Cursor's agent mode, GitHub Copilot Workspace, Devin). User says "add OAuth login". Planner produces: "1. Add dependencies → 2. Create OAuth handler → 3. Update routes → 4. Add tests". Executor implements each step, possibly with intermediate human approval.

**Trade-off.** Plan quality is a single point of failure: a bad plan cascades into bad execution. Replanning is possible but adds complexity, so the pattern works best when the task is structured enough that planning has high signal.

## A.3 Hierarchical

```
              Top Supervisor
              ↓           ↓
        Sub-Sup A      Sub-Sup B
        ↓  ↓  ↓        ↓  ↓  ↓
       a1 a2 a3       b1 b2 b3
```

Supervisors of supervisors: the top-level coordinator routes to a sub-coordinator, which routes on to a specialist, and the tree can run three or more levels deep.

**When to use.**
- Specialist count exceeds what one supervisor can reason about effectively (typically past 8–10)
- Specialists naturally cluster into domains (e.g. all "billing" specialists, all "technical" specialists)
- Different domains need different supervisor prompts because routing logic differs per domain

**Concrete example.** A large enterprise customer service system. Top supervisor routes between billing-domain, technical-domain, and sales-domain. Within billing, sub-supervisor routes between invoices, refunds, payment methods, dispute handling. Each level adds context that the layer below assumes.

**Trade-off.** Each level adds an LLM call to the turn: a 3-level hierarchy costs at least 3 routing calls before any specialist runs. That tax compounds, so it is only worth paying when single-supervisor prompts have actually become unmaintainable.

## A.4 Peer-to-Peer

```
   Agent A ←──→ Agent B
      ↕            ↕
   Agent C ←──→ Agent D
```

Peer-to-peer has no central coordinator: agents communicate directly, often pushing against each other (debate) or building on each other's work (collaboration). Termination comes from consensus, an iteration limit, or a shared judge.

**When to use.**
- Adversarial setup is the design (debate, red-team / blue-team, proof-by-contradiction)
- Iterative refinement where multiple perspectives improve the output (writer + critic + editor cycles)
- Workloads where no single agent has the authority to decide what is "done"

**Concrete example.** A code-review pipeline. Agent A reviews for security, Agent B for performance, and Agent C for readability; all three run independently and produce critiques, then Agent D synthesizes them into a final review. No supervisor decides which critic to call: they all run, and the synthesizer decides what matters.

Another: debate simulators for evaluation. Agent A argues a position, Agent B argues the opposing one, and both iterate until a judge agent (or human) scores at the end.

**Trade-off.** Coordination is harder: without a central decision-maker, deciding when to stop and how to combine outputs needs either explicit rules (max iterations, consensus thresholds) or another agent acting as a judge, at which point the system starts looking hierarchical anyway.

## A.5 Comparison

| Pattern          | Coordinator         | Decision style         | Best fit                                |
| ---------------- | ------------------- | ---------------------- | --------------------------------------- |
| Supervisor       | Central LLM         | Reactive, per-turn     | Domain dispatch with clear intents      |
| Plan-and-execute | Planner LLM         | Deliberative, up front | Structured multi-step tasks             |
| Hierarchical     | Tree of supervisors | Reactive, multi-level  | Many specialists clustered into domains |
| Peer-to-peer     | None (or judge)     | Iterative / consensus  | Adversarial or collaborative refinement |

A workload may also combine patterns: a plan-and-execute system whose executors are themselves supervisors, for instance, or a peer-to-peer debate where each "agent" is internally a supervisor over its own specialists.

## A.6 Choosing among them

The decision criteria from Section 5 (Pattern Selection) apply across all four patterns, with shifts of emphasis:

- **Routing complexity:** high routing complexity favors supervisor or hierarchical
- **Sub-task independence:** high parallelism favors peer-to-peer
- **Need for upfront planning:** high planning value favors plan-and-execute
- **Specialist count:** high count favors hierarchical
- **Adversarial design:** required favors peer-to-peer (the others cannot model it)

The default starting point is still single-agent, escalating to supervisor when that is not enough. The other patterns are specialized tools: reach for them only when the workload's shape specifically calls for it.
