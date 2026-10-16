---
name: github
description: Standard workflow for GitHub repos (YOURUSERNAME/*): open an issue, implement on a fresh branch, run two to three independent reviews, submit the PR, and merge only on user instruction. Use for any GitHub issue/PR work outside GerritHub; Gerrit repos (quads) use the gerrithub_* skills instead.
---

# GitHub workflow (YOURUSERNAME repos)

Reusable procedure for feature/fix work on GitHub-hosted repos such as
`YOURUSERNAME/easyman` and `YOURUSERNAME/winslopper`. This is GitHub flow: `git push`
plus `gh pr create`, not Gerrit.

## Golden rules (never break)

- **Never use `redhat.com` hostnames** in any examples, docs, comments, commits, issues, or PRs, always use a neutral placeholder like `example.com`.
- **Never attribute the AI** (`assisted-by: claude`) in commits, issues, PRs, or comments.
- **Never run `git add -A`.** Stage explicit paths only. Untracked scratch files and review notes must never be staged.
- **Never commit or push unless the user asked for GitHub activity.** The user's request to "create a PR / submit a patch" authorizes add + commit + push + PR for that change; everything else (force-push history rewrites, merging, other changes) needs separate approval.
- **One issue, one branch, one commit.** Branch off `origin/main` fresh: `git checkout -b <name> origin/main`. Never stack two issues in one branch. Reviewer-fix rounds amend that commit (`git commit --amend`), then `git push --force-with-lease`.
- **Rebase merges only.** When the user offers a merge option, take rebase: `gh pr merge --repo <owner>/<repo> <N> --rebase`. Never squash and never create a merge commit, so the branch's own commits land unchanged and a one-commit branch stays one commit with its message intact. If the repo forbids rebase merges, say so and ask before using anything else. Rebase the branch onto the base and resolve conflicts first (`git fetch origin <base> && git rebase origin/<base>`, `git push --force-with-lease`), then re-check `gh pr checks` right before merging.
- **Close what you satisfy.** Once a merged change satisfies an issue, close that issue with a comment citing the merge and, when the repo publishes releases from merges, the release that shipped it. Never leave a satisfied issue open, and never close an issue the change does not actually satisfy.
- **Delete merged branches.** Remove the PR's head branch from the remote as part of merging: `gh api -X DELETE repos/<owner>/<repo>/git/refs/heads/<branch>` (works without a local clone; `git push origin --delete <branch>` from a clone is equivalent). Delete merged branches only; leave an unmerged branch someone else is working on alone.
- **Semantic commit message**, summary under 50 chars, body wrapped at 72, and a `fixes: <issue URL>` line (3-space indent) when an issue exists:
  ```
  feat: one-click self-extracting installer

  Body paragraph, wrapped at 72 chars.

     fixes: https://github.com/YOURUSERNAME/easyman/issues/3
  ```
- **Two to three independent reviews before submitting.** Always run a refute pass and a deep review, plus an independent adjudicator that decides the course of action is sane, simple, and issue-free (see Reviews). This is mandatory, not optional.

## Preflight

```bash
gh auth status                    # must show the repo owner (e.g. YOURUSERNAME)
git fetch origin
git log --oneline -3 origin/main  # know the base
git status --short                # clean tree or known state
```

Read the existing repo conventions first (README, workflows, commit
history, tests). Match them; a second convention beside the existing one
is prohibited.

## GitHub markdown alerts (docs, issues, PR bodies)

Use the native alert syntax for callouts, never plain `> NOTE:` text:

- `> [!NOTE]` - information users should take into account, even when skimming
- `> [!TIP]` - optional information that helps a user be more successful
- `> [!IMPORTANT]` - crucial information necessary for users to succeed
- `> [!WARNING]` - critical content needing immediate attention due to risk
- `> [!CAUTION]` - negative potential consequences of an action

Formatting rules (always):

- Every content line inside an alert starts with `> ` (including the
  `[!TAG]` header line).
- Any new sentence or paragraph that stays inside the SAME alert is
  preceded by a lone `>` line, e.g.:
  ```markdown
  > [!CAUTION]
  > First paragraph text.
  >
  > Second paragraph text, still in the same alert.
  ```
- No trailing `>` line after the last content line of an alert.
- Separate alerts and following prose with a blank line.
- No emdashes anywhere in docs.

## 1. Open the issue first

```bash
gh issue create --repo <owner>/<repo> --title "feat: <what>" \
  --body "$(cat <<'EOF'
## Request
...
## Proposed solution
...
## Acceptance criteria
- ...
EOF
)"
```

The issue body must state the problem, the proposed solution, and
observable acceptance criteria. Note the issue number to cite in the
commit.

## 2. Implement and verify locally

1. Branch: `git checkout -b <name> origin/main`.
2. Implement with the repo's patterns; keep the diff surgical.
3. Verify: run the repo's tests/checks that cover the change
   (e.g. `pytest --cov`, `black --check`, YAML parse for workflows,
   `dotnet build`). For non-testable changes, smoke test what you can and
   report what could not be run locally.

## 3. Reviews (mandatory 2-3 independent teams)

Spawn read-only reviewer subagents in one `task` batch, each with a facts
file (`local://<topic>-facts.md`) containing the verified facts, repo
context, and the mandate:

1. **Refute team** (`agent: "reviewer"`): attack the change and try to
   refute it. Every load-bearing assumption needs evidence (file:line,
   command output, primary source). Returns REFUTED / STANDS per attack.
2. **Deep review team** (`agent: "reviewer"`): line-by-line correctness,
   security, maintainability. Returns BLOCKING / FOLLOW-UP / NIT per
   finding with evidence.
3. **Independent adjudicator** (`agent: "reviewer"`): decides from first
   principles whether the course of action is sane, simple, and
   issue-free, plus a risk table. Must not see the other teams' output.

Reconcile yourself: drop refuted nits, keep standing findings, fix real
issues, then amend the single commit and re-push (only after re-running
the local checks).

## 4. Submit the PR

```bash
git add <explicit paths>          # never -A
git commit -m "<prefix>: <summary>" -m "<wrapped body + fixes: URL>"
git push -u origin <branch>
gh pr create --repo <owner>/<repo> --base main --head <branch> \
  --title "<same as commit summary>" --body "$(cat <<'EOF'
Implements https://github.com/<owner>/<repo>/issues/<N>.

What it does:
- ...
Notes:
- ...
EOF
)"
```

Update rounds: fix, re-run checks, `git add <paths>`, `git commit --amend`,
`git push --force-with-lease` (own PR branch only). The PR URL must stay
the same.

## 5. Merge

Never merge on the user's behalf unless told. When the user says to:
`gh pr merge --repo <owner>/<repo> <N> --rebase` (rebase is always the choice
when one is offered; never `--squash` and never a merge commit). A rebase merge
needs the branch current: `git fetch origin <base> && git rebase origin/<base>`,
`git push --force-with-lease`, wait for `gh pr checks` to pass again, then
merge. Retarget a stacked PR (`gh pr edit <N> --base <base>`) before merging it
once its own base has landed.

Finish the merge, always:

1. Delete the head branch:
   `gh api -X DELETE repos/<owner>/<repo>/git/refs/heads/<branch>`.
2. Close every issue the change satisfies, with a comment naming the merge
   commit and the release that shipped it.

## 6. Verification checklist before reporting

- [ ] Issue created with acceptance criteria (cited by commit)
- [ ] Branch fresh off `origin/main`, one commit, semantic message, no AI attribution
- [ ] Explicit paths staged only
- [ ] Two to three independent reviews run and reconciled
- [ ] PR open with `fixes:` link in commit and issue link in body
- [ ] Repo tests/checks pass; anything unverifiable locally reported
- [ ] Diff shown to the user; user handles any commits/merges not explicitly authorized

## Known failure modes

- `gh` rate limits -> wait and retry; never report a PR as created on a failed call.
- PR head branch deleted/force-pushed wrongly -> recreate branch from the
  commit, push, reopen PR against the issue.
- Reviewers claim findings without evidence -> verify the finding yourself
  (file:line, repro) before changing anything; the user's rule is to fix
  only confirmed findings.

## Staging and temp data

Never stage work under `/tmp`: it is a small RAM-backed tmpfs here (about 3 GB free) and heavy runs fail there. Clone, create venvs, download, and keep scratch payloads under `~/REVIEWS/<repo>-<topic>/`; run suites with `mkdir -p ~/REVIEWS/tmp && TMPDIR=~/REVIEWS/tmp COVERAGE_FILE=~/REVIEWS/tmp/.coverage`. Full details: `skill://reviews_staging`.
