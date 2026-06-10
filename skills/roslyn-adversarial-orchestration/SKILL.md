---
name: roslyn-adversarial-orchestration
description: >-
  Playbook for orchestrating adversarial validation subagents on a Roslyn/C# compiler
  feature: when to launch, how many to fan out in parallel per phase, the severity
  taxonomy, carve-outs for deferred work, and the union/dedupe/triage/sign-off protocol
  for turning findings into fixes. Use when running hostile review agents against a
  feature implementation, deciding how to deploy them, or triaging what they return.
  Pairs with roslyn-adversarial-prompts (the actual prompt text per role).
disable-model-invocation: true
---

# Roslyn Adversarial Orchestration Playbook

How to deploy and triage the hostile review agents whose prompts live in
`roslyn-adversarial-prompts`. The core idea, in the user's words:

> "Don't trust your own work, spawn adversaries whose only goal is to prove you wrong,
> run them in parallel from different angles, and treat their findings as the baseline
> before asking for human review. Spend more cycles disbelieving the work than producing it."

## When to launch

- **After each phase passes its own tests**: syntax, binding, lowering, semantic-model/IOperation, emit.
- **Again after any bugfix commit** (don't skip the re-audit).
- **After a spec merge** (re-audit code against the now-merged spec).
- **Before a PR leaves draft** (PR-author + optionally cleanroom).
- **Never auto-proceed**: the human reviews between phases; adversaries run *before* you call a phase done.

## How to launch

- Read-only `explore` subagents. Inherit the parent model unless told otherwise.
- **Parallel by default**: fan out 4-7 at once, each with an orthogonal mandate.
- **Self-contained briefs**: branch name + HEAD SHA + spec path + changed-file list +
  scope fence + carve-out list + report schema. The agent must not need parent chat history.
- **Propose the split, get a sanity-check, then launch.** ("What sort of subagents do you
  plan on creating? ... Ready to launch all six if the split looks right, want me to go?")

## Per-phase deployment

| Phase | Fan out (typical) |
|-------|-------------------|
| **Syntax** | 2 parsing adversaries (missing-cases + bugs), cross-feature parity, roundtrip/tree-invariant (PublicAPI, SyntaxKind switches). Scope-fence to SYNTAX ONLY. |
| **Binding** | spec-vs-impl, spec-vs-tests, impl-vs-tests, cross-feature, diagnostic/error-recovery. |
| **Lowering** | code-simplification, impl-vs-tests, spec-vs-impl, spec-vs-tests (the user's canonical 4-up). |
| **Emit** | IL/allocation-quality, hidden-bug (emit/IL), feature-matrix (state-machine vs runtime-async). |
| **Semantic model / IOperation** | SM-surface survey, IOperation survey, then adversarial SM + IOperation audits. |
| **Late / pre-PR** | test-suite mining (one per adjacent corpus), IDE-coverage, speclet/standardese, PR-author, optionally cleanroom. |

Use the **feature impact-surface** discovery skill (`roslyn-feature-impact-surface`) to find
the consumer/IDE/intersection targets the interaction-audit and IDE-coverage agents should cover.

## Severity taxonomy (enforce it)

`BLOCKER` (feature incorrect) > `HIGH` > `MEDIUM` > `LOW` > `NIT`. Spec audits sometimes use
`BUG / DIVERGENCE / GAP / OBSERVATION` or `BUG / GAP / NIT` (BUG = wrong compile outcome,
GAP = correct but untested, NIT = comment/wording). Tell agents **"do not pad with nits."**

## Carve-outs (always include)

List deferred-phase work in every brief so agents don't fault it, e.g.:
- "Constant folding returns null, follow-up PR."
- "IOperation does not expose `IsChainedRelational` yet."
- "IDE tests are a separate PR."
- "IL-verify failure on asymmetric-widening chains is pinned with `Verification.FailsILVerify`."

Confirm the carve-outs actually appeared in the reports (agents that ignore them are noisy).

## Triage protocol (turning findings into fixes)

1. **Wait for all agents**, then **union** their findings.
2. **Dedupe** overlapping items; note **convergence** (2+ agents flagging the same thing → prioritize).
3. **Synthesize by severity, with source attribution**: "(spec-vs-impl)", "(consumer audit)",
   so it's clear what converges and what each auditor uniquely saw.
4. **Push back on false positives** with a one-line rationale; do not silently drop them.
5. **Never dismiss a finding as "out of scope" without proposing a follow-up.**
6. **Propose an order of attack** (First / Second / Third batches) and **get sign-off before fixing.**
   > "Sharing for your sign-off rather than fixing silently."
7. **Fix in ordered batches**, one concern per commit; **attribute** the source in the commit message
   ("surfaced by the spec-vs-code auditor subagent").
8. **Re-run the test suite**; **re-audit** after fixes.
9. If an agent's output was **truncated**, resume it: "findings 1-24 appear truncated; resend in the
   same format; do not repeat 25-33."

## Synthesis format to the user

```
All <N> subagents returned. Grouped by severity and by where each finding came from:

## Blockers
1. <title> (spec-vs-impl), file:line, <one line>
## High
...
## Checked and disagree (false positives)
- <finding>, <why it's fine>
## Carve-outs confirmed clean
## Suggested order of attack
First: ...  Second: ...  Third: ...
Want me to proceed with the first batch, or push back on any?
```

## Anti-patterns (all observed and corrected)

- One "general reviewer" instead of orthogonal mandates.
- Prescriptive checklists that stop the agent reading the diff itself (biasing).
- Parent writing tests/fixes without running the agents at all.
- Happy-path-only output with no malformed-input / error-recovery cases.
- Auto-applying audit fixes without a sign-off gate.
- Letting agents fault deferred work because no carve-out list was given.
