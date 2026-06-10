---
name: roslyn-language-design-principles
description: >-
  Core design principles for implementing a C# language feature in Roslyn: parse permissively on
  all language versions and enforce semantics strictly in the binder (gated via
  CheckFeatureAvailability), and reuse existing binder/lowering infrastructure instead of
  duplicating logic for new syntactic forms. Use when deciding where to enforce a feature
  (parser vs binder), how to gate language versions, or how to structure a feature's binding/lowering.
disable-model-invocation: true
---

# Roslyn Language-Design Principles

Two foundational design rules for this user's Roslyn feature work.

## 1. Parse permissive, bind strict

Parse the new shape on **all** language versions; reject/gate semantics in the **binder**.

- The parser should accept the new ordering/form regardless of langver, "code that was an error to
  parse before is fine to now parse legally."
- Language-version gating lives in the binder via `CheckFeatureAvailability` →
  `ERR_FeatureInPreview` (CS8652), not in the parser.
- New preview feature → `MessageID.IDS_Feature...` mapped to `LanguageVersion.Preview`.

> "We want to always support parsing in the different orders (no matter the lang version). but the
> binder may reject some locations if on a prior lang version (not on 'preview')."

Benefits: clean error recovery, good langver diagnostics, no parser ambiguity churn. When a
construct could change the parse tree for existing code (contextual-keyword disambiguation like
`record`/`union`), that is the rare case where langver affects parsing, and it stays documented in
preserved rationale comments.

### The "two checks" shape
When adding a new binding path beside an existing one, structure it so the existing route runs,
then the IDS feature-availability check, then the new path, so prior behavior is preserved and the
new behavior is cleanly gated.

## 2. Reuse, don't duplicate

Extend existing binder/lowering paths rather than writing parallel logic for the new form.

> "trying as much as possible to not write code that duplicates existing logic. instead we should
> reuse existing functions as much as possible (untouched if possible), or tweak them slightly so
> they work for both `await e` and `await? e`."

- Prefer a boolean/parameter on the existing method (e.g. `BindAwait(..., bool isNullConditional)`)
  over a parallel `BindNullConditionalAwait`.
- Extract a shared helper when two forms converge (e.g. `ComputeConditionalAccessResultType` shared
  by `?.` and `await?`); reuse `BindConditionalAccess*` / conditional-access lowering infrastructure
  for new null-propagating forms.
- Thread a flag through lowering as a compile-time dependency without prematurely changing lowering
  (e.g. pass `isNullConditional: false` from the binding PR; real lowering lands in the emit PR).
- The code-simplification adversary (see `roslyn-adversarial-prompts`) exists to catch missed reuse,
  run it after the implementation lands.

Keep new code **as close as possible to the original Roslyn code**; when an audit suggests a new
helper, prefer inlining to match the surrounding style unless the shared helper genuinely pays off.
