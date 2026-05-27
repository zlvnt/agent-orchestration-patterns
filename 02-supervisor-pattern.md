# 2. The Supervisor Pattern

The supervisor pattern has three components: a supervisor LLM, a set of specialist agents, and a handoff mechanism. Most multi-agent frameworks ship a version of it. This section breaks down the parts mechanically, using LangGraph as the reference because it is the most explicit about what each piece does.

## 2.1 Components

**1. The supervisor.** 
An LLM with a prompt describing the available specialists and their responsibilities. The supervisor does not call domain tools (no `lookup_invoice`, no `refund_charge`). It has exactly one kind of tool: handoff tools, one per specialist.

**2. Specialists.** 
Agents with focused prompts and small tool sets. Each specialist is itself a ReAct agent, it can call tools, observe results, and reason in a loop until done. The specialist is unaware of the supervisor; it sees a request and responds.

**3. Handoff tools.** 
Functions the supervisor can call to transfer control to a specialist. They are not regular tools. Calling a handoff tool ends the supervisor's turn and starts the named specialist's turn, with conversation state passed along.

A minimal customer-support example:

```python
from langgraph_supervisor import create_supervisor
from langchain.agents import create_agent as create_react_agent

billing = create_react_agent(
    model=llm,
    tools=[lookup_invoice, refund_charge],
    name="billing_agent",
    system_prompt="You handle billing questions: invoices, refunds, payment methods.",
)

technical = create_react_agent(
    model=llm,
    tools=[check_service_status, create_ticket],
    name="technical_agent",
    system_prompt="You handle technical issues: outages, error messages, configuration.",
)

workflow = create_supervisor(
    agents=[billing, technical],
    model=llm,
    prompt="Route billing questions to billing_agent, technical issues to technical_agent.",
)

graph = workflow.compile()
```

`create_supervisor` wires the supervisor LLM, generates handoff tools (`transfer_to_billing_agent`, `transfer_to_technical_agent`), and produces a graph. The graph is what you invoke per turn.

## 2.2 How handoff works

A turn proceeds in steps. Trace through what happens for a billing question.

1. User message lands on the supervisor.
2. Supervisor LLM call. Output: a tool call `transfer_to_billing_agent`.
3. The handoff tool executes. It does not run domain logic. It signals the graph to switch the active node from `supervisor` to `billing_agent`. Conversation state (full message history by default) is passed to the specialist.
4. Specialist LLM call. The billing agent reasons about the request and calls `lookup_invoice` or whatever it needs.
5. Tool execution. Tool result becomes a new message in state.
6. Specialist may call more tools or produce a final reply. ReAct loop runs until no tool call is emitted.
7. Specialist's final message routes back to the supervisor.
8. Supervisor LLM call again. It decides whether to forward, route to another specialist, or stop.

That is three to four LLM calls for one turn: two for the supervisor (steps 2 and 8), one or more for the specialist depending on how many tools it chains.

The pattern looks simple in the diagram. The actual flow is a state machine with the supervisor running before and after every specialist invocation.

## 2.3 State

State is the conversation history plus any framework-managed metadata. By default, every message (user input, supervisor decisions, tool calls, tool results, specialist replies) accumulates into a single list and gets passed to whichever agent runs next.

This has consequences.

**1. Specialists see everything.** 
The billing agent sees the supervisor's reasoning, prior specialists' outputs, and the user's full message. This is helpful for context and dangerous for prompt drift, the specialist may pick up tone or instructions from the supervisor's chatter.

**2. Tool results stay visible.** 
A specialist that ran `lookup_invoice` leaves the invoice data in state. The next specialist (or the supervisor) can read it. This is how implicit data sharing works across agents and where it breaks down when prompts do not specify what to read.

**3. Output mode controls what flows back.** 
LangGraph's `create_supervisor` accepts `output_mode="full_history"` (everything) or `output_mode="last_message"` (only the specialist's final reply). `last_message` is cheaper and reduces context contamination. `full_history` preserves trace for debugging and gives the supervisor more to reason about.

```python
workflow = create_supervisor(
    agents=[billing, technical],
    model=llm,
    prompt=...,
    output_mode="last_message",  # default for production
)
```

## 2.4 Variations

The supervisor pattern has dimensions you can vary without leaving it.

**1. Model per agent.** 
The supervisor and specialists do not need the same model. A common optimization: a small fast model for the supervisor (routing is cheap reasoning), a larger model for specialists (where the actual quality matters). This is called the *router-worker split*.

**2. Synchronous vs streaming.** 
The graph can run synchronously (collect everything, return at the end) or stream tokens / events as they arrive. Streaming changes how you handle the supervisor's intermediate decisions in the UI.

**3. Handoff payloads.** 
By default, handoff tools take no arguments. You can extend them to pass structured context (`transfer_to_billing_agent(reason="invoice_question", invoice_id=12345)`) so the specialist receives explicit hints rather than re-parsing the message history.

**4. Cross-specialist handoff.** 
Specialists can have handoff tools to other specialists, not just back to the supervisor. This enables `technical_agent → billing_agent` direct transfers. It also enables loops, which is why most production setups restrict cross-handoff to a small whitelist.

**5. Supervisor scope.** 
A supervisor that only routes is the lean version. A supervisor that also does anti-fabrication checks, formatting normalization, and language enforcement is the fat version. Fat supervisors are easier to write rules for and harder to keep consistent. See [pitfall 4.2 (Prompt rule accumulation)](04-common-pitfalls.md#42-prompt-rule-accumulation) for what happens when fat supervisors grow uncontrolled, and how to mitigate.

## 2.5 What the framework gives you, and what it does not

Frameworks like LangGraph provide the state machine, the handoff tool generation, the message accumulation, and the graph compilation. They do not solve:

- Prompt design for the supervisor and specialists
- Decisions about output mode, model split, cross-handoff
- Failure modes (loops, routing errors, state misreads)
- Evaluation

A framework will let you stand up a supervisor in twenty lines. The rest is what the next sections cover.
