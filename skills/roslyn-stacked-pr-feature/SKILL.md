---
name: roslyn-stacked-pr-feature
description: >-
  Workflow for implementing a C# language feature in Roslyn as a stack of phased PRs
  (syntax -> binding -> lowering/emit -> semantic-model/IOperation), with code and tests as
  separate commits, human review gates between phases, and a repeatable merge-main
  propagation procedure through the whole stack. Use when planning the branch/PR structure
  for a Roslyn feature, sequencing the work, or syncing feature branches with upstream main.
disable-model-invocation: true
---

# Roslyn Stacked-PR Feature Workflow

How to structure a language-feature implementation as a reviewable stack. For test content
use `roslyn-feature-test-design`; for validation use the adversarial skills.

## Phase stack

Implement in dependency order, one PR per phase, each branching off the previous:

1. **Syntax / parsing**: `Syntax.xml`, `LanguageParser`, parsing tests, PublicAPI, generated nodes.
2. **Binding**: binder + bound nodes, diagnostics, `ErrorCode`/`MessageID`/resources, binding tests.
3. **Lowering / emit**: `LocalRewriter`, codegen, emit/IL tests.
4. **Semantic model / IOperation / flow**: SemanticModel APIs, `IOperation`, CFG, nullable, those tests.

IDE work (refactorings, completion, analyzers) is usually a separate later track, not in the core stack.

Each phase branches off the prior: `feature-binding` off `feature-syntax`, etc. The feature
integration branch is `features/<feature>` upstream.

## Commit discipline

- **Code and tests are separate commits.** Write the code, get it right, then add tests as their own commit.
- One concern per commit; attribute adversarial-driven fixes in the message
  ("Address hostile-audit findings", "surfaced by the spec-vs-code auditor").
- When you change an existing test *area* mid-stack, **amend** the existing commit and **flow it
  down** the dependent branches rather than appending fixups.
- Plain commit messages: no AI-isms, no em-dashes. Lead with the why.

## Review gates

- Stop after each phase's draft for human review before starting the next.
- Run the adversarial suite (`roslyn-adversarial-orchestration`) **before** declaring a phase done.
- Approvals are terse ("ok", "continue", "push"); corrections quote the exact line.

## Error-code / MessageID hygiene

New `ErrorCode` and `MessageID` values collide constantly when multiple features merge. On any
collision, **take the next free number** rather than reshuffling. Update in lockstep:
`ErrorCode.cs`, `ErrorFacts.cs` (IsBuildOnlyDiagnostic), `MessageID.cs` (+ LanguageVersion case),
`CSharpResources.resx`, all `xlf/*.xlf`, and any test that references the old `CSNNNN` string.

New preview features go under `LanguageVersion.Preview`; pin old behavior with a specific
`LanguageVersion`/`TestOptions.RegularNN`.

A new `IDS_Feature...` string needs an entry in **all 13 `xlf/*.xlf`** files: insert after an
existing anchor (e.g. `IDS_FeatureUnions`) with `target state="new"`, and regenerate xlf via tooling,
not by hand. Missing xlf entries break localization CI.

## Area-complete PR cadence

For features split across many areas (receiver kinds, member kinds, operators), **finish one area's
full PR set before starting the next**, so the overall PR stack stays consistent. Tests for each area
go in **separate files** to avoid collisions. If a later test surfaces a missing edge case in earlier
product code, **amend the upstream PR and flow the change down** the stack via merge-forward (not
rebase + force-push) once any PR in the stack is open.

## Merge-main propagation (repeatable procedure)

When upstream `main` advances, propagate through the whole stack. Do it in the parent shell
(branch state must persist across steps), not in throwaway subshells.

```
Phase A, create the merge into the feature integration branch:
  git checkout -B merge-main-into-<feature> upstream/features/<feature>
  git merge upstream/main --no-edit -m "Merge `main` into `features/<feature>`"
  # resolve, build the compiler project, push, open PR against features/<feature>

Phase B, flow it down each stack branch in order:
  for each branch b in stack order (syntax -> binding -> emit -> semantic-model):
    git checkout -B b origin/b
    git merge <previous-branch-or-merge-branch> --no-edit -m "Merge ... into b"
    # resolve conflicts, BUILD, commit, push
```

### Conflict resolution playbook

- **PublicAPI.Unshipped.txt / resx / xlf**: almost always adjacent additions, keep both sides.
  After resolving xlf, validate XML well-formedness (a conflict can split a `<trans-unit>`).
- **ErrorCode.cs / MessageID.cs / ErrorFacts.cs**: renumber your feature's codes to the next free
  slot; reconcile the feature list. Then bump any `CSNNNN` references in your tests in lockstep.
- **Generator-owned files (e.g. `SourceWriter.cs`)**: when a general improvement landed on main,
  take main's version; don't keep the local copy.
- For parallel (non-stacked) PRs off the same feature branch, resolve the shared infra files once
  on the first branch, then `git checkout <that-branch> -- <file>` onto the rest.

### Per-step verification

- **Build the compiler project after every merge** (`Microsoft.CodeAnalysis.CSharp.csproj`); 0 errors
  before pushing. A build catches stale `CSNNNN` test references and conflict markers a grep can miss.
- Final sweep across all branches: confirm no committed conflict markers
  (`git grep -l "^<<<<<<< " origin/<branch>`).

## Cleanup

Once the merge-main PRs land, delete the `merge-main-into-*` branches (local + origin). The stack
branches then sit cleanly on top of the updated feature branch.
