---
name: change-impact-review
description: Review a Git diff in an unfamiliar repository by tracing structural and behavioral impact, testing risk hypotheses with evidence, and reporting findings and uncertainty. Use for PR reviews, especially when downstream effects are unclear.
---

# Change Impact Review

Review **the impact of changes**, not only the changed lines. Work with unfamiliar languages and architectures. Prefer evidence from code, specifications, and tests over conjecture. Never imply completeness where the tooling or budget does not support it.

## Non-negotiable: visible review journal in the terminal

**Do not silently investigate and only show a final summary.** The human reviewer must be able to follow the review as it runs, challenge assumptions, and interrupt unproductive exploration.

At each meaningful step, print a short **decision record in the live terminal/session** (ordinary assistant output; not hidden reasoning, and not only in a generated file). Show:

1. **Context** — what was changed and what existing behavior/contract matters.
2. **Risk hypothesis** — what could go wrong, and why this change raised the concern.
3. **Probe** — exactly what file/symbol/reference/test/command is being examined and what evidence would settle the hypothesis.
4. **Observation** — what was actually found, with paths/lines or command results.
5. **Judgment** — confirmed issue, ruled out **within examined scope**, or still uncertain; **why**.
6. **Next step / stop reason** — why another branch is pursued or why exploration stops.

These are concise, factual **rationales and audit trails**, not a demand to expose private chain-of-thought. Never invent tools, results, or reasoning steps. If terminal output cannot be emitted as work progresses, state that limitation and provide the same sequence of records in the visible response, not just the final conclusion.

### Terminal message examples

```text
[IMPACT] Changed: getUser now reads a shared cache before repository lookup.
[RISK H1] Stale reads: updateUser may not invalidate the cache; relevant because previously reads always hit the repository.
[PROBE H1] Inspect cache writers/invalidation and updateUser callers; look for invalidation after successful writes.
[EVIDENCE H1] src/users/update.ts:42 updates the DB; no invalidation in its call path (checked ...). Cache TTL is configured at src/cache.ts:18.
[DECISION H1] OPEN — TTL limits duration but does not establish required freshness. Next: check explicit consistency contract and tests.
```

For a resolved case:

```text
[RISK H2] Potential duplicate processing when two requests overlap.
[PROBE H2] Checked transaction boundary and relevant concurrent test.
[EVIDENCE H2] ... (file:line, test command/result).
[DECISION H2] NOT REPRODUCED in tested scenario; not proof of absence under all schedules.
[STOP H2] Budget reached; record untested interleavings.
```

A visible journal should appear **during** analysis: after initial scoping, when a hypothesis is added, before/after a substantive probe, and when it closes or is deferred. For very large changes, group trivial probes but never hide the reason for prioritization or abandonment. Do not dump enormous tool outputs; summarize them with navigable references.

## 0. Establish the diff and capabilities

- Identify repo root, language/build/test manifests, project docs, and explicit base/head ref. If absent, determine a reasonable comparison and **state it**. Include staged/unstaged/untracked files only if in review scope.
- Inspect `git status --short`, `git diff --stat`, and exact relevant hunks. Account for renames, configuration, generated files, migrations, schemas and dependency changes.
- Detect available tools before use. Prefer **Serena MCP** for symbol-aware definition/reference search when its language-server project setup works. Use `rg` for cross-cutting textual links, dynamic registration, configs and fallback. Use **ast-grep** when syntax-pattern searching adds value and it is installed. Use native LSP facilities where available.
- Missing tool/indexing support is a coverage gap, **not** evidence of no callers.
- **Print [SCOPE]** exact comparison, notable changes, tools and limitations.

## 1. Interpret semantic changes

For each logical change, read the before/after implementation and surrounding context. Identify entry points, external contracts, call sites, state changes, async/error behavior, resource paths and existing tests. Distinguish intended behavior backed by docs from intent inferred from implementation.

**Print [CHANGE]** a brief before → after explanation and the most important unknown.

## 2. Generate and prioritize hypotheses

Use two lenses:

- **Change-driven:** Which callers, consumers, shared data, downstream systems or runtime behaviors can this change affect?
- **Property-driven:** Which local guarantees could be violated? Gather candidates from API schemas, type constraints, DB constraints, tests, documentation, ADRs and state transitions. Mark each as `explicit_contract`, `observed_behavior`, or `inferred_property`; existing implementation/tests are not automatically correct.

Consider correctness, reliability/concurrency, maintainability/readability, performance, security, and testability. Do not mechanically manufacture a risk for every category.

For each hypothesis define an ID, trigger, potential failure, affected areas, expected confirming/disconfirming evidence, and priority (impact/reach/uncertainty). **Print [RISK Hn]** with the concrete reason it is being investigated.

## 3. Adaptive evidence-driven exploration

Investigate the highest-value unresolved hypothesis first. Use these edges as needed:

| Trigger | Probe |
| --- | --- |
| Changed public symbol | Serena/LSP references, implementations, callers/callees |
| State/cache/DB writes | Reads, writes, invalidation, transaction boundaries, concurrent access |
| Async/stream/retry/cancel | Ordering, dropped errors, resource cleanup, shared state, partial responses, backpressure |
| API/schema/event | Producers, consumers, validators, serialization, compatibility |
| Auth/trust boundary | Sources, validation, authorization, sensitive sinks |
| Hot path | Frequency, loops, I/O, allocations, measurement or profiling |
| Test change | Preconditions, control of scheduling/environment, assertion relevance, counterexamples |
| Dynamic binding | String/config/DI/route/event searches with `rg` or ast-grep |

Before meaningful investigation **print [PROBE Hn]** with *what* and *why*. Afterwards **print [EVIDENCE Hn]** including precise file/symbol/line or test command/result. Always explain the finding's relationship to the hypothesis.

Avoid blindly expanding high-fan-out dependency graphs. Update priority as new evidence arrives. Serena is not a complete system dependency graph; string-based protocols, reflection and cross-service behavior require separate exploration.

## 4. Verification and stopping rules

A **verified finding** needs an identifiable execution path, a violated property/contract, and relevant code evidence or a reproducible test. Prefer targeted tests when feasible, never claim a check passed without running it. Performance allegations require caution: mark unmeasured impacts as hypotheses.

For each branch emit **[DECISION Hn]** and one stopping classification:

- `resolved`: supported evidence establishes impact or sufficiently rules out the specific hypothesis **within stated scope**;
- `merged`: same path already investigated elsewhere, with the linked evidence;
- `budget_limited`: time/token/exploration depth cap reached;
- `tool_limited`: indexing/unavailable tools/inaccessible dependency;
- `unknown`: cannot substantiate or refute.

Explicitly say **why the evidence is sufficient or insufficient**. Avoid “no problem” without a rationale. Distinguish “test passed for case X” from “race is impossible”. Unresolved high-risk hypotheses must be visible, not silently discarded.

Default first pass: changed hunks and context; bounded reference search for key symbols; up to ~5 prioritized hypotheses; deepen when evidence indicates risk. These are starting heuristics, not guarantees. Announce budget adjustments in the journal.

## 5. Final review and impact report

After live journal entries, output:

```markdown
# Change Impact Review
Scope: <exact diff / exclusions>
Tools and coverage: <working and unavailable>

## What changed
- <behavior before → after>

## Impact map
| Change | Affected path / property | Evidence | Status / stop reason |
| --- | --- | --- | --- |
| ... | ... | file:line | resolved / merged / budget_limited / tool_limited / unknown |

## Verified findings
- [severity] <failure, when and why it occurs>
  Evidence: <file:line + execution path + test/reproduction if run>
  Fix direction: <actionable>

## Investigated hypotheses and decisions
| Hypothesis | Why considered | What was checked | Observation | Judgment and reason |
| --- | --- | --- | --- | --- |
| H1 | ... | ... | ... | ... |

## Open questions and residual uncertainty
- <not explored, missing evidence/tools, next effective probe>

## Perspective coverage
- Correctness / Reliability / Maintainability / Performance / Security / Testability: <what was checked and omitted>

## Checks executed
- <exact commands and outcomes; state if none>
```

If no verified findings, write **“No verified findings within examined scope”**, never **“No issues.”** Final report must preserve the journal's decisions (including discarded hypotheses and their reasons), not just successful bug discoveries.

## Final self-check

- Is the comparison exact and visible?
- Could the human observe **why** each major risk was raised, **what** was checked, **what** was found, and **why** the conclusion followed **as the review ran**?
- Are contracts separate from guesses?
- Does every verified finding have evidence?
- Are unsearched paths and tool limitations explicit?
- Did a successful test get overstated as a proof of absence?
