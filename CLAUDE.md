# CLAUDE.md

Guidance for working on this repository.

## What this is

A practical reference guide on multi-agent orchestration patterns: supervisor-focused, with plan-and-execute, hierarchical, and peer-to-peer variants in the appendix. This is a prose and markdown project, not a codebase. It is derived from a real project (the Health Intelligence Agent), whose retrospective ships here as `LESSONS_LEARNED.md` and whose worked example is pointed to in Section 7.

Structure:
- `README.md`: index and entry point (the guide's table of contents)
- `01`–`07` chapters: the guide itself
- `appendix-patterns.md`: pattern variants and when to reach for them
- `LESSONS_LEARNED.md`: the case-study retrospective behind the guide

## Writing style

This is the main constraint, since the repository is almost entirely prose. The guide reads as a practitioner reference: declarative, sparing with words, free of AI-typical phrasing. When editing or adding prose:

- **No AI-ish preamble or closers.** Open with substance, not "Let's dive into" or "Here's how X works". Drop closing summaries that restate the section, and fillers like "it's important to note" or "interestingly".
- **No staccato uniformity.** Avoid runs of short declarative sentences ("X is Y. Z is W."), single-word sentences, parallel repetition ("The fix is X. The fix is Y."), and closing one-liners that restate the obvious. Vary sentence length. When fixing staccato, dissolve the short punchy sentence into its neighbor with a colon or connector; do not just move it elsewhere.
- **No em dash.** Use a colon, a connector, or split the sentence. Label separators follow the same rule: write "Tier 1: critical paths", not "Tier 1 (em dash) critical paths".
- **Concise by default.** Every sentence should change what the reader knows.
- **Claims.** In the instructional chapters (01 through 06), refer to providers (Claude, and so on) rather than specific model versions, and avoid performance numbers that cannot be verified. In the Case Reference (07) and `LESSONS_LEARNED.md`, naming the actual models and eval results is fine, because those sections document a specific real project.
- **Framework framing.** Code examples use LangGraph because it is explicit about how the pattern works, but the patterns translate to other frameworks. Keep that framing; this is not a framework comparison.

## Editing conventions

- Surgical changes: touch only what the task needs, match the existing voice and formatting, and leave prose that is not broken alone.
- Discuss structure and approach before any large rewrite.

## Owner

Ted (GitHub: zlvnt). Native Bahasa Indonesia, conversational English. Discussion in Bahasa is welcome; the guide content itself stays in English.
