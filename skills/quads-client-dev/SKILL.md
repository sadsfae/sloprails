---
name: quads-client-dev
description: Development workflow for quadsproject/quads-client (and its python-quads-lib dependency): branch off development, thin-client design, server-side validation, schedule date parsing conventions, testing and live-verification patterns. Use for any quads-client TUI/GUI/CLI work, schedule/assignment features, or quads-lib integration.
---

# quads-client development

## Branch and PR flow

- New work branches off `origin/development`; PRs target `development`. `main` is the release branch where RPMs are built (0.x.y tags; release notes link RPMs).
- Open the GitHub issue first, cite it in the commit (`fixes:` line, 3-space indent).
- One issue, one branch, one commit. Semantic message (`feat:` / `fix:`), summary under 50 chars, no AI attribution.
- Never use real redhat.com hostnames in docs, commits, issues, or PR text; use placeholders like `host01.example.com`.
- Git/PR mechanics (independent reviews, rebase merges, branch deletion) follow skill://github.

## Architecture rules

- quads-client and quads-client-gui are thin wrappers over the python library (`quads_lib`). Use `connection.api` / `quads_lib` methods (e.g. `create_schedules_batch`, `filter_clouds`, `is_available`) instead of re-implementing domain logic in the client.
- The client owns UX: argument parsing, display, error rendering. The server owns domain validation: date format/range, availability, assignment state. Prefer server API guards over client-side guards; do not stack redundant client validation.
- The GUI routes through the same command layer (`gui/views/admin_schedule.py` builds an args string and calls `cmd_schedule_admin`), so parser or command changes affect both surfaces.

## Admin schedule command (lessons from the strptime(QUADS-204) bug)

- Parsing contract: dates may appear anywhere after the host list; options are order-independent. A single date is the end date, with start defaulting to `now`.
- Zero, more than two, or malformed date tokens raise a clean `ValueError`. Never let `start`/`end` reach `datetime.strptime(..., None)` (the historical `Scheduling failed: strptime() argument 1 must be str, not None` crash).
- Dates are quoted `"YYYY-MM-DD HH:MM"`; the server accepts only that format or `now`. Malformed shapes (unquoted date+time, single-digit fields, `/` separators) must error loudly, never be silently absorbed into `description`.
- Date detection is shape-based (`now`, `YYYY-MM-DD[ HH:MM]`), no extra flags. Option values are never dates: only value-taking keywords (`description`, `cloud-owner`, `cc-users`, `cloud-ticket`, `vlan`, `qinq`, `os`) exclude the next token from date detection. Flags such as `nowipe` never take a value.
- The server's `POST /schedules/batch` requires both dates, so no implicit "now to +2 weeks" default exists; do not invent one client-side without a server change.

## Verification

- Local tests: `PYTHONPATH=src <venv>/bin/python -m pytest tests/ --cov=quads_client --cov-report=term`. Must be green with no coverage decrease (full suite only; the repo's pytest.ini assumes pytest-cov).
- NEVER run tests without `PYTHONPATH=src`: the installed RPM copy shadows the checkout and yields false results.
- Tooling venv: `python3 -m venv --system-site-packages <venv>` then `pip install pytest pytest-cov==7.1.0 black==24.4.2 flake8`. Repo pins black 24.4.2 (line-length 119); flake8 max-line-length 119.
- Interactive smoke: `PYTHONPATH=src python3 -c "from quads_client.shell import QuadsClientShell; shell = QuadsClientShell(); shell.cmdloop()"`.
- One-shot / piped: `printf 'connect <server> <command>\n' | PYTHONPATH=src python3 -m quads_client.cli.main` (quotes survive in piped mode; from a shell argv they do not, so `"2026-11-03 09:10"` splits into two tokens in true one-shot mode).
- Live testing against the dev server: proxy comes from `~/.config/quads/quads-client.yml` (per-server `socks5h://`). Never mutate without an explicit cloud/host/ticket from the user. Failure paths (typo/past dates, unknown cloud) are safe: the server rejects before any write. Verify reads via `ls-schedule host <host>`, `assignment-status <id>`, `ls-vlan`.
- `terminate <id>` prompts for confirmation; in piped mode answer `y` on the next stdin line.
