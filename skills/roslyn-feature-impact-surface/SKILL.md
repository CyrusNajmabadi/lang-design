---
name: roslyn-feature-impact-surface
description: >-
  Sweep the Roslyn codebase to map everything a C# language feature touches before/while
  implementing it: bound-tree and syntax-node consumers that hard-cast or special-case
  adjacent constructs, compiler passes that must learn the new node (flow analysis,
  NullableWalker, RefSafety, lowering, IOperation factory, CFG), IDE/analyzer areas to fix
  up, intersecting language features, and PublicAPI/SyntaxKind switch obligations. Use when
  starting a Roslyn feature, scoping its blast radius, or hunting for places that need
  updating to handle a new or changed syntax/bound node.
disable-model-invocation: true
---

# Roslyn Feature Impact-Surface Discovery

Find *everything the feature intersects* so nothing silently breaks. Run this early (to scope
the work) and again after the node shape is final (to catch consumers). Fan these out as
read-only `explore` subagents in parallel; feed results to the interaction-audit and
IDE-coverage adversaries (`roslyn-adversarial-prompts`).

> Recurring instruction: "find everything that specializes on `<adjacent construct>` and make
> sure `<new marker>` gets the same treatment."

## What to sweep

### 1. Bound-tree / syntax-node consumers
Every pass that consumes the node you add or the bound node you reshape, especially ones that
**hard-cast** or **flatten left spines**:
- `LocalRewriter` (all partials), `ExpressionLambdaRewriter` (expression trees)
- `NullableWalker`, `DefiniteAssignment`, `AbstractFlowPass`, reachability
- `RefSafetyAnalysis` and any `BoundTreeWalker` that walks an operator spine
- `CSharpOperationFactory` / `IOperation`, `ControlFlowGraphBuilder` (CFG)
- Constant folding, `SymbolDisplay`, speculative-binding paths

For a node that **short-circuits** or carries a **new marker flag** (e.g. `IsChainedRelational`,
a new `QuestionToken`), confirm each walker either handles it or correctly stops — a flatten that
ignores short-circuiting is a classic bug.

### 2. Syntax/API obligations
- `SyntaxKind` additions → every `switch` over `SyntaxKind` (`SyntaxFacts.IsName`/`IsExpression`,
  `GetStandaloneNode`, visitor/rewriter overrides).
- `PublicAPI.Unshipped.txt` entries; experimental marking if applicable.
- Roundtrip / red-green tree integrity; `ToFullString()` fidelity.

### 3. Parser grammar positions
For an expression-shaped feature, enumerate **every place an expression can appear** by finding all
`ParseExpression`/`ParseExpressionCore`/`ParseSubExpression` call sites, then make sure the feature
parses (or is correctly rejected) in each: `foreach (var x in .Y)`, `if/while/do/switch/lock`,
`yield return`, `throw`, ctor `: base(...)`, LINQ clauses, interpolation holes, parenthesized, etc.

### 4. Adjacent / intersecting features
Diff behavior against the nearest existing features for parity: `await using` / `await foreach`,
`?.`/`??`/`??=`, collection expressions, classical operators, classic vs modern extensions. List
where the new feature must compose with each (and where it must be rejected).

### 5. IDE / analyzer surface
Everything that pattern-matches the changed syntax: refactorings, code fixes, completion, signature
help, quick info, classification/highlighting, find-refs, navigation, and analyzers. For each, decide:
needs handling, currently breaks, or safe no-op (justify). This is its own follow-up PR track.

## How to run it

- Use Grep/Glob to enumerate consumers and `switch`/cast sites; don't read the tree linearly.
- One subagent per area (bound-tree consumers, syntax/API, parser positions, IDE) in parallel.
- Each returns a **locator report**: `file:line`, the method, 5-10 lines of context, and a verdict —
  `Confirmed breaks` / `Fragile but currently-gated` / `Safe` — with a minimal repro for breaks.

## Output: an impact map

Consolidate into a checklist that feeds the implementation plan and the test matrix:

```
## Must-change (product code)
- file:method:line — why — phase

## Consumers to verify (may break)
- Confirmed breaks / Fragile-but-gated / Safe

## Parser grammar positions to cover
- <position> -> <ParseExpression* call site>

## Intersecting features
- <feature> — composes how / rejected where

## IDE / analyzer fixups (separate track)
- <feature area> — needs handling | breaks | safe no-op
```

Then drive: must-change items become implementation work; consumer breaks become bugs to fix in the
right phase; parser positions and intersections become test-matrix rows (`roslyn-feature-test-design`);
IDE fixups become their own PR track.
