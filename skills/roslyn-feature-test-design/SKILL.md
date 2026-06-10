---
name: roslyn-feature-test-design
description: >-
  Strategy for designing the test suite of a Roslyn/C# language feature: build exhaustive
  receiver x operator x container x context matrices, mine existing analogue tests, execute
  success cases (not just assert diagnostics), keep tests in separate files, and consolidate
  to confidence rather than raw exhaustiveness. Use when planning or writing tests for a new
  C# compiler feature, deciding test coverage, or organizing compiler test files.
disable-model-invocation: true
---

# Roslyn Feature Test Design

How to decide *what* to test for a language feature and how to organize it. For *hostile
auditing* of the resulting suite, use the `roslyn-adversarial-prompts` roles (impl-vs-tests,
spec-vs-tests, test-suite mining, hidden-bug).

## Principles (the user's recurring rules)

- **"The compiler team rarely thinks something is too contrived to test for."** Don't self-censor.
- **Execute success cases.** For everything that should work, run the test and assert the real
  runtime output — not just "it compiled" or "no diagnostics."
- **Validate the failure cases too** — everything that should *not* work, with the exact diagnostic.
- **Confidence over exhaustiveness.** After mapping every case, *consolidate*: you don't need 40
  variants of "named parameters"; you need enough to be confident success/failure both work.
  > "we likely don't need exhaustiveness on, say, 'named parameters' ... but we need enough to feel
  > like the success/failure cases are working."
- **Separate test files** per area to avoid painful merge collisions: `<Feature>ParsingTests.cs`,
  `<Feature>BindingTests.cs`, `<Feature>EmitTests.cs`, `<Feature>SemanticModelTests.cs`, etc.
- **Smoke tests first, then deep areas.** Broad coverage without craziness, then drill per area.
- Expect a high **test:product code ratio** (often ~100:1 for cross-cutting features).

## Step 1 — mine the analogues

Before inventing tests, crawl the existing corpus for the nearest existing construct and ask
"do we have the `<feature>` analogue of each?" Launch test-suite-mining subagents (see
`roslyn-adversarial-prompts`). Adjacent corpora by feature shape:

| Your feature touches | Mine these existing tests |
|----------------------|---------------------------|
| `await` / async | `AwaitExpressionTests`, async iterator / EH / state-machine tests |
| null-propagation `?.` `??` `??=` | conditional-access, null-coalescing tests |
| object/collection initializers | `ObjectAndCollectionInitializerTests`, `RecordTests` |
| operators / comparisons | classical operator + comparison tests, constant-folding tests |
| extension members | classic extension-method + modern extension corpora |

## Step 2 — build the matrix

Enumerate the axes that matter for the feature, then cross them. Common axes:

- **Receiver / operand kind**: reference type, nullable value type, non-nullable value type,
  unconstrained generic, `dynamic`, pointer, ref-struct, tuple, null/default literal.
- **Result-type lifting**: `T` vs `T?` vs `Nullable<R>` vs `void`.
- **Operator / form**: each operator or syntactic form the feature allows.
- **Container / context**: statement, expression-bodied, field/property initializer, ctor,
  finalizer, lambda, local function, expression tree (usually rejected), query, attribute arg.
- **Execution model**: classic state machine vs runtime-async; sync vs async.
- **Control flow**: try/catch/finally, `using`, `lock`, `foreach`/`await foreach`, iterators.
- **Composition**: `??`, `??=`, switch-expression arms, interpolated strings, deep nesting,
  the feature combined with itself.

For value-type / generic axes, test **every type permutation** that changes behavior (e.g. a
widening chain `short < int < long` has 6 orderings worth pinning).

A feature-matrix adversarial subagent can generate and risk-rank the grid for you.

## Step 3 — pin the categories that always bite

- **Type inference & overload resolution** interacting with the feature's result type.
- **Nullable reference flow** (NRT): does the feature learn/forget null state correctly? `!`, `out var`.
- **Pattern matching / deconstruction** over feature results.
- **Definite assignment & reachability**.
- **Forbidden contexts**: default-param value, attribute arg, expression tree, const context.
- **Diagnostics**: exact code, message, and span on each malformed input (one targeted error, no cascade).
- **Emit/IL**: state-machine shape, spilling, unused-result handling, and an IL matrix across receiver/result.
- **Runtime behavior**: exception on the non-null branch, single-evaluation of shared sub-expressions,
  short-circuit (prove the skipped side is *not* evaluated, e.g. via a counter or throwing member).
- **LangVersion gating**: feature rejected pre-preview with `ERR_FeatureInPreview`.

## Step 4 — consolidate

Once the matrix + categories are mapped, cut to the minimal set that gives confidence. Drop tedious
near-duplicates; keep one test per distinct bug-generating corner. State explicitly what you dropped.

## Step 5 — organize & commit

- Tests live in their own files, per layer, language-version-scoped where relevant (e.g. a `CSharp15`
  test project). Comments in tests: no quoting compiler source, no line numbers, no narration.
- Tests are a **separate commit** from the code (see `roslyn-stacked-pr-feature`).
- When updating an existing test *area*, amend the existing commit and flow the change down the stack.

## Then audit it
Run impl-vs-tests, spec-vs-tests, test-suite mining, and hidden-bug auditors from
`roslyn-adversarial-prompts`, orchestrated per `roslyn-adversarial-orchestration`.
