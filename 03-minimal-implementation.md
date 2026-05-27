# 3. Minimal Implementation

This section walks through a complete, runnable supervisor setup. Continuing the customer-support example from Section 2: a router that handles billing questions and technical issues. Code is in Python with LangGraph. Translating to other frameworks is mostly substitution.

## 3.1 Setup

Dependencies:

```bash
pip install langgraph langgraph-supervisor langchain langchain-anthropic
```

Environment:

```bash
export ANTHROPIC_API_KEY=...
```

The model factory:

```python
from langchain_anthropic import ChatAnthropic
import os

def make_llm():
    return ChatAnthropic(
        model="<model-id>",  # specific model ID from your provider
        api_key=os.getenv("ANTHROPIC_API_KEY"),
        temperature=0.7,
    )
```

The model choice is orthogonal to the pattern. Any LangChain-supported chat model class works in the same code with a different import. This example uses Anthropic for concreteness; provider, model size, and pricing are workload-specific decisions.

## 3.2 Tools

Tools are plain functions decorated with `@tool`. Type hints become the schema the LLM sees. Docstrings become the description the LLM reads when deciding whether to call.

```python
from langchain_core.tools import tool

@tool
def lookup_invoice(invoice_id: str) -> str:
    """Fetch invoice details by ID. Returns amount, status, due date."""
    # In a real app this hits a database. Mock for the guide.
    return f"Invoice {invoice_id}: $42.00, paid, 2026-04-15"

@tool
def refund_charge(invoice_id: str, amount: float, reason: str) -> str:
    """Issue a refund for an invoice. Reason is required for audit."""
    return f"Refunded ${amount} on invoice {invoice_id}: {reason}"

@tool
def check_service_status(service: str) -> str:
    """Check the operational status of a named service."""
    return f"{service}: operational, last incident 12 days ago"

@tool
def create_ticket(title: str, severity: str, description: str) -> str:
    """Create a support ticket. Severity is one of: low, medium, high."""
    return f"Ticket created: {title} (severity={severity})"
```

Two design points worth noting now (Section 4 covers more):

- **Required arguments are typed.** 
`severity: str` is weak because the model can pass anything. `severity: Literal["low", "medium", "high"]` would constrain it. Stronger schemas reduce hallucinated arguments.

- **Docstrings are not optional.** 
They are the model's only signal for tool selection. Vague docstrings cause vague tool calls.

## 3.3 Specialists

Each specialist is a ReAct agent with a focused prompt and a small tool set:

```python
from langchain.agents import create_agent as create_react_agent

BILLING_PROMPT = """You handle billing-related questions.

Capabilities:
- Look up invoice details with lookup_invoice
- Issue refunds with refund_charge

If the user asks about something outside billing (technical issues, feature questions),
say so briefly and stop. Do not attempt to answer.
"""

TECHNICAL_PROMPT = """You handle technical issues.

Capabilities:
- Check service status with check_service_status
- Create support tickets with create_ticket

If the user asks about billing or account questions, say so briefly and stop.
"""

billing = create_react_agent(
    model=make_llm(),
    tools=[lookup_invoice, refund_charge],
    name="billing_agent",
    system_prompt=BILLING_PROMPT,
)

technical = create_react_agent(
    model=make_llm(),
    tools=[check_service_status, create_ticket],
    name="technical_agent",
    system_prompt=TECHNICAL_PROMPT,
)
```

The `if the user asks about X, say so and stop` clause matters. Without it, specialists drift outside their scope and answer adjacent questions. The supervisor cannot enforce scope after the fact: once the specialist replies, the supervisor's options are forward, re-route, or stop.

## 3.4 Supervisor

The supervisor is wired with `create_supervisor`:

```python
from langgraph_supervisor import create_supervisor

SUPERVISOR_PROMPT = """You route customer requests to the right specialist.

Specialists:
- billing_agent: invoices, refunds, payment methods, charges
- technical_agent: service outages, error messages, configuration, support tickets

Rules:
- Route to exactly one specialist per turn.
- Do not answer directly unless the request is a greeting or unclear. For unclear
  requests, ask one short clarifying question.
- Do not invent context. If the specialist asks the user a question, forward it
  to the user verbatim.
"""

workflow = create_supervisor(
    agents=[billing, technical],
    model=make_llm(),
    prompt=SUPERVISOR_PROMPT,
    output_mode="last_message",
)

graph = workflow.compile()
```

`output_mode="last_message"` means only the specialist's final reply propagates back to the supervisor, not the full intermediate trace, which makes calls cheaper and reduces context noise. Use `full_history` only when debugging.

## 3.5 Invoking

Synchronous invoke:

```python
result = graph.invoke({
    "messages": [{"role": "user", "content": "Can you refund invoice INV-1042?"}]
})

# result["messages"] is the full message history
# The last AI message is the response to send back to the user
last_message = result["messages"][-1]
print(last_message.content)
```

Streaming events for observability:

```python
async for event in graph.astream(
    {"messages": [{"role": "user", "content": "Is the API service up?"}]},
    stream_mode="updates",
):
    print(event)
```

`astream` yields one event per node update: supervisor decision, tool call, tool result, specialist reply. This is what you log for trace inspection in production.

## 3.6 Persistent state

The setup above is stateless. Each `invoke` starts from scratch. For multi-turn conversations the graph needs a checkpointer.

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

checkpointer = AsyncPostgresSaver.from_conn_string("postgresql://...")
await checkpointer.setup()  # creates tables, run once

graph = workflow.compile(checkpointer=checkpointer)
```

Now invokes are scoped to a thread:

```python
config = {"configurable": {"thread_id": "user-12345"}}
result = await graph.ainvoke({"messages": [...]}, config=config)
```

Subsequent invokes with the same `thread_id` resume from the saved state. This is what makes multi-turn conversations possible without sending the full message history every call.

In-memory and SQLite checkpointers exist for development. PostgreSQL is the most commonly used production backend in LangGraph (officially supported with full async via `AsyncPostgresSaver`); community checkpointers exist for Redis, MongoDB, and others. Pick based on what your stack already runs.

## 3.7 What this skeleton gives you and what it does not

You now have:

- A supervisor that routes to two specialists
- Specialists with focused prompts and tools
- Multi-turn conversation via checkpointer
- Streaming events for observability

You do not have:

- Any guarantee the supervisor routes correctly. Routing accuracy on real traffic is typically well below 100% out of the box, depending on prompt and model.
- Any handling of edge cases (ambiguous input, multi-intent messages, off-topic chatter, prompt injection attempts).
- Loop prevention. The setup above will happily run a `supervisor → A → supervisor → A → ...` cycle until something breaks.
- Tool-use discipline. Some models call tools reliably; others reason about calling them and then skip the actual call. There is no framework-level fix for this.
- Evaluation infrastructure. You cannot measure routing accuracy without test cases and metrics.

[Section 4 (Common Pitfalls)](04-common-pitfalls.md) covers the failure modes you will hit. [Section 6 (Evaluation)](06-evaluation.md) covers how to measure them.
