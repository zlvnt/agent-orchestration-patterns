# 6. Evaluation

Single-agent systems can often ship without a formal eval suite, since a handful of manual test conversations and production logs catch most regressions. Multi-agent systems do not get that margin: the number of failure modes, the coupling between routing and content quality, and the per-turn cost together make a formal eval suite necessary rather than optional.

This section covers what to measure, how to design test cases, how LLM-as-judge fits in, and how to read the results without fooling yourself.

## 6.1 What to measure

A multi-agent turn produces five things worth measuring independently:

- **Routing accuracy.** Did the supervisor pick the right specialist?
- **Tool correctness.** Did the specialist call the right tools, with the right arguments?
- **Task completion.** Did the user's request get fulfilled (data saved, plan generated, ticket created)?
- **Response quality.** Is the final reply useful, accurate, in the right language and tone?
- **No-loop / efficiency.** Did the turn complete without redundant handoffs or runaway recursion?

The five are not redundant; each catches a different failure surface. A turn can route correctly, call the right tool, and produce a polished reply while still failing task completion (the tool ran with bad arguments). Or it can complete the task with the right tool and reply while the routing was technically wrong (the message went to a specialist whose tools happened to overlap with what was needed).

Optional, depending on workload:

- **Latency** (p50, p99 per turn)
- **Token cost** per turn, broken down by agent
- **Number of LLM calls** per turn

These do not need a judge. They come from the trace.

## 6.2 Test case design

A test case is a tuple: input message, expected route(s), expected tools, optional reference response. The set of test cases is what you run the system against repeatedly as you iterate.

Two principles for designing the set.

**1. Tier cases by criticality.** 

Not every test case is worth the same weight. A four-tier system works:

- **Tier 1: critical paths.** The most common, highest-stakes user requests. If these fail, the system is broken. Examples: "refund invoice INV-1042", "the API is returning 500s". Should pass at high reliability.
- **Tier 2: common variations.** Requests that look similar to tier 1 but differ in phrasing or completeness. Tests robustness. Examples: "my last bill was wrong" (no invoice ID), "site won't load" (vague technical signal). Should pass at acceptable reliability.
- **Tier 3: edge cases.** Specific failure modes you want to defend against. Ambiguous input, multi-intent messages, off-topic chatter, prompt injection. Lower pass rate is acceptable but failures should be understood.
- **Tier 4: known failures.** Cases the system under test currently fails on. Tracked to confirm fixes don't regress, and to flag when behavior changes unexpectedly.

A practical case set is 15–30 cases, with most weight on tier 1 and 2.

**2. Cases are not interchangeable.** 

Each case should have a clear hypothesis: "this case tests X". If two cases test the same thing, drop one. If a case is in the set "for completeness", it is not pulling its weight.

A test case in code:

```python
from dataclasses import dataclass
from typing import Literal

@dataclass
class TestCase:
    id: str
    tier: Literal[1, 2, 3, 4]
    input: str
    expected_route: list[str]       # e.g. ["supervisor", "billing_agent"]
    expected_tools: list[str]       # e.g. ["lookup_invoice", "refund_charge"]
    description: str                # what this case tests
    reference_response: str | None  # optional, for quality judge

cases = [
    TestCase(
        id="case_01",
        tier=1,
        input="Refund invoice INV-1042",
        expected_route=["supervisor", "billing_agent"],
        expected_tools=["lookup_invoice", "refund_charge"],
        description="Direct refund request with explicit invoice ID",
    ),
    # ...
]
```

## 6.3 The metrics in detail

### Routing accuracy

Routing accuracy is the simplest of the five to compute. The metric compares the agents actually visited to the agents expected:

```python
def routing_accuracy(actual_route: list[str], expected_route: list[str]) -> float:
    expected_set = set(expected_route)
    actual_set = set(actual_route)
    if not expected_set:
        return 1.0
    return len(actual_set & expected_set) / len(expected_set)
```

The version above checks set membership: which agents appeared, ignoring their order and any repetition. A stricter version instead requires the visited sequence to match the expected sequence exactly, which catches more failures but flags equivalent re-orderings as wrong, so the right choice depends on whether order matters for the workload.

### Tool correctness

Compare actual tools called to expected tools, using the same strict (sequence match) or lenient (set membership) choice as routing. Checking the arguments themselves splits into two levels. Value-level checking compares the exact argument values, which is fragile because values can legitimately vary across runs (timestamps, generated IDs). Schema-level checking instead verifies that the required arguments are present with the right types, which is more robust because it tolerates that variation.

### Task completion

The hardest of the five to measure mechanically, with two viable approaches.

**1. Effect-based.** 
After the run, query the database (or whatever the side effect target is) and verify the expected change. Did `refund_charge` actually persist? Did `create_ticket` produce a ticket ID? This is the most reliable signal but requires the test to run against a real or test database.

**2. LLM-as-judge.** 
A separate LLM reads the test case and the agent's response, and scores whether the task appears to have been completed. Cheaper to set up; less reliable than effect-based.

Use effect-based when feasible. Use judge as fallback for cases where the side effect is hard to verify (a diagnosis was "produced", what does that mean concretely?).

### Response quality

This is what most needs LLM-as-judge. The judge prompt receives the user's input, the agent's response, and a rubric, then returns a score and a reason. A typical rubric:

```
Score the agent's response on these dimensions (0.0 to 1.0 each):
- Relevance: does it address what the user asked?
- Specificity: is it actionable, with concrete numbers/details?
- Language match: is it in the same language as the user's input?
- No fabrication: does it avoid claiming things that did not happen?

Overall score is the minimum of the four (worst-dimension scoring).
```

DeepEval and LangSmith both have built-in judges and let you write custom ones. The judge model should be at least as capable as the agent being evaluated, since a weaker model judging a stronger model's output is unreliable.

### No-loop / efficiency

Count handoffs in the actual trace. If the same agent is invoked more than once in a single turn (excluding the supervisor's initial and finalize calls), score 0, otherwise 1.

```python
def no_loop_score(actual_route: list[str]) -> float:
    # Strip supervisor (it appears twice by design)
    specialists = [r for r in actual_route if r != "supervisor"]
    if len(specialists) != len(set(specialists)):
        return 0.0
    return 1.0
```

This uniqueness check is a blunt proxy: it scores any repeated specialist as a loop, which over-penalizes legitimate cases where the same specialist is genuinely needed twice in one turn. A truer loop signal caps the total number of handoffs, or flags repeated handoffs that produce no change in state, rather than treating every repeat as failure.

## 6.4 LLM-as-judge setup

The judge is a separate LLM call per metric per test case. With 15 cases and 3 judge-evaluated metrics, that is 45 judge calls per eval run. Costs add up.

A minimal judge in Python (using DeepEval's interface, but the pattern is universal):

```python
from deepeval.metrics import BaseMetric
from deepeval.test_case import LLMTestCase

class ResponseQualityMetric(BaseMetric):
    def __init__(self, judge_llm, threshold: float = 0.7):
        self.judge_llm = judge_llm
        self.threshold = threshold

    def measure(self, test_case: LLMTestCase) -> float:
        prompt = f"""
        Score this agent response on relevance, specificity, language match,
        and no-fabrication. Return a JSON object with each dimension scored
        0.0 to 1.0.

        User input: {test_case.input}
        Agent response: {test_case.actual_output}
        """
        result = self.judge_llm.invoke(prompt)
        scores = parse_json(result)
        self.score = min(scores.values())
        return self.score

    def is_successful(self) -> bool:
        return self.score >= self.threshold
```

Three things matter for judge reliability:

**The judge prompt is its own surface to test.** Run the judge on a small set of hand-graded responses. If the judge disagrees with humans systematically, refine the prompt before trusting it for the full eval.

**Use structured output.** Free-form judge responses are harder to parse and easier to misalign. JSON schema or function calling is the difference between a robust eval pipeline and one that breaks on every model update.

**Calibrate against humans periodically.** Pick 10 cases, have a human grade them, compare to judge scores. If the agreement rate drops over time (model drift, prompt changes), adjust.

## 6.5 Running the eval

A full eval run does the following per test case:

1. Invoke the agent with the test input
2. Collect the trace (route taken, tools called, final response)
3. Compute mechanical metrics (routing, tool, no-loop)
4. Run judges for response quality and (if needed) task completion
5. Aggregate scores per case and across the suite

LangSmith handles most of this if you instrument the agent's invocations. Without LangSmith, the same setup can be built with a small runner script that logs traces to JSON.

```python
async def run_eval(graph, cases: list[TestCase]):
    results = []
    for case in cases:
        trace = await invoke_with_tracing(graph, case.input)
        scores = {
            "routing_accuracy": routing_accuracy(trace.route, case.expected_route),
            "tool_correctness": tool_correctness(trace.tools, case.expected_tools),
            "no_loop": no_loop_score(trace.route),
            "response_quality": await response_quality_judge(trace.final, case.input),
            "task_completion": await task_completion_judge(trace, case),
        }
        results.append({"case": case.id, "tier": case.tier, "scores": scores})
    return results
```

Aggregate by metric and by tier. Tier 1 averages indicate whether the system is fit for production, tier 3 averages show how robust it is under stress, and the per-metric breakdown points to where the next fix should go.
## 6.6 Reading results

The single most useful framing: track all five metrics across iterations, not just one.

A change that improves routing accuracy from 0.60 to 0.82 looks like progress. The same change that drops task completion from 0.88 to 0.64 and pushes per-turn cost up 45% is a regression in disguise. Without all five metrics, you do not see the trade.

Common patterns:

**Routing up, completion down.** A new prompt rule made the supervisor route more accurately, but the now-correct routing introduces a multi-handoff sequence that the system does not finish reliably. Fix the routing or fix the multi-handoff, but know which one you are choosing.

**Tool correctness flat across iterations.** Suggests the issue is model-level, not prompt-level. The model is not emitting tool calls reliably. Either swap models, or restructure the work so fewer tool calls are needed per task.

**No-loop drops on a specific case.** Almost always traceable to one prompt clause that triggers the loop. Read the trace for that case and remove or tighten the clause.

**Quality up, cost up.** Sometimes this is the right trade, and the eval makes the cost visible so the decision is informed instead of incidental.

## 6.7 What evaluation does not give you

The eval suite measures behavior on the test cases, and says nothing about production traffic that does not resemble them.

Chasing exhaustive coverage by adding cases does not converge; sampling does. Pull a random 20 production conversations weekly, grade them by hand or with the judge, and look for cases where production behavior diverges from the test set. Add cases for the failure modes you find, and keep the test set bounded (15–30) but representative.

A test suite that grows past 100 cases without curation becomes dead weight: maintenance cost rises, signal-to-noise drops, and full runs get slow enough that you stop running them. Keeping the suite small and curated beats letting it grow large and stale.
