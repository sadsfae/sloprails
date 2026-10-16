# Gerrithub Patchset Skill

Create, amend, and submit a Gerrit patchset for `quadsproject/quads` when the user
has explicitly authorized Gerrit activity for the session. Gerrit operations are
**write operations on shared review state**; only run them after the user says to
do so (e.g. "make a patchset", "push this", "address the comments", "git review").

## Golden rules (never break)

- **Never run `git push` on Gerrit.** The only submit path is `git review development`.
- **`git review development` always runs over SSH** (remote `gerrit` =
  `ssh://YOURUSERNAME@review.gerrithub.io:29418`, configured in `.gitreview`). Never
  switch that remote to HTTPS or feed it credentials.
- **HTTP credentials are for Gerrit REST only.** The Gerrit HTTP password lives in
  `~/git_modules/quads/.gerrit_credentials` (gitignored as of change 1248936).
  It is used solely for inline-comment/vote POSTs via
  `https://review.gerrithub.io/a/...`; never pass it to git transports, and
  never print it to output.
- **Never run `git add -A`.** Stage explicit paths only. Untracked side files
  (`.md` review notes, `.gerrit_credentials`, scratch artifacts) must never be staged.
- **Never commit files on the user's behalf unless authorized.** The user's
  explicit request to "make/push a patchset" authorizes add + commit + review for
  that change; everything else (force push history rewrites, merging other
  changes) needs separate approval.
- **Never hand-manage the Change-Id.** The gerrit commit-msg hook owns it.
  - To create: commit a message with no `Change-Id` line; the hook adds one.
  - To update a message: `git commit --amend` (editor) **keeping the existing
    `Change-Id` line**. Do not rebuild the message with `-m` alone, that drops
    the line and a re-push then opens a *new* change instead of a new patchset.
  - Recovery if dropped: `git commit --amend -m ... -m "... Change-Id:
    <original>"` restoring the original ID.
- **Never attribute the AI** (`assisted-by: claude`) or add accreditation to
  commit messages or Gerrit comments.
- **One branch per patchset**, fresh off `remotes/gerrit/development` (or the
  current Gerrit tip): `git checkout -b <name> remotes/gerrit/development`.
  Never stack two issues in one branch.
- **One change, one commit.** Reviewer-fix rounds are amends of that commit, not
  new commits.

## Preflight

```bash
git fetch gerrit
git log --oneline -3 remotes/gerrit/development    # know the base
git status --short                                 # clean tree or known state
```

## Commit message format (per repo convention)

```
<prefix>: <short summary>

<body paragraph, wrapped at 72 chars>.

   fixes: https://github.com/quadsproject/quads/issues/<N>

```
(If there is no issue, omit the `fixes:` line.)

- Prefix: `fix:`, `feat:`, `chore:`, etc.
- Summary line short (under 50 chars preferred; Gerrit warns beyond that).
- Body lines wrapped <= 72 chars (Gerrit warns otherwise).
- `fixes:` line: three spaces indent, blank line after, trailing blank lines.

## Workflow to create a patchset

1. Fetch + confirm base: `git fetch gerrit`.
2. Create the branch, implement, run tests (repo convention):
   ```bash
   TZ=UTC SQLALCHEMY_DATABASE_URI=postgresql://postgres:postgres@localhost:55432/quads \
     PYTHONPATH=src venv/bin/python -m pytest tests/api/ -q
   venv/bin/black --check <changed files>
   ```
   (If no local Postgres: run `postgres:16-alpine` in podman on port 55432,
   `POSTGRES_PASSWORD=postgres POSTGRES_DB=quads`. Never use the production/dev
   server database.)
3. Stage **explicit paths only**, then commit:
   ```bash
   git add src/... tests/...           # never -A
   git commit -m "<prefix>: <summary>" -m "<wrapped body>"
   ```
   The hook adds the `Change-Id`; verify: `git log -1 --format=%B | tail -3`
   shows a real hex `Change-Id: I<40 hex>` (not placeholders).
4. Submit:
   ```bash
   git review development
   ```
5. Verify the upload:
   - Output shows `SUCCESS` and `https://review.gerrithub.io/c/quadsproject/quads/+/<N>`.
   - Confirm it updated the *same* change (same Change-Id) when amending;
     a message-only amend shows `no files changed, message updated`.
   - CI runs after upload; check `gerrit query --comments change:<N>` for the
     `quads-ci` build result. No further action needed until CI/review.

## Workflow to address reviewer comments

1. Fetch the comments:
   - Change-level messages: `ssh -p 29418 YOURUSERNAME@review.gerrithub.io
     "gerrit query --format JSON --comments change:<N>"`.
   - Inline comments: `curl https://review.gerrithub.io/changes/<N>/comments`
     (`)]}'` prefix must be stripped; each entry has `patch_set`, `line`,
     `author`, `message`).
   - If REST comments show only old patchsets, fetch refs:
     `git fetch gerrit refs/changes/<XX>/<N>/<M>:refs/review/<N>/<M>` and diff.
2. Verify each comment is valid against the current code before changing
   anything (the user's rule: address only confirmed findings).
3. Make the changes, re-run tests, then:
   ```bash
   git add <explicit paths>
   git commit --amend          # editor: keep the Change-Id line; wrap <= 72
   git review development      # next patchset on the same change
   ```

## Rebase / stacked changes

Changes touching the same files must land in a deterministic order (smallest /
most independent first). When a change must be rebased onto newly merged work:

```bash
git rebase remotes/gerrit/development     # or after a later change merges
# resolve conflicts manually, keep BOTH sides' features when the files overlap
git add <resolved files>
git rebase --continue
# re-run tests, then amend? no: single commit preserved by rebase
git review development
```

Merge-conflict resolution convention in this repo: when two of our patches touch
the same function/file (e.g. `schedules.py`), the rebased result must keep both
features (imports from both, both handlers' behavior) and the test files must
keep both changes' test classes (append, never drop).

## Recovery from `git add -A` / junk-in-commit

If untracked side files got staged or committed:

```bash
git reset --soft <last-clean-commit-sha>   # branch back; index/worktree kept
git restore --staged .                     # unstage everything
git add <explicit paths only>
git commit --amend                         # keep the existing Change-Id
```

## Post-submit note

- Only the submitter/maintainer merges (`Submit`). Never merge on Gerrit's behalf
  unless explicitly told.
- Draft Gerrit reply messages for the user; do not post review votes/messages
  unless asked. If asked to post: change messages via
  `ssh ... gerrit review <N>,<patchset> --message '...'`, votes via
  `--code-review +2`; inline comments use the HTTP password from
  `~/git_modules/quads/.gerrit_credentials` (never via SSH; SSH has no inline
  comment support) and go through
  `POST https://review.gerrithub.io/a/changes/<N>/revisions/current/review`
  with a `comments` map (this Gerrit does not implement the modern
  `/comments` POST endpoint).
