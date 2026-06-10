---
name: legacy-test-fallout-triage
description: >-
  Triage existing (legacy) test failures after a behavior-changing Roslyn compiler edit. Classify
  each failure into buckets before editing expectations, decide true-bug vs stale-test, and (for
  large batches) hand parallel fix subagents a shared briefing plus per-failure dumps. Use when a
  compiler change makes many pre-existing tests fail, when CI reports failures on a feature PR, or
  when deciding whether a failing test found a real bug or just needs updating.
disable-model-invocation: true
---

# Legacy Test Fallout Triage

A behavior-changing compiler edit makes a pile of pre-existing tests fail. Do not blindly update
expected results. Classify, decide bug-vs-stale, then fix consistently.

## 1. Collect and bucket

Run the affected suites, dump failures to files, and sort each failure into a bucket. The buckets
from relaxed-modifier-ordering (45 failures across Syntax/Symbol/Emit) generalize:

| Bucket | Meaning | Action |
|--------|---------|--------|
| **A: ordering/availability now legal** | The old error no longer fires in preview | Pin the test to `TestOptions.RegularNN` (the last pre-preview version), swap the old error for `ERR_FeatureInPreview` + the feature name, and add a clean `RegularPreview` run |
| **B: duplicate diagnostic** | Same error now reported once instead of twice | Remove the redundant expected diagnostic |
| **C: parse-tree shape changed** | Cleaner/different tree | Update the `UsingTree` / `N(SyntaxKind.*)` assertions |
| **D: cross-feature cascade** | Failure owned by a later phase (e.g. `ref partial`) | Accept/park until that phase; do not over-fit the cascade now |
| **E: ambiguous** | Intent unclear or possibly a real regression | Skip and escalate to the user |

Adapt the bucket names per feature, but keep the principle: **decide the category before editing.**

## 2. Bug vs stale test

For each failure (especially CI failures on an open PR), read the test's intent and trace the
relevant parser/binder path before changing anything. Conclude **true bug** or **stale test that
needs updating**, and report that conclusion to the user.

> "See if these have found a true bug, or are just missed tests that need updating."

- True bug: fix the product code (new commit), keep/repair the test.
- Stale test: update expectations per its bucket. If a "misplaced" test now parses legally, give it
  a genuinely-invalid construct so it still tests what it intended.

## 3. Briefing handoff for large batches

When the batch is big, do not fix serially in the parent. Write a shared briefing and fan out
parallel fix subagents.

- `BRIEFING.md`: one-paragraph summary of the compiler change, the bucket definitions, **example
  before/after transforms per bucket**, the csproj paths, the verification commands, and tone rules
  (no emojis, no unrelated refactors, no AI-isms, no em-dashes).
- One dump file per failing test: `<FullyQualifiedName>.txt` with the failure output.
- Subagent prompt: "Read the shared briefing at `<path>` FIRST." Split into batches by suite to avoid
  merge collisions (e.g. Symbol+Emit, then Syntax, then the remainder).
- Each fix is a focused commit; keep existing-test updates as a separate commit from product changes
  and from new dedicated tests (see `roslyn-stacked-pr-feature`).

## 4. Verify

Rebuild and re-run the affected suites. For Category A, confirm both the pinned `RegularNN` (error)
and the `RegularPreview` (clean) runs pass. Re-run the adversarial suite if product code changed.
