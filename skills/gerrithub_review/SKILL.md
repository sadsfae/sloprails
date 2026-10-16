---
name: gerrithub_review
description: Deep-review a GerritHub patchset (with a contra-review pass), then post review comments as the current user via the GerritHub REST API. Use for Gerrit reviews in the quads repo.
---

# GerritHub review workflow (quads)

Reusable procedure for reviewing a Gerrit change on review.gerrithub.io and posting comments as the local user (YOURUSERNAME / Will Foster), derived from the 1248884 review (2026-09-08).

## 1. Setup

- Repo remote `gerrit` = `ssh://YOURUSERNAME@review.gerrithub.io:29418/quadsproject/quads`; `.gitreview` host is `review.gerrithub.io`; gitreview username `YOURUSERNAME`.
- The patchset is checked out on `review/<change-id>`; the change is one commit ahead of `gerrit/development`. Read it with `git show <sha>` (or `git diff gerrit/development HEAD`).
- Fetch change metadata anonymously (responses start with `)]}'`; strip that prefix before parsing):
  `curl -s 'https://review.gerrithub.io/changes/quadsproject%2Fquads~<N>/detail?o=ALL_REVISIONS&o=MESSAGES'`
- List existing comments (anonymous is fine): `.../changes/quadsproject%2Fquads~<N>/comments`.
- Confirm the local HEAD matches the Gerrit current revision before commenting on line numbers.

## 2. Deep review

- Read the whole diff, then the surrounding context: imports, the endpoint, every callee the change touches, and every caller of changed symbols.
- Trace return types and failure modes of called functions (e.g. `User.decode_auth_token` returns strings on failure; both success and failure are strings, so `isinstance(x, str)` cannot distinguish them).
- Check every user-creation/validation path that could conflict with new validation introduced by the change (CLI, API, config defaults, OAuth).
- Check dependency ranges: `requirements.txt` pins are often lower-bounded only; behavior can differ across allowed versions (verified example: `validators.email("user@localhost")` is True on 0.20.0, False on 0.35.0; PyJWT only validates string `sub` from 2.10.0 while `PyJWT>=2.6.0` allows 2.6-2.9).
- Run the affected tests locally (no local Postgres; the SQLite override works):
  `SQLALCHEMY_DATABASE_URI=sqlite:////tmp/quads_test.db ./venv/bin/python -m pytest tests/api/test_auth.py -q`
  Run the full module: some tests rely on a module-global token set by an earlier test.
- Prove new tests are genuine: `git worktree add /tmp/qq_nofix HEAD`, revert only the guard, run the new test (must fail), then `git worktree remove --force /tmp/qq_nofix`. Never run git add/commit in the user's repo.
- Run black/flake8 on changed files (project style: black; .flake8 config).

## 3. Contra-review

- Spawn 3 reviewer subagents (task call, `agent: "reviewer"`, read-only): one per finding cluster, plus one novel-defect hunter.
- Give each a findings file (write to `local://` first), the verified facts (commands and probe results), and an explicit mandate to REFUTE with evidence (file:line, command output) and to return REFUTED/STANDS plus severity.
- Reconcile: drop refuted nits, keep standing findings, correct any claim that a contra agent disproved with a reproducible check.

## 4. Post comments (as the user; NEVER git push to GerritHub)

- Credentials: repo root `.gerrit_credentials` (or env `GERRIT_HTTP_PASSWORD`); username `YOURUSERNAME`. Never echo the password.
- Auth is HTTP **Basic**, not digest (GerritHub returns 401 for digest). Verify first:
  `curl -s -u "YOURUSERNAME:$PASS" https://review.gerrithub.io/a/accounts/self` (expect Will Foster, account id 1006732).
- Endpoint: `POST https://review.gerrithub.io/a/changes/quadsproject%2Fquads~<N>/revisions/current/review`
  Body: `{"message": "<cover summary>", "comments": {"<path>": [{"line": <new-file line>, "message": "...", "unresolved": true|false}]}}`
  - `comments` is a MAP of path to array on this GerritHub version. The dedicated `/changes/.../revisions/.../comments` endpoint returns 405 (not implemented) here.
  - `line` is 1-indexed in the NEW file. `unresolved: true` for issues that need attention; nits get `unresolved: false`.
  - Do not include `labels` unless the user explicitly asks for a vote.
  - `message` is optional; keep it short.
- Verify after posting:
  `GET /a/changes/.../comments` (check author, line, unresolved flags) and `/detail?o=MESSAGES` (check `unresolved_comment_count`).
- Deleting a change message requires admin (`403 administrate server not permitted`). NEVER post probe/test messages on a real change; if one slips through, tell the user to remove it.
