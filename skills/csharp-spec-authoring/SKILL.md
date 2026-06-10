---
name: csharp-spec-authoring
description: >-
  Conventions for authoring and revising a C# language proposal (csharplang speclet) that
  defers to the existing ECMA/C# standard sections, drafts in reviewable staged chunks, and
  survives a hostile "standardese" audit. Use when writing or editing a csharplang proposal
  markdown, aligning a speclet with the C# standard, or doing a discovery pass on prior
  csharplang issues and LDM notes for a feature.
disable-model-invocation: true
---

# C# Spec / Proposal Authoring

How to write a csharplang proposal the way these features were specced. For the hostile
audit pass itself, use the `speclet / standardese auditor` role in `roslyn-adversarial-prompts`.

## Source-of-truth layout

- Proposals live in `csharplang/proposals/<feature>.md`.
- The normative standard lives in `csharpstandard/` (e.g. `standard/expressions.md`); section
  anchors like §12.8.8 (null-conditional member access), §12.8.10 (null-conditional invocation),
  §12.9.8 (await) are what you cite and defer to.

## Discovery first

Before drafting, run a discovery pass (a subagent is ideal) over `dotnet/csharplang`:
- Find prior issues/threads, championed proposals, and LDM notes for the feature.
- "Be exhaustive, I'd rather have 20 results most of which I dismiss than 5 cherry-picked ones.
  Include closed and rejected items if substantive."
- Capture the motivating scenarios and any prior LDM objections / opt-out questions.

## Spec sufficiency audit (before rewriting Detailed design)

When a proposal's "Detailed design" may be incomplete, audit it against the real standard sections
*before* rewriting: read the ECMA prose it diffs and produce a **numbered gap analysis** in rough
priority order (uniqueness rules, additional valid targets, cross-ref dispatch, statement-expression
context, accessor combinations, missing worked examples, etc.). Then surface the few points that
"need your call rather than falling out" as numbered decisions, and gate the rewrite on the answers.

## Grammar production impact survey

When introducing a new grammar production (e.g. `compound_assignment_operator`), grep the standard
for the sibling production's call sites and **document** which sections could now reference the new
one. The expectation is to survey and document candidates, not necessarily to refactor them all.

## Standard-version migration

To re-anchor a proposal from `standard-vN` to `vN+1`: renumber §refs (section numbers shift, e.g.
§11 → §12), fix anchors, and check whether edits can be **simplified** because the newer standard
already added machinery the proposal was hand-rolling. Produce an anchor-mapping table.

## GitHub diff rendering

These render badly in GitHub's proposal diffs; avoid them:
- No `>` blockquote indentation on normative prose ("it makes github very unhappy in the diff").
- Bold/strikethrough around commas breaks (`field**,** or property`). Prefer **appending** a clause
  (`**or event**`) over inline comma surgery, or accept the comma placement and move the `**` markers.

## Drafting principles

- **Defer, don't restate.** Maximize clarity and lower redundancy by deferring to existing standard
  sections rather than re-enumerating their rules. If `?.` already defines result-type lifting and
  null-tests, say the new feature inherits them from §X rather than re-listing them.
- **Operational equivalence.** Where possible, define the feature by a desugaring to existing
  constructs (e.g. `await? t` behaves as `((object)t == null) ? default(X) : await t`).
- **Normative carve-outs.** When the implementation is intentionally stricter than a literal reading
  (e.g. an error on a known non-nullable value-type operand), state that carve-out normatively in the
  applicability section so the spec and Roslyn agree.
- **Stage the writing.** Draft in large chunks and get feedback on each chunk before continuing.
  "Come back with a plan before writing any spec text."
- **Examples use `var` on the left.** Typed declarations hide the pain the feature addresses; `var`
  shows it. Place examples before Notes.
- **Open LDM questions** for genuinely undecided points (per-category opt-outs, etc.) rather than
  silently picking.

## House style

- Standardese, not vernacular: "is not performed because there is no `T`", not "vacuous"; "argument
  expression", not "argument"; "the type of which is `T`", not "whose type is a type".
- No em-dashes. No AI-isms. Present tense for normative text; avoid "today"/"currently" in normative sections.
- Linked §refs, not bare ones. Keep terminology consistent with the section you diff against.
- Markdown-diff conventions follow the surrounding csharplang proposals (bold/strikethrough as the
  repo already does it, don't invent new diff markup).

## Self-check before the hostile audit

Verify links resolve, every quoted standard passage is verbatim, the diff against the standard is
accurate, and the issue/slug format is right. Then launch the standardese auditor.

## Hostile standardese audit

Run the `speclet / standardese auditor` (see `roslyn-adversarial-prompts`): aggressive, anal,
flags every vernacular phrase / missing rigor / broken ref / internal inconsistency, with a verbatim
quote + concrete fix per finding, severity BLOCKER/MAJOR/MINOR, and *only* problems (no "this is fine").

## Triage the findings

Bucket into **fix now / dismiss (with rationale) / open LDM question**. Dismiss explicitly, e.g.
"our prior proposals are verbatim-in-content, not verbatim-in-formatting", never silently. Get
sign-off, then apply as one "Address audit findings" commit on the proposal's own branch.

## Optional late pass

A prose-simplification reviewer can find *remaining* low-hanging redundancy after several tightening
rounds, but constrain it: no structural reorg, no changing technical claims, no introducing em-dashes,
stylistic tightening only.
