---
name: issue-to-pr
description: Take a GitHub task from observation to a linked pull request. Find or file the issue, create or reuse the issue branch, implement in atomic commits, verify end-to-end, push, and open a PR that closes it. Use when asked to fix, implement, or work on a change that should land as an issue, branch, commits, and a pull request, or when resuming one that already has any of them. Quick local edits and experiments stay local.
---

# issue-to-pr

Carry one task the whole way: issue → branch → atomic commits → end-to-end verification → pushed branch → PR that closes the issue.

Each phase begins by **detecting** its artifact. Re-running is safe: an existing issue, branch, or PR is reused and the phase moves on. Run Phase 0, then start the chain at Phase 1; each detect step skips completed work. Track `DEFAULT`, `ISSUE`, and `BRANCH` in your working state and substitute their literal values into commands — shell variables do not persist across separate tool calls.

Requires an authenticated `gh`. If `gh auth status` fails, say so and stop — a half-run leaves orphaned branches.

## Phase 0 — Ground the task

```sh
gh auth status
gh repo view --json defaultBranchRef -q .defaultBranchRef.name
git fetch origin
git status --short
```

Save the default branch name as `DEFAULT`.

Write down one outcome sentence and the acceptance criteria. For a bug, capture the exact reproduction command and its current failure; for a feature, the observable behavior that proves it works. This list becomes the issue checklist and the Phase 4 evidence checklist.

If the request cannot produce an issue title, ask one focused question and wait. When resuming, read the existing branch and PR before asking anything.

## Phase 1 — Issue

Detect from both ends: a previous run may have filed the issue, or opened a PR whose body links it.

```sh
gh issue list --state all --search "<distinctive words> in:title,body" --json number,title,state,url --limit 20
gh pr list --state open --search "<distinctive words> in:title,body" --json number,title,url,headRefName,body --limit 20
```

Resuming from a branch? Search its name and commit subjects.

- Exactly one open match → set `ISSUE=<number>`, then read it.
- Several plausible matches → list titles and ask which; file nothing.
- The only match is closed → stop and ask: reopen it (`gh issue reopen <ISSUE>`), or file a new issue that references the old.
- No match → file it:

```sh
gh issue create --title "<area>: <outcome>" --body-file - <<'EOF'
## Context
<what was observed and where>

## Expected
<observable behavior>

## Acceptance
- [ ] <criterion>
EOF
```

Add labels only when `gh label list` offers fitting ones. Keep the reproduction command in the body for bugs.

Done when `gh issue view <ISSUE> --json state` is `OPEN` and the body carries the acceptance checklist.

## Phase 2 — Branch

```sh
git branch --list "<ISSUE>-*"
git branch --remotes --list "origin/<ISSUE>-*"
git ls-remote --heads origin "refs/heads/<ISSUE>-*"
```

One candidate → strip any leading `origin/` prefix to get `<branch>`, then switch and fast-forward:

```sh
git switch "<branch>" || git switch --track "origin/<branch>"
git pull --ff-only
gh pr list --head "<branch>" --state all --json number,state,url
```

- Open PR on it → this is a resume; keep going on the branch.
- Merged PR → the task likely shipped; confirm with the user before redoing it.

Several candidates → ask which one. None → create:

```sh
git status --porcelain
git switch -c "<ISSUE>-<slug>" "origin/<DEFAULT>"
```

Clean tree → branch from fresh `origin/<DEFAULT>`. Changes already belong to this task → `git switch -c "<ISSUE>-<slug>"` from current HEAD and carry them into Phase 3. Anything else uncommitted → stop and ask the user to commit or stash first.

Done when `git branch --show-current` is `<ISSUE>-<slug>`, and a new branch satisfies `git merge-base --is-ancestor "origin/<DEFAULT>" HEAD`. Leave reused branches un-rebased unless the user asks.

## Phase 3 — Implement in atomic commits

Match the repo's habits before writing:

```sh
git log --oneline -10
```

For a bug, run the reproduction first and keep the failing output for Phase 4.

Stage whole files, then commit:

```sh
git add <path> [<path> ...]
git commit -m "<subject in repo style>" -m "<why this change is needed>"
```

One logical change per commit, reviewable alone. When one file mixes concerns, stage a trimmed patch instead of reaching for `git add -p`, which stalls without a terminal:

```sh
git diff <path> > /tmp/issue-to-pr.patch
# delete unrelated hunks from the patch
git apply --cached /tmp/issue-to-pr.patch
```

Stage explicit paths; `git add -A` sweeps unrelated work into the commit. Keep secrets, `.env` files, and build output out. If a hook rewrites files, re-stage and commit again instead of reaching for `--no-verify`.

Done when every acceptance criterion has an implementation and `git log "origin/<DEFAULT>..HEAD" --oneline` reads as a coherent series.

## Phase 4 — Verify end-to-end

Review the diff, then verify in three layers:

```sh
git diff "origin/<DEFAULT>...HEAD"
```

1. **Reproduction** — rerun the exact command from Phase 3; it now passes.
2. **CI mirror** — read `package.json` scripts, `Makefile`, and `.github/workflows/*`, then run the same lint, typecheck, test, and build commands with the repo's package manager.
3. **Real path** — drive the changed entry point: the CLI invocation, HTTP request, or page flow from the issue, plus one edge case the change could plausibly break.

Record each result as criterion, command, observed output. It goes into the PR body as written evidence.

Failures: fix, commit, rerun. Collapse fixups before the first push against the merge-base, so a reused branch is not rebased onto a newer default:

```sh
git commit --fixup=<sha>
GIT_SEQUENCE_EDITOR=: git rebase --autosquash "$(git merge-base "origin/<DEFAULT>" HEAD)"
```

After the branch is pushed, add plain commits so review history stays stable.

If a check needs a service that is not running, ask the user to start it and wait. If a check is impossible, mark it unverified in the PR body — unverified work is never reported as verified.

Done when every acceptance criterion has a command and an observed result (or an explicit unverified reason), the project's checks pass, and the diff contains nothing beyond the task.

## Phase 5 — Push

```sh
git push -u origin "<BRANCH>"
git rev-parse HEAD "origin/<BRANCH>"
```

A rejected push — protected branch, non-fast-forward — is reported and resolved with the user, not forced. `--force-with-lease` only on your own branch before review, never once others may have pulled.

Done when both hashes match and `git status -sb` shows no ahead/behind.

## Phase 6 — PR

```sh
gh pr list --head "<BRANCH>" --state all --json number,state,url,body
```

- Open PR → reuse it. Push any new commits, then `gh pr edit` the body only if it is stale or missing the issue link.
- Closed or merged PR → stop and ask; the work may already be delivered.
- None → create. Add `--draft` when any acceptance criterion is unverified:

```sh
gh pr create --base "<DEFAULT>" --head "<BRANCH>" --title "<outcome>" --body-file - <<'EOF'
## Summary
<what changed and why>

## Changes
- <commit subject>

## Verification
- [x] <criterion> — `<command>` → <observed result>
- [ ] <criterion> — unverified (<reason>)

Closes #<ISSUE>
EOF
```

The `Closes` keyword is the link; a duplicate comment on the issue adds nothing.

Done when `gh pr view "<BRANCH>" --json number,url,state` is `OPEN` and the body contains `Closes #<ISSUE>`.

Report back: issue URL, branch, number of commits, one-line verification summary, PR URL.
