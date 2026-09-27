---
name: gardener
description: Triage a project's open issues by trying to reproduce each one on the default branch, and close only those that are verifiably no longer reproducible. Use when asked to garden, prune, or clean up stale issues.
---

# gardener

Walk the open issues, try to reproduce each one on current code, and close the ones that are provably gone. Everything else is left untouched: no comments, labels, or edits on issues that stay open.

Closing a real bug is far worse than leaving a fixed one open. When in doubt, leave it.

## Arguments

All optional:
- a ref to test against (default: the remote's default branch)
- issue numbers to check (default: every open issue)
- a cap on issues per run (default: 10)

## Setup

Detect the forge from `git remote get-url origin`: GitHub uses `gh`, GitLab uses `glab`. Stop if neither is available and authenticated.

```bash
git fetch origin
REF=<ref or origin's default branch>
WT=$(mktemp -d)/garden; git worktree add --detach "$WT" "$REF"
```

Do all testing in `$WT`. Never touch the user's working tree.

## Pick issues

List open issues, oldest first. Skip, without comment:
- feature requests, questions, discussions, tracking issues, anything that isn't a defect report
- issues with activity in the last 14 days
- issues with an open PR/MR linked to them
- issues labelled or described as won't-fix, blocked, upstream, or keep-open

## Check each issue

1. **Read everything:** body, comments, linked issues, attached logs. Pin down the repro steps, environment, and expected vs actual behaviour. If there's no concrete repro you can run, leave it open.
2. **Reproduce on `$REF`:** follow the steps literally in `$WT`, with the same config, flags, and inputs. Actually run it; reading code is not reproducing. Try the reporter's variants and anything commenters added.
3. **Prove the repro is faithful:** check out the last commit before the issue was filed (`git rev-list -1 --before=<created_at> $REF`) and run the same steps. If the bug doesn't show there either, your repro is wrong, so leave the issue open.
4. **Find the fix:** search merged PRs/MRs and commits that reference the issue (`#N`, the title, key error text), plus the issue's timeline cross-references. Bisect between the pre-issue commit and `$REF` if nothing turns up. Read any candidate fix and confirm it addresses this issue's cause, not something adjacent.
5. **Check for partial fixes:** make sure every symptom and variant in the thread is gone, not just the headline one.

## Close only when all of these hold

- the bug reproduced at the pre-issue commit (or, if that commit can't be built, a merged change clearly fixes this exact cause)
- it does not reproduce on `$REF`, including every variant from the thread
- you ran the steps and saw the result yourself

Then close it with one comment:

```
Gardening. Closing as this issue appears no longer reproducible.

- Reproduced at <old sha>: <one line result>
- Not reproducible at <ref sha>: <one line result>
- Fixed by <PR/MR or commit link>, if found
```

GitHub: `gh issue close <n> --reason completed --comment "<comment>"`. GitLab: `glab issue note <n> -m "<comment>"`, then `glab issue close <n>`.

## Never

- close an issue you couldn't run the repro for
- comment on, label, assign, or edit issues you leave open
- change code, push, or open PRs
- reopen, lock, or delete anything

## Finish

Remove the worktree: `git worktree remove --force "$WT"`.

Report a table: issue, closed or left open, one-line reason.
