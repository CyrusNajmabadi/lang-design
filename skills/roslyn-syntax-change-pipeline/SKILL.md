---
name: roslyn-syntax-change-pipeline
description: >-
  The exact pipeline for changing C# syntax in Roslyn: edit Syntax.xml, run the compiler-code
  generator, hand-author backward-compatible partial forwarders, update PublicAPI.Unshipped.txt,
  and build with analyzers. Covers experimental API marking and the generated-file and chmod
  gotchas. Use when adding or modifying a syntax node, token, or SyntaxFactory overload in Roslyn.
disable-model-invocation: true
---

# Roslyn Syntax-Change Pipeline

Dense, error-prone mechanics for changing the C# syntax model. Follow in order.

## 1. Edit `Syntax.xml`

- Add/modify the node, field, or token in `src/Compilers/CSharp/Portable/Syntax/Syntax.xml`.
- Follow an existing precedent node with the same shape (e.g. an optional token field like
  `ClassOrStructConstraintSyntax`).
- For an experimental field/node, set `ExperimentalUrl` pointing to the tracking issue; the
  generator propagates `[Experimental]` to the affected `Update`/`SyntaxFactory` overloads.

## 2. Regenerate

- Run the generator: `eng/common/dotnet.sh run --file eng/generate-compiler-code.cs`
  (or build, the `*.Syntax.Generated.cs` come from the source generator at build time).
- Do **not** hand-edit generated files.
- If you `chmod +x eng/common/dotnet.sh` to run it, **revert the chmod**: it is not ours to commit.

## 3. Hand-author backward-compatible forwarders

When a new optional field changes a node's `Update`/factory signature, add partial-class
forwarders so existing call sites keep compiling, e.g. a 2-arg `AwaitExpressionSyntax.Update`
and the matching `SyntaxFactory.AwaitExpression` overload that forward to the new signature.
Mirror the precedent node's forwarder pattern.

## 4. Public API

- Add every new public member to `src/Compilers/CSharp/Portable/PublicAPI.Unshipped.txt`.
- Prefix experimental entries with the analyzer code (e.g. `[RSEXPERIMENTAL00N]`).
- A new shorthand factory whose signature did **not** change must NOT be marked experimental.

## 5. Wire the SyntaxKind into every switch

A new `SyntaxKind` must be handled in the switches that classify it: `SyntaxFacts.IsName`/
`IsExpression`, `GetStandaloneNode`/`GetStandaloneType`, visitor/rewriter overrides, and any
`SyntaxKind` switch in the IDE/analyzer layers (see `roslyn-feature-impact-surface`).

## 6. Build with analyzers

```
./eng/build.sh --solution Compilers.slnx --restore --build --configuration Debug \
  --runanalyzers --warnaserror /p:RoslynEnforceCodeStyle=true
```

Expect a SemanticSearch reference-assembly baseline regeneration on public syntax changes; commit
the regenerated baseline.

## 7. Resource/diagnostic plumbing (if the change adds diagnostics)

New `MessageID`/`ErrorCode` need entries in `MessageID.cs` (+ LanguageVersion case), `ErrorCode.cs`,
`ErrorFacts.cs`, `CSharpResources.resx`, and **all 13 `xlf/*.xlf`** files (insert after an existing
anchor like `IDS_FeatureUnions`, `target state="new"`); regenerate xlf via tooling, not by hand.
On numbering collisions take the next free number (see `roslyn-stacked-pr-feature`).
