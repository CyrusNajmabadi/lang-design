---
name: roslyn-pr-conventions
description: >-
  Conventions for creating and managing dotnet/roslyn (and dotnet/csharplang) pull requests:
  fork/upstream remote setup, targeting the upstream features/<feature> branch, draft PRs via
  gh, PR bodies that link the championed issue + speclet + test plan and chain prev/next in the
  stack, replying to review threads with only the commit SHA, and the gh auth / SSH-remote
  gotchas. Use when opening, describing, retargeting, or responding to PRs on Roslyn or csharplang.
disable-model-invocation: true
---

# Roslyn / csharplang PR Conventions

How PRs are created and managed for this user's compiler work.

## Remotes & auth

- **SSH remotes, not HTTPS.** Clone/set `origin` to `git@github.com:CyrusNajmabadi/<repo>.git`.
  HTTPS pushes can pick up the wrong (EMU `*_data`) keychain account and 403.
  Fix an HTTPS clone with: `git remote set-url origin git@github.com:CyrusNajmabadi/<repo>.git`.
- `origin` = the user's fork (`CyrusNajmabadi/<repo>`); `upstream` = `dotnet/<repo>`.
- If `gh` operations hit a permission error, the wrong account is active:
  `gh auth switch --user CyrusNajmabadi`.

## Branch targeting

- The user creates the upstream `features/<feature>` integration branch (and its seed PR) themselves.
- All your feature PRs target **`features/<feature>`**, never `main`.
- Local branches live on the fork with short descriptive names (e.g. `relax-ref-ordering`,
  `crc/binding`, `null-conditional-await-binding`); push to `origin`, open PR against the upstream
  feature branch. When `features/<feature>` currently equals `main`, retargeting is metadata-only
  (no rebase).

## Creating PRs

- Open as **draft** while iterating: `gh pr create --draft --repo dotnet/roslyn --base features/<feature> --head CyrusNajmabadi:<branch> --title "<title>" --body "$(cat <<'EOF' ... EOF)"`.
- Run the PR-author adversary (see `roslyn-adversarial-prompts`) before taking a PR out of draft.

## PR body format

Lead with a standard link block, then a short human-voice summary, then the stack chain:

```
- Championed issue: dotnet/csharplang#NNNN
- Speclet: <feature>.md
- Test plan: #NNNNN

<one or two plain sentences on what this PR does>

First in a series. Next up is #NNNNN.    (or: Follows #NNNNN. Next: #NNNNN.)
```

- Each PR links the **previous and next** PR in the stack.
- Titles and bodies are plain and factual. No AI tone, no em-dashes (see the human-voice rule).

## Responding to review feedback

- Address **one scoped topic at a time**. Make a single commit, push, then reply to each affected
  thread with **only the commit SHA** and resolve it. Nothing else in the reply.
  > "All the response should be is the git sha of the single commit you just pushed ... Do not
  > respond with anything but the sha."
- Do not address unrelated feedback in the same pass.

## Git hygiene

- Avoid `--amend` / force-push once a PR is open; use **merge-forward** to flow fixes down a stack
  (see `roslyn-stacked-pr-feature`). Force-push is acceptable only pre-PR.
- Commit messages: plain imperatives, detailed HEREDOC bodies for non-trivial changes, no AI tells.
