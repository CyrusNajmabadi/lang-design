---
name: roslyn-test-conventions
description: >-
  Mechanics and hygiene for Roslyn compiler tests: preview-feature test layout under CSharp15
  (partial files per area, linked ParsingTests), validating parse trees with UsingTree, default
  CreateCompilation parse options (only pass parseOptions to pin an old langver), identifier and
  comment conventions (Goo not Foo, no brittle internals comments, preserve historical rationale).
  Use when writing, placing, or cleaning up Roslyn compiler tests. Pairs with roslyn-feature-test-design.
disable-model-invocation: true
---

# Roslyn Test Conventions

Mechanics and hygiene (distinct from `roslyn-feature-test-design`, which is about *what* to test).

## Layout for preview features

- Tests for a preview feature go in the language-version-scoped test project (e.g. `CSharp15/`).
- Use a `partial class <Feature>Tests` split into per-area files: `<Feature>Tests.Partial.cs`,
  `<Feature>Tests.Ref.cs`, etc. Keeps PRs in separate files to avoid merge collisions.
- When the parsing-test base classes aren't yet on `main`, **link** them into the csproj rather
  than moving them (lower blast radius):
  `<Compile Include="..\Syntax\Parsing\ParsingTests.cs" Link="Parsing\ParsingTests.cs" />`
  (and `SyntaxExtensions.cs`).

## Parse-tree validation

- **Every parsing test validates via `UsingTree`** (then `N(SyntaxKind.X)` walks), not
  `ParseSyntaxTree` + descendant queries.
  > "basically, all tests should be validating with the UsingTree."
- `UsingTree` is parser-only: do not assert binder diagnostics inside tree-shape assertions.
- Exhaust the internal parser switches the feature touches (e.g. every case in `IsAwaitExpression`
  for non-async contexts).

## CreateCompilation parse options

- `CreateCompilation` defaults to the current preview language version. **Do not pass
  `parseOptions: TestOptions.RegularPreview`**: it's the default.
  > "we only pass when we want a different past value that we're explicitly validating old lang
  > version behavior on."
- Pass `parseOptions: TestOptions.RegularNN` only to pin and validate older-langver behavior. For
  an ordering/availability violation, pin a `RegularNN` (error) case AND a `RegularPreview` (clean) case.

## Identifiers & comments

- Identifiers in test source: **`Goo`, never `Foo`**.
- Test comments describe **observable / spec behavior**, not compiler internals. No quoting compiler
  code, no line numbers, no phase labels, no private helper names ("walker", "ResultIsUsed", etc.).
  > "remove comments like this ... this is brittle and gets out of sync with the actual code."
- In **product** code, by contrast, **preserve existing rationale comments verbatim** when
  refactoring (especially LangVersion disambiguation notes for `record`/`union`/etc.). Do not
  replace design-history comments with summaries.

## Emit/runtime tests

- For async-related features, run scenarios in **both** state-machine and runtime-async modes, with
  both IL-shape and execution assertions (see `roslyn-feature-test-design`).
- Avoid brittle IL substring asserts that also match unrelated branches (e.g. a bare
  `Assert.Contains("brfalse")`).

## Build/run
See `roslyn-stacked-pr-feature` and the repo's `eng/build.sh` / `eng/common/dotnet.sh` for build and
test invocation; on macOS use the repo-pinned dotnet and expect `net472` runs to be CI-only.
