---
name: roslyn-feature-planning-and-gating
description: >-
  Plan-and-gate workflow for Roslyn language features: research read-only first, capture the
  user's numbered decisions as a working contract, write a durable plan file under
  ~/.cursor/plans, and execute phase by phase (code -> review -> tests -> review) without
  auto-advancing. Use when starting a Roslyn feature, structuring a plan, or deciding whether
  to proceed to the next phase.
disable-model-invocation: true
---

# Roslyn Feature Planning & Gating

How this user wants language-feature work planned and paced. Pairs with
`roslyn-stacked-pr-feature` (branch/commit mechanics) and the impact-surface / adversarial skills.

## 1. Research before planning

- Switch to plan mode for cross-cutting features.
- Launch read-only `explore` subagents on the concrete code paths (parser, binder, bound nodes,
  lowering, semantic model) and require **concrete code citations** in their reports.
- Use `roslyn-feature-impact-surface` to map the blast radius.
- Do not touch code yet. "plan must be agreed before any code moves."

## 2. Capture the working contract

When the user dumps a design, restate it as a **numbered contract** and confirm before writing
anything: "Captured below as the working contract; I'll proceed to discovery on this basis and
come back before any spec text or code gets written." Include explicitly deferred items and any
open LDM questions. The contract becomes the acceptance checklist.

The user locks decisions tersely and numbered, honor them exactly:
> "1. message-id should come in phase2. 2. 'QuestionToken'. 3. yes, features/null-conditional-await. 4. yes. just parse normally."

## 3. Write the plan file

Location: `~/.cursor/plans/<feature>_<hash>.plan.md`, with `TodoWrite` todos tracking every PR.

Plan structure:
- **Status**: spec PR URL, branch, test project path.
- **Goal**: one paragraph.
- **Phase plan**: the stacked phases and PR ordering (see `roslyn-stacked-pr-feature`).
- **Directory layout**: file naming / ASCII tree.
- **Test philosophy**: "confidence over exhaustiveness."
- **Reference inventory**: categories mined from the existing test corpus.
- **Test-style conventions**: `Goo` not `Foo`, `VerifyDiagnostics`, no `#nullable disable`, etc.
- **Risks / unknowns**.
- **Process rules**, e.g.:
  - "Do NOT edit the plan file once attached to chat; reuse todos via `TodoWrite` with `merge: true`."
  - "Commit messages are plain imperatives; no AI tells, no em-dashes."
  - "Merge-forward only after a PR is open; force-push allowed only pre-PR."

Keep the plan body and the todos in sync, update both, not just one.

## 4. Execute with phase gates

- Order: **code → review → tests → review → next phase.** Never auto-advance.
- After each phase's code: stop, present the diff, ask for review. "Just make the code change and
  commit locally" until the user signs off.
- Before tests: "tell me what you're going to do."
- Run the adversarial suite before declaring a phase done.
- When asked what's next: **describe it, don't start it.** "what's the next phase? don't start on
  it, just tell me."

## 5. Verify subagent findings before acting

Explore/research subagents on large specs and codebases get section numbers and anchors wrong.
Re-read the primary source before baking a subagent's `file:line` or `§ref` into a plan or edit.
