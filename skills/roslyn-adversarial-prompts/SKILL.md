---
name: roslyn-adversarial-prompts
description: >-
  Verbatim prompt templates for spinning up adversarial validation subagents on
  a Roslyn/C# compiler feature: spec-vs-impl, spec-vs-tests, impl-vs-tests,
  code-simplification, test-suite mining, cross-feature, diagnostic/error-recovery,
  IL/allocation, feature-matrix, IDE-coverage, speclet/standardese, cleanroom, and
  PR-author auditors. Use when you need to write or launch hostile review agents
  against compiler code, tests, or a language spec. Pairs with
  roslyn-adversarial-orchestration (how to fan them out and triage findings).
disable-model-invocation: true
---

# Roslyn Adversarial Prompt Library

A catalog of ready-to-paste prompts for hostile validation subagents. For *how many*
to run, *when*, and how to *triage* their findings, use `roslyn-adversarial-orchestration`.

## The posture (non-negotiable preamble)

Every adversarial agent opens with this stance. Use it verbatim; it is the point.

> You are an adversarial reviewer. Assume this was written half-heartedly by an agent
> more interested in pleasing than being correct. Be aggressive, anal, nitpicky, and
> distrusting. Do not give the code the benefit of the doubt. Where the spec is
> ambiguous, the impl is wrong unless proven otherwise. Find the problems.

Variants the user has used interchangeably (pick one, keep the spirit):
- "Assume the implementation may be completely wrong."
- "You are a senior compiler engineer reviewing a PR from a stranger. Assume the code is wrong."
- "Assume the test suite is INSUFFICIENT until proven otherwise."
- "Assume common C# patterns are silently broken until you prove otherwise."

## Two rules that override everything

1. **Do not bias the agent.** Never tell it which file, API, or construct to attack.
   Give it the branch + diff + spec + tests and let *it* decide the attack surface.
   > "look at all my changes and figure out what's wrong yourself"

   Telling an agent "look at `IsTrueIdentifier`" just projects your own blind spots
   onto it. The whole value is the cases *you wouldn't think to prompt for*.

2. **Different mandate per agent.** Run several with orthogonal missions so they
   cannot converge on the same blind spot. Never one "general reviewer."

## Shared deliverable contract (append to every prompt)

> Read-only. Do not edit anything.
>
> Deliverable: a punch-list of concrete, actionable findings. Every finding must cite
> `file:line` and quote the offending text (or the missing spec rule). No vague worries.
> Do NOT summarize what the code does or what is correct, only what is wrong, risky, or
> untested. Severity each finding: BLOCKER / HIGH / MEDIUM / LOW / NIT. Do not pad with
> nits. Be exhaustive: I would rather see 30 real issues than 5 cherry-picked ones.

Then add a `## Areas audited clean` section request so you know what was actually looked at.

## Report schema (request this exact shape)

```
## Executive summary
<one paragraph>

## Findings
### 1. [SEVERITY] <short title>
- Spec / expectation: <quote or rule>
- Code / test: `file:line`, <quote>
- Problem: <concrete technical objection>
- Suggested fix: <specific change or action>

## Areas audited clean
- <area>: <why it's fine>
```

Role-specific sections to swap in:
- impl-vs-tests → `## Uncovered code paths` + `## Weak / unfaithful tests`
- spec-vs-tests → `## Untested spec rules` + `## Tests not backed by spec` + **`## Spec / test matrix`** (the single highest-value artifact, always request it)
- consumer/interaction audit → `## Confirmed breaks` / `## Fragile but currently-gated` / `## Safe`
- feature-matrix → a table with axes as columns + `Predicted behavior` + `Risk / test` columns

## The role catalog

Deploy a subset per phase (see `roslyn-adversarial-orchestration`). Full prompt text for
each is in [templates.md](templates.md).

| Role | Hunts for | Typical phase |
|------|-----------|---------------|
| **spec-vs-impl** | Spec rules the code doesn't enforce; silent deviations | binding, lowering |
| **spec-vs-tests** | Normative rules with no test pinning them (+ spec/test matrix) | every phase |
| **impl-vs-tests** | Uncovered branches; tests whose asserts don't prove the claim | every phase |
| **code-simplification** | Duplication, missed reuse, over-engineering in NEW code | after code lands |
| **test-suite mining** | "Do we have the `await?`/compound/chained analogue of this existing test?" | late, per layer |
| **cross-feature comparison** | Parity gaps vs adjacent features (`await using`, `?.`, collection exprs) | syntax, binding |
| **diagnostic / error-recovery** | Wrong/missing diagnostics, squiggle spans, cascading errors on malformed input | binding |
| **IL / allocation-quality** | State-machine size, spilling, `pop` vs temp local, redundant boxing | emit |
| **feature-matrix** | Every receiver × operator × container × context cell; adversarial compositions | binding, emit |
| **IDE-coverage** | Refactorings/analyzers/completion/quick-info pattern-matching the changed syntax | late |
| **speclet / standardese** | Vernacular vs standardese, redundancy, broken §refs, internal inconsistency | spec |
| **cleanroom** | Fresh agent reimplements from spec alone; diff vs real impl exposes ambiguity/bugs | pre-PR |
| **PR-author adversary** | Worst-case senior-reviewer comments, to pre-empt human review | before leaving draft |

## Hard-won meta-rules (the user corrected agents on these repeatedly)

- "The compiler team rarely thinks something is too contrived to test for." Don't self-censor edge cases.
- For success cases, **execute** tests and show real output, don't only assert diagnostics.
- Don't pin buggy behavior as "expected." A whole audit role exists just to catch tests that lock in bugs (see `hidden-bug auditor` in templates.md).
- Run the agents. Don't claim a phase is done without them. ("what did your adversarial agents say here?")
- Keep test comments clean: no quoting compiler code, no line numbers, no narration.

## Additional resources
- Full per-role prompt text: [templates.md](templates.md)
- Fan-out counts, severity triage, sign-off gates: use the `roslyn-adversarial-orchestration` skill.
