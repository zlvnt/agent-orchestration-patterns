# 4. Common Pitfalls

The supervisor pattern produces a small set of failure modes that recur across projects. They are not bugs in the framework. They are inherent to how the pattern works, and most of the work after the skeleton is in place is defending against them.

## 4.1 Handoff loops

**1. What it looks like.** 

The trace shows `supervisor → specialist_A → supervisor → specialist_A → ...` cycling. Each iteration is a full LLM call. The conversation does not progress. Eventually a token limit, recursion limit, or rate limit halts the loop, and the user gets either an error or a confused partial response.

**2. Why it happens.** 

The supervisor sees a specialist's reply and decides the work is incomplete, so it routes back to the same specialist. The specialist sees the same context and produces a similar reply. The loop is stable: each side does what its prompt says.

A common trigger: the specialist asks the user a clarifying question instead of finalizing. The supervisor reads this as "the specialist did not finish" and routes again. The specialist is waiting for a user reply that never comes within this turn.

**3. Mitigation.**

A hard recursion limit at the graph level. LangGraph's `recursion_limit` config caps total node executions per turn:

```python
graph.invoke(state, config={"recursion_limit": 25})
```

This is a backstop, not a fix. The fix is in the supervisor prompt: a STOP rule that says once any specialist has replied in the current turn, the turn ends, regardless of whether the reply contains a question.

```
STOP RULE: Once any specialist has produced a reply in the current turn,
do not route to another specialist. Forward the reply to the user and stop.
A question from a specialist is a question to the user, not a request for more routing.
```

Cross-specialist handoffs (specialist A to specialist B directly) require their own loop guard: each handoff tool can be called at most once per turn, enforced in code if the framework allows hooks, otherwise in the prompt.

## 4.2 Prompt rule accumulation

**1. What it looks like.** 
The supervisor prompt grows from 200 words to 800 over the course of a project. New rules get added each time a behavior is observed in production: STOP rules, anti-fabrication rules, output formatting rules, language rules, ambiguous-input rules, anti-overstep rules. Each rule fixes one observed failure. The prompt becomes hard to reason about as a whole.

**2. Why it happens.** 
Multi-agent failure modes do not come with neat names. Each one looks like a unique bug the first time you see it. The natural reaction is to add a prompt clause that prevents that specific case. Over time, clauses interact in ways that were not anticipated.

The deeper cause: the supervisor is being asked to enforce policy with natural-language rules. Some kinds of policy are easier to enforce in code than in prompts. Routing rules that match keywords are an example, easier to implement as a classifier or regex than to express as prompt instructions the LLM follows reliably.

**3. Detection signal.** 
The prompt has more than three CRITICAL or ABSOLUTE markers. Rules contradict each other in edge cases. Different LLMs interpret the same prompt and produce different routing.

**4. Mitigation.**

Promote frequently-violated rules out of the prompt and into code. If the supervisor keeps routing to the wrong specialist for "refund" requests, replace LLM routing for that intent with a keyword classifier. The supervisor becomes the fallback for ambiguous cases.

For rules that genuinely need to stay in the prompt, group them into named blocks and keep the count small. A flat list of fifteen rules is harder to follow than four blocks of three to four rules each.

## 4.3 Implicit state via conversation history

**1. What it looks like.** 
A specialist asks for information the user already provided. Or the specialist generates a generic output despite specific user data being present in the message history. Or two specialists disagree about a value because they parsed the same message differently.

**2. Why it happens.** 
Multi-agent state is the conversation history. Specialists do not share a typed object; they share a stream of messages, and each one parses what it needs out of free-form text. Parsing is LLM inference, with all the variance that implies.

A concrete pattern, assuming a third specialist `triage_agent` (an intake role that collects ticket context via `collect_ticket_info` before billing or technical takes action): the billing specialist's prompt says "use the ticket context from triage when processing the refund." The model interprets this literally: ticket context means a structured summary from the triage specialist. If triage ran but only persisted data via tool calls (no summary text), or if triage was skipped entirely and the user described the issue directly in their message, billing does not find what it expects and produces a generic "please provide your account details" reply.

**3. Mitigation.**

Make state explicit. Persist user data through tools that write to a typed store (database, structured file). Specialists read from the store via tools, not by parsing message history.

```python
@tool
def get_customer_profile(customer_id: str) -> dict:
    """Return the customer's stored profile: account tier, billing address, payment methods."""
    return db.fetch_profile(customer_id)
```

Billing now has a deterministic source of profile data. The conversation history is for conversation context, not for cross-agent communication.

This shifts complexity from prompt engineering to schema design, which is the better tradeoff. Schema bugs are easier to debug than prompt-interpretation drift.

## 4.4 Anti-fabrication and over-step

**1. What it looks like.** 
The supervisor, instead of routing to the triage specialist, asks the user for their account ID and issue description directly. The user replies. The supervisor then claims the ticket has been logged, even though no `collect_ticket_info` tool was ever called. The ticket exists nowhere.

A related variant: the triage specialist collects the issue details and, instead of stopping, also issues a refund that the billing specialist was supposed to handle. The refund is not recorded through `refund_charge` because billing never ran.

**2. Why it happens.** 
The supervisor and specialists are generative LLMs. They are trained to be helpful. When a user asks for a refund, the supervisor's model wants to help. If the next obvious step is "ask for account ID and invoice number", the model will ask, even though that is the triage specialist's job. If the triage specialist has the issue details, the model will issue the refund, even though billing is supposed to.

**Mitigation.**

Explicit anti-overstep rules in each prompt. The supervisor's rule:

```
ANTI-OVERSTEP: Do not ask intake questions yourself (account ID, issue type,
contact details). Route to the triage specialist instead.
```

The triage specialist's rule:

```
SCOPE: Your only job is to collect ticket information via the collect_ticket_info
tool and summarize what was collected. Do not issue refunds, take resolution
actions, or escalate on your own.
```

These rules work imperfectly. Different models follow them with different fidelity, so the same prompt can produce conforming behavior on a tool-reliable model and over-stepping on a smaller one.

A more durable defense: tool design. If the only way to "save" data is to call `collect_ticket_info`, and the specialist's prompt forbids replying without calling tools, the model has fewer paths to the over-step failure mode.

## 4.5 Tool-use reliability is model-dependent

**1. What it looks like.** 
The supervisor decides correctly to route to a specialist (the reasoning trace shows "I should call transfer_to_triage") but no tool call is emitted. The supervisor replies with text instead. Or the triage specialist correctly identifies four ticket fields to save but only emits one `collect_ticket_info` call. The remaining three are mentioned in the response text but never persisted.

**2. Why it happens.** 
Tool calling is a separate capability from text generation. Some models, especially smaller or older ones, have a noticeable gap between deciding to use a tool and actually emitting the structured tool call. Reasoning models can produce long chain-of-thought that concludes "I will call X" and then skip emission.

**3. Detection.** 
Compare the tool calls emitted to what the prompt and reasoning suggest should happen. Run cross-model tests on the same prompt and input: if model A emits the tool call and model B does not, the issue is in the model, not the prompt.

**Mitigation.**

If the budget allows, use a tool-reliable model for any agent whose value depends on tool emission. Frontier models from major providers (Anthropic, OpenAI, others) are reliable in this dimension at the time of writing; smaller models trade reliability for cost.

If the budget does not allow that, design tools so the model gets immediate feedback when it skips. A `collect_data` tool that returns "data not saved" if called with empty arguments is more useful than one that silently accepts anything. The model's next reasoning step has something to react to.

## 4.6 The supervisor's finalize step

**1. What it looks like.** 
A turn that should be one routing decision and one specialist reply ends up with three or four LLM calls. The supervisor runs once to route, the specialist runs (possibly with multiple tool calls), and then the supervisor runs again to "finalize", wrapping or forwarding the specialist's reply. Sometimes the supervisor runs a third time after that.

**2. Why it happens.** 
This is how the supervisor pattern is wired. After a specialist completes, control returns to the supervisor. The supervisor's prompt includes routing logic, so it evaluates whether more routing is needed. Even a STOP rule still requires an LLM call to read the rule and decide to stop.

The cost is real, the finalize step typically runs a full prompt and produces a non-trivial output. For a supervisor running on the same model as the specialists, this is ~1× the specialist cost again.

**3. Mitigation.**

Use a smaller model for the supervisor. Routing is cheaper reasoning than content generation. A small/fast model for the supervisor paired with a larger model for the specialists is a common production split (the *router-worker split*).

Some frameworks support skipping the finalize step entirely (return the specialist's reply directly). Check whether yours does. The tradeoff is losing the supervisor's ability to normalize formatting, enforce language rules, or catch anti-fabrication violations in the specialist's output.

Prompt caching helps for the part of the cost that is fixed: the supervisor's system prompt and persistent context. Major providers offer it (Anthropic, OpenAI, others) with discounts on the cached portion of subsequent calls, which is meaningful when the supervisor runs twice per turn.

## 4.7 Audience ambiguity in specialist output

**1. What it looks like.** 
A specialist's reply reads like a message to the end user, with greeting, conversational tone, and a question directed at the user. But the immediate reader of that reply is the supervisor (or another specialist), not the user. The supervisor either forwards verbatim and the user sees a reply that almost works, or worse, the supervisor reads the specialist's question as a question to *itself* and routes again, or wraps the reply in its own voice and double-greets the user.

A concrete pattern: triage specialist replies "Hi! I've logged your ticket — account ACC-1042, issue: payment failed on checkout. Want me to escalate to billing?" The supervisor consumes this. Two failure modes are possible:
- Supervisor treats "Want me to escalate to billing?" as the user's intent and routes to billing, processing a refund the user never asked for.
- Supervisor's finalize step adds its own greeting on top: "Hi! Your ticket is logged: account ACC-1042, payment failed on checkout. Want me to escalate to billing?" The user sees double-greeting and an offer for a step the supervisor decided unilaterally.

**2. Why it happens.** 
LLMs are trained on conversational data where the receiver is a human. Default reply style is human-facing: greeting, polite closing, offer of next steps. The specialist does not know its reply will be parsed by another LLM before reaching the user. There is no natural cue in the prompt or the runtime to make it write differently.

**3. Detection.** 
Read specialist outputs and ask: "if the supervisor reads this verbatim, what does it think the user wants?" If the answer is ambiguous or wrong, the specialist is producing user-facing prose where structured status would serve better.

**4. Mitigation.** 
Four options, from lightest to heaviest.

**4.1. Limit context flow with `output_mode="last_message"`.** 
Reduces how much of the specialist's chatter the supervisor sees. Does not solve the problem (the last message is still user-facing prose) but reduces the surface area for misinterpretation.

**4.2. Tell the specialist who its reader is.** 
Add to the specialist's prompt. Pick the variant that matches the specialist's role.

For specialists whose reply goes directly to the user (e.g., billing or technical resolving a request):

```
Your reply is forwarded directly to the end user by the supervisor. Reply concisely
to the user. Do not address the supervisor. Do not include greetings, sign-offs, or
offers of further help — those are added by other parts of the system.
```

For specialists whose reply is consumed by the supervisor for routing decisions (e.g., a triage step before an action specialist runs):

```
Your reply is read by a coordinator agent, not the user. Output a brief structured
status: what was done, what data was collected, what (if anything) requires the user.
The coordinator will compose the user-facing message.
```

**4.3. Two-tier output via structured response.** 
Specialist returns a structured object with separate user-facing and internal fields:

```python
{
  "user_message": "Ticket logged: account ACC-1042, payment failed on checkout.",
  "internal_status": "triage_complete",
  "next_step_suggestion": "billing"
}
```

The supervisor consumes `internal_status` for routing decisions and forwards `user_message` to the user. Cleanly separates the two audiences. Requires framework support for structured output or a custom response parser.

**4.4 Handoff payloads with explicit context.** 
For cross-specialist handoffs, pass structured payloads instead of relying on the receiver to parse prose. See [Section 2.4 #3 Handoff payloads](02-supervisor-pattern.md#24-variations) for the mechanics; it eliminates audience ambiguity at the seam where it most often breaks.

