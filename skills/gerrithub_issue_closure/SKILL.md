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
github_review/SKILL.md0000644000175000017500000001466015252270216015076 0ustar  wfosterwfoster---
name: github_review
description: Deep, full-coverage GitHub PR review focused on DRY, prior-art awareness (what already exists in the base commit), and fit with recently-merged features and the project's architecture, then post the review as the local authenticated user via the `gh` API. Use when a GitHub PR needs an in-depth review for any repository and any contributor.
---

# GitHub PR review workflow

Reusable procedure for a deep, exhaustive GitHub pull-request review and for posting review comments as the local authenticated user. Use it for any GitHub PR, in any repo, regardless of author skill level. The review is written for a normal contributor: findings are actionable, cite exact code, and name the concrete fix.

## 1. Scope & lens

- **Focus (upweight):** DRY against the repo's *recently merged* features; whether new code builds on the existing flow/spirit/architecture instead of forking it; prior-art awareness, places where the submitter re-implements or is unaware of functionality that already exists at the base commit (cite BOTH `new file:line` and `base file:line`); consistency with existing UI/API conventions; doc-vs-code contradictions; security-relevant flows.
- **Downweight:** style nits, academic/defensive posture, hypothetical hardening, perfect-case exhaustiveness.
- Every finding: `file:line` what's wrong, why it matters, concrete fix, severity (major/minor/nit). End each team report with: fit verdict (1-2 sentences), doc contradictions, top 3 issues. Silence is not a finding.

## 2. Setup

- Clone the repo (`git clone`), fetch the PR head: `git fetch origin pull/<N>/head:pr<N>`. Base = `git merge-base pr<N> origin/<base-branch>`; prefer `development` as the base if it exists (`git ls-remote --heads origin development`), otherwise use the repo's default branch, check it with `gh repo view <owner>/<repo> --json defaultBranchRef` (do not assume `main`). Record `headRefOid` via `gh pr view <N> --repo <owner>/<repo> --json headRefOid`.
- Read the PR description, the full diff, and the structural map (changed files ranked by churn) *before* delegating. Re-read the PR head frequently: authors push while you review, always `git fetch` and re-verify anchors against the latest head.
- Write a shared review brief to a local file (repo path, base commit, feature summary, recently-merged context, architecture description, mandate + report format). Every team reads it first.

## 3. Team decomposition

No fixed team count, break out until coverage is complete. Member count scales with PR size; a small PR needs one pass, a large one needs several. Each team is a read-only `reviewer` subagent; give it a domain, the exact file list, and 4-6 directed questions. Do NOT pass a `tools` array (session tools are available by default). Use `task` with all teams in one batch; run the independent passes while teams work.

Typical split (adjust per PR):
1. **Auth/session/identity** - auth router, session helpers, role/permission checks, middleware, main wiring.
2. **Admin roles + admin surfaces** - role grant/revoke, admin pages, templates/JS for them, sign-in behavior.
3. **Core domain / primary feature path** - the models, services, or endpoints the PR actually adds or changes.
4. **Platform/API service layer** - API routes, serialization, validation, error handling, client-facing contracts.
5. **Frontend + docs** - templates, JS, CSS, config, README/UPGRADING cross-check of every documented route/behavior.
6. **Tests/verification honesty** - do new tests cover the contracts they claim; are any asserting internals or source text; missing tests at security-relevant contracts; coverage floor risk.
7. **Deployment/config/integration** - service units, env vars, proxy, middleware scope on streaming, doc requirements.
8. **Prior-art awareness** - compare every new element against the base commit; list "build on this instead" opportunities with base citations.

## 4. Independent owner pass (contra/verify)

While teams run, read the security- and DRY-critical paths yourself: routing/scope resolution, storage derivation, middleware, re-adoption paths. Then verify each top finding against code before accepting it (contra pass): re-read the exact lines, confirm added-vs-context status, check callers, refute what doesn't hold. Use `git diff -U0 <base> HEAD -- <file>` to find ADDED lines for anchors.

## 5. Verification (behavior proof)

- Run the PR's own new/changed suites using the project's documented setup (README/CONTRIBUTING/tox/nox). If the repo has a project rules file, follow it. Always use a virtual environment, never system-wide installs. Re-run after any head move.
- Check every doc claim that matters (status codes, caller lists, allowlist enforcement, "requires X") against code.

## 6. Consolidate

Write a findings ledger (per finding: severity, file:line, source team, `verified` flag) as you collect team reports. Merge duplicates (many teams independently find the same core issues, keep one row, note corroboration). Identify the top 3 must-fix items.

## 7. Post the review

- Auth: `gh auth status` must be the user's account; never expose the token.
- Head: `gh pr view <N> --repo <owner>/<repo> --json headRefOid`; use the *current* head SHA as `commit_id`.
- Anchors: every inline comment must land on a line that exists in the diff; prefer ADDED lines from `git diff -U0`.
- Payload: POST `repos/<owner>/<repo>/pulls/<N>/reviews` with `{"commit_id": ..., "event": "COMMENT"|"REQUEST_CHANGES", "body": "<summary>", "comments": [{"path","line","body"}]}` (via `gh api --input payload.json`). Use `REQUEST_CHANGES` whenever any finding needs a change; otherwise `COMMENT` (no vote unless asked).
- Summary body: verdict, verification results (test counts + head SHA), must-fix list, DRY/prior-art highlights, doc contradictions, credit to good catches (including commits pushed mid-review).
- Comments: label severity (`**Blocking**`, `**Should fix**`, `**Nit**`, `**Doc fix**`), cite the code, give the concrete fix. Write for a normal contributor: explain why it matters in plain terms, do not assume repo history. No AI attribution, no emojis in comments.
- Never `git add`/`commit`/`push`; the review is API-only.

## 8. Verify & report

- `gh api repos/<owner>/<repo>/pulls/<N>/comments` confirm every comment landed, at the right line, under the right author. A pre-existing comment on the same line is corroboration, not duplication, say so.
- Report to the user: review id, state, comment count by severity, anchors, any mid-review head move and how it was handled.
