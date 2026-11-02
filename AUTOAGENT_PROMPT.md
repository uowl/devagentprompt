# Prompt: build "devagent", an autonomous multi-job dev agent with dashboard and job submission

Paste everything below the line into a fresh coding-agent session (Claude Code, Codex CLI, ...) in an empty folder.

---

You are an expert Python engineer. Build **devagent**: a fully autonomous software-development agent that
takes jobs (build a new app, fix a bug, add a feature), runs an LLM coding agent on them (Claude via the Claude Agent SDK, OpenAI Codex via its CLI, or any other
CLI coding agent, chosen per job),
tests and improves the result, commits and pushes to the user's git host (GitHub, Gitea, GitLab, or any plain
git remote), survives usage limits, and
is operated through a login-protected web dashboard where the user submits and manages jobs.

Target: Windows 11, Python 3.15 (run everything with `py -3.15`), standard library wherever possible.
Third-party Python dependencies are only `keyring` and, for the Claude backend, `claude-agent-sdk` (imported lazily so
it is optional when only other backends are used). The Codex backend shells out to the user's installed `codex` CLI. Work in the current directory.
Never ask me questions: choose sensible defaults, write them in README.md, and keep going.
Build in the order given in "Build order", testing each stage before the next.

## 1. Architecture

```
devagent.py     the worker: runs ONE job as phases of LLM-agent work in a project folder (backend-agnostic)
agentbackend.py pluggable LLM agent backends (claude / codex / generic CLI / mock): run a phase, stream activity,
                report cost-or-tokens, classify limit/auth errors; registry in data/backends.json
scheduler.py    long-running supervisor + job queue + CLI (start/stop/status/confirm/resume/reset/submit/jobs/cancel)
jobs.py         job store (JSON on disk), lifecycle, validation, priority ordering
dashboard.py    localhost HTTP server: JSON API + serves dashboard.html
dashboard.html  single-file UI (no build step, no CDN): metrics, current run, kanban, trends, history, job submission
visibility.py   daily digest email, email-command poller, starts the dashboard
telemetry.py    per-phase event log and live "current run" file, stored OUTSIDE the generated repos
gitprovider.py  pluggable git hosts (github / gitea / gitlab / plain git): repo creation, PRs, per-command git auth;
                provider registry in data/providers.json, tokens in Windows Credential Manager via keyring
notify.py       SMTP email with fallback to notifications.log
data/           runtime state (jobs.json, per-project telemetry); projects/ holds generated apps (each its own git repo)
```

The scheduler runs exactly one job at a time (single worker) but accepts any number of queued jobs.

## 2. Jobs and job submission (core feature)

A **job** is a JSON record: `id` (short unique), `kind` (`new` | `fix` | `feature`), `title`, `request`
(free text), `repo` (`{provider: "<registered provider name>", mode: "create"|"existing"|"none", url?, org?, private?}`), `project_name`,
`backend` (registered backend name, default = the default backend), `model` (optional override), `priority` (1-5, lower runs first), `max_budget_usd` (optional), `iterations` (0 = until done), `branch`
(fix/feature), `open_pr` (bool), `status`, `created_at`, `started_at`, `finished_at`, `cost_usd`, `result`
(short text), `attempts`.

Status lifecycle: `queued -> running -> (done | needs_input | failed | cancelled)`; also `paused`.
- `done`: the agent reported nothing worthwhile left (exit code 77) or the iteration/budget cap was reached.
- `needs_input`: waiting for the user (3-day confirmation expired or agent finished and the user may add
  work). A job returns to `queued` when the user resumes it.
- `failed`: three consecutive failed phases or an unrecoverable error; keep the log tail.
- Usage-limit and login problems are NOT failures: the job stays `running`/`queued` and is retried.

Ways to submit a job (all go through `jobs.py`, validated identically):
1. **Dashboard form**: kind, title, request (textarea), **LLM backend + model dropdown** (from the registered backends;
   shows the backend's login state), git provider dropdown (filled from the registered
   providers, with the default preselected; a link to add one), repo mode (create new / existing URL / local only),
   optional org/namespace, private checkbox, priority, optional budget, optional iterations, PR/merge-request
   toggle for fix/feature (disabled with a tooltip when the provider cannot open PRs).
   Reject empty requests, absurd lengths (>8 KB), repo URLs whose host does not match the chosen provider's
   `base_url` (see the host-matching rule in section 3), and project names that are not `[a-z0-9-]{1,40}`.
2. **CLI**: `py -3.15 scheduler.py submit new "build a recipe app" --repo create --private --priority 2`.
3. **Email command** (section 7): `<secret> submit new: build a recipe app`.
4. **Drop-in file**: any `*.job.json` placed in `inbox/` is validated, imported, and moved to `inbox/done/`.

Queue behaviour: pick the lowest priority number, oldest first; one running job; a new higher-priority job
does not interrupt a running phase but runs next. Provide dashboard/CLI actions per job: **cancel**
(kills the running worker process tree, keeps work), **pause/unpause**, **retry**, **reprioritise**,
**add features** (appends to that project's `REQUESTED_FEATURES.md`), **delete** (only if not running).
Jobs and their ordering survive restarts (`data/jobs.json`, written atomically via temp file + rename,
guarded by a file lock or single-writer discipline because the dashboard thread and scheduler both write).

## 3. The worker (devagent.py)

CLI: `devagent.py <new|fix|feature> "<request>" [--project PATH] [--name N] [--provider NAME] [--repo-url URL] [--create-repo]
[--private] [--branch B] [--pr] [--no-push] [--iterations N] [--max-turns N] [--max-budget USD] [--max-tokens N]
[--backend NAME] [--model M] [--safe]`.
The scheduler launches it as a subprocess per job; it must also work standalone.

Phases (each is ONE call to the selected backend with `cwd` = the project, permissions bypassed unless `--safe`,
the backend's file/shell tools enabled, and any "ask the user" tool disabled or answered with "decide yourself"):
- `new`: `build` -> `test` -> `improve-N` repeatedly. If the project already has commits (resume), skip build/test.
- `fix`: reproduce with a failing test -> minimal fix -> full suite -> `verify`.
- `feature`: implement following existing conventions -> tests -> full suite -> `verify`.
After every phase: `git add -A`, commit (`<phase>: complete|partial`), push if a remote exists.

Agent rules (system prompt where the backend supports one, otherwise prepended to the phase prompt; never rely on a
config file the repo would have to commit): work only inside the project; never ask the user anything; write and run
tests; keep README.md and NOTES.md accurate; maintain **BOARD.md** with exactly `## Planned`,
`## In progress`, `## Done` bullet lists every phase; never commit secrets; end every phase with a message
beginning `SUMMARY:` listing finished+tested work, or `SUMMARY: none`.

Improve-phase rules: if `REQUESTED_FEATURES.md` exists, implement every unchecked item first, however large,
ticking `[x]` when done. Never skip an item for being big or needing something external: build the largest
locally-working part (interfaces, mock/local implementation, tests) and tick it with `(partial: what is left)`.
Large backlog items are split into steps; ship a working tested increment each round.
Do not invent work; if the backlog (including large items) is truly empty, change nothing and end with
`NO_FURTHER_IMPROVEMENTS` and `SUMMARY: none`; do not touch NOTES/README in a no-change round.

Stop conditions and exit codes: `0` normal end; `75` usage/rate limit hit; `76` agent/backend auth failure
(login expired / invalid key / not logged in); `77` no further improvements. Stop at 77 only when the sentinel is given or two
consecutive improve rounds report `SUMMARY: none` AND no requested feature is still unchecked (if some are,
allow up to four empty rounds). Also stop at `--max-budget`, `--iterations`, or three consecutive failed phases.
Each backend supplies its own error classifier (text/exit-code patterns) mapping a failed phase to `limit`, `auth`
or `other`; stop immediately on `limit` and `auth`. Typical patterns: limit = `usage limit`, `rate limit`, `quota`,
`credit balance`, `429`; auth = `failed to authenticate`, `oauth session expired`, `invalid api key`, `not logged in`,
`401`, `please run codex login`.

**Agent backends (agentbackend.py).** Interface every backend implements:
`run_phase(project, phase, prompt, system_rules, options) -> AsyncIterator[Event]` ending with a final
`Result{ok, text, cost_usd|None, tokens_in|None, tokens_out|None, turns|None, reason: None|limit|auth|other}`;
intermediate `Event{kind: text|tool|log, detail}` feed the live activity log. `auth_status() -> logged_in|not_logged_in|
unknown` (shown in the dashboard), `classify(text, exit_code) -> reason`, and `cancel()`. Backends are registered by name in
`data/backends.json` (`{name, type, model, default, limit_window_hours, retry_minutes, ...}`), never hard-coded, and
secrets (API keys) come from keyring/env, never that file.
- **claude**: `claude-agent-sdk` `query()`; tools Read/Write/Edit/Glob/Grep/Bash, `permission_mode=bypassPermissions`
  unless `--safe` (`acceptEdits`), `AskUserQuestion` disallowed; cost comes from the result message. Auth: Claude Code
  login or `ANTHROPIC_API_KEY`.
- **codex**: spawn `codex exec --json --cd <project>` (non-interactive) with the prompt on stdin or as the argument,
  `--model` when set, and `--dangerously-bypass-approvals-and-sandbox` (or `--sandbox workspace-write` with `--safe`);
  parse the JSONL event stream into text/tool events and take token usage from it (cost is `None`: estimate only if the
  user configured per-token prices). Auth: `codex login` or `OPENAI_API_KEY`; `auth_status` via `codex login status`.
  Treat a missing `codex` binary as an actionable error, not a crash.
- **cli (generic)**: any agent CLI (Gemini CLI, aider, a local-model wrapper...) described in the registry as an argument
  list template with `{prompt_file}`, `{cwd}`, `{model}` placeholders (arguments are passed as a list, never through a
  shell string); success = exit code 0; stdout lines become activity; limit/auth patterns configurable per backend.
- **mock**: scripted backend that replays canned events/exit conditions; used by every automated test so no real model,
  network or money is needed.
Budget enforcement is backend-aware: `--max-budget` uses `cost_usd` when known; when a backend reports only tokens, use
`--max-tokens`; if neither is available say so in the dashboard instead of showing a fake `$0.00`.
Different backends have different usage-limit windows, so cadence is per backend (`limit_window_hours`, default 5;
`retry_minutes`, default 30), not a global constant. Optional per-phase model overrides (stronger model for
`build`/large features, cheaper for routine `improve` rounds) via `models: {build: ..., improve: ...}` in the registry.
The phase prompts, BOARD.md/REQUESTED_FEATURES.md/SUMMARY protocol, exit codes and stop logic above are identical for
every backend; if a weaker backend ignores the `SUMMARY:` convention, treat a missing summary as `SUMMARY: unknown`
(commit, but do not count it as an empty round).

Known pitfalls to design around (the first two are Claude-SDK specific, the rest apply to every backend):
- Claude SDK: do **not** `return` from inside `async for msg in query(...)`; let the stream finish, then return, or the
  SDK logs "aclose(): asynchronous generator is already running".
- Do not call `sys.exit` inside the async `main`; return an exit code and call `sys.exit(asyncio.run(main()))`.
- Wrap every phase in try/except so a backend/SDK/network/CLI crash becomes a recorded failed phase, not a dead worker.
- Skip pushing when there are no commits yet; email once (not per phase) if a push fails.

**Git hosting is pluggable (gitprovider.py).** devagent must work with GitHub, Gitea (and Forgejo/Codeberg),
GitLab, and any plain git remote, including self-hosted servers. Never hard-code a host.

*Provider registry*: `data/providers.json` holds named hosts, e.g.
`{"name": "home-gitea", "type": "gitea", "base_url": "https://git.example.lan", "verify_tls": true, "default": true}`.
`type` is one of `github | gitea | gitlab | git`; `github` defaults to `https://github.com`. Tokens are NOT in this
file: they live in Windows Credential Manager (keyring service `devagent-git`, account = provider name), with
`DEVAGENT_TOKEN_<NAME>` (upper-cased, `-` -> `_`) as an override, and `GITHUB_TOKEN` honoured for a provider named `github`.

*Interface* every provider implements: `login()` (hidden-prompt token, validated via the whoami call before it is
stored), `whoami()`, `create_repo(name, private, description, org=None) -> clone_url`,
`open_pr(remote_url, head, title, body) -> url` (looks up the default branch first), `parse_remote(url) ->
(owner, repo) | None`, and `git_auth_args(remote_url) -> ["-c", ...]`. API details:
- **GitHub**: `https://api.github.com`, `Authorization: Bearer`, `POST /user/repos`, `POST /repos/{o}/{r}/pulls`.
- **Gitea/Forgejo**: `{base_url}/api/v1`, `Authorization: token <T>`, `GET /user`, `POST /user/repos` (or
  `/orgs/{org}/repos`), `GET /repos/{o}/{r}` for `default_branch`, `POST /repos/{o}/{r}/pulls` with `head`/`base`.
- **GitLab**: `{base_url}/api/v4`, `PRIVATE-TOKEN`, `POST /projects`, merge requests via
  `POST /projects/{id}/merge_requests` (call it a "PR" in the UI).
- **git (generic)**: no API. Push/clone only, using a token over HTTPS, an SSH URL with the user's own agent/keys,
  or an already configured credential helper; `create_repo` and `open_pr` are reported as unsupported with a clear
  message telling the user to create the empty repo manually and give its URL.
Auth for git itself is injected per command, never stored in the remote URL, a file or a log line:
`git -c http.<base_url>/.extraheader="AUTHORIZATION: basic <b64 user:TOKEN>"` (GitHub uses user `x-access-token`;
Gitea/GitLab use the `whoami` username, cached). SSH remotes need no injection.

*Host-matching safety rule (important)*: a job's repo URL is attacker-influenced text (dashboard, email command,
drop-in file). Only attach a token to a request or git command when the URL's scheme+host+port exactly matches the
registered provider's `base_url`; otherwise refuse the job with a clear error. Accept `https://` (and `http://`
only for loopback/RFC1918 hosts the user registered explicitly); never follow redirects to another host with
credentials; `verify_tls: false` must be opt-in per provider and shown as a warning in the dashboard.

*CLI* (`py -3.15 gitprovider.py ...`): `add <name> --type gitea --url https://...`, `list`, `login <name>`,
`whoami <name>`, `logout <name>`, `remove <name>`, `default <name>`. `devagent.py` gets `--provider NAME`
(default: the default provider, or inferred from `--repo-url` by host match). A resumed job with an existing
`origin` must not recreate the repo; a resumed job whose provider was removed fails with an actionable message.
Document per-provider token tips: GitHub fine-grained tokens often cannot create user repos (classic token with
`repo` scope works) and a 403 on push means missing Contents-write on that repo; Gitea needs a token with
`write:repository` (and `write:user` to create repos); GitLab needs `api` (or `write_repository` for push only).

## 4. Scheduler (scheduler.py)

A single long-running process (PID file; refuse a second instance; `stop` kills the process tree).
Loop: check approval window -> pick next job -> run worker -> interpret exit code -> sleep -> repeat.
- **Cadence**: after a normal run wait the backend's `limit_window_hours` (default 5); if the worker exited 75 (limit) or
  76 (auth) retry every `retry_minutes` (default 30) until it works. Sleep in short slices (`nap()`) that wake within ~2 s when the state file changes,
  so `resume`/`confirm`/a newly submitted job takes effect immediately.
- **Approval window**: every 3 days the scheduler stops launching and waits for `confirm`
  (dashboard button, CLI, or email). Writes `AWAITING_CONFIRMATION.txt`.
- **Finished jobs**: exit 77 marks the job `done` (or `needs_input` when the user may add work), emails the
  user, and the scheduler moves to the next queued job; with an empty queue it idles until a job arrives.
- **No-response timeout**: if the scheduler is waiting on the user (confirmation or finished) for more than
  1 hour it emails and exits; `start` restarts the clock.
- **Alerts (email, deduplicated until new progress)**: feature/phase ready (with the agent's summary and repo
  link, skipped for `SUMMARY: none`), job finished/failed, no commit for 5+ hours (limit or confirmation),
  `<backend> login required`, push failed, daily digest (24 h: status, commits in the last day, open features).
- CLI: `start [--resume]`, `stop`, `status`, `confirm`, `resume ["new feature" ...]`, `submit ...`,
  `jobs`, `cancel <id>`, `reset`. Registering/queueing a job never blocks on user input.

## 5. Dashboard (dashboard.py + dashboard.html)

Serve on `127.0.0.1:8765` only (`DEVAGENT_DASH_PORT`), HTTP basic auth (`DEVAGENT_DASH_USER` default `admin`,
`DEVAGENT_DASH_PASSWORD` required; the dashboard is disabled with a clear message without it). Constant-time
credential comparison. State-changing POSTs require a custom header (`X-Devagent: 1`) and a localhost `Host`
header (basic-auth credentials are sent automatically by browsers, so this blocks cross-site forms). Security
headers incl. a CSP that only allows inline script/style and same-origin connections. Escape all dynamic text
(agent output is untrusted).

Endpoints: `GET /api/data` (everything the page needs, selectable job), `GET /api/jobs`, `POST /api/jobs`
(submit), `POST /api/jobs/<id>/<action>` (cancel, pause, unpause, retry, priority, add-features, delete),
`POST /api/action` (resume, confirm, stop). Cache expensive git statistics for ~30 s.

Page (single file, vanilla JS, light/dark with a saved toggle, responsive down to phone width, keyboard
accessible, toasts for every action, 5 s auto-refresh with an "updated/offline" indicator):
- **Header**: selected app/job, status pill (working / waiting for usage limit / needs confirmation /
  finished / login required / stopped), Resume, Confirm 3 days, Stop, theme toggle.
- **Jobs panel**: queue table (status, kind, title, priority, cost, age) with per-row actions, a
  **Submit job** form (all fields from section 2, client + server validation, inline errors), and the ability to
  switch the dashboard to any job's project.
- **Metrics cards**: total spend, commits (+week-over-week delta), phases run and success rate, lines added/removed
  and file count, average phase time/cost/turns, last run, approval-window expiry; sparklines on spend and commits.
- **Current run**: phase, elapsed timer, live activity log with a "follow" toggle.
- **Kanban board**: Requested (from `REQUESTED_FEATURES.md`) / Planned / In progress / Done (from `BOARD.md`),
  per-column pagination (5/10/20), a filter box, click-to-expand cards.
- **Trends**: line charts (hand-written inline SVG) for commits/day, cost/day, cumulative commits, cumulative cost
  over 7/14/30 days, and per-phase cost/duration/turns for the last 40 phases; each draws the actual line, a moving
  average (dashed) and a linear trend (dotted), with hover tooltips and a guide line.
- **History**: tabs for past phases (expandable rows with the full agent summary) and commits; search, result filter,
  pagination (12/page).
- **Add features** box that appends to `REQUESTED_FEATURES.md` and resumes.

## 6. Telemetry (telemetry.py)

Outside the generated repos, under `data/<project>/`: `events.jsonl` (one record per finished phase: timestamp,
phase, backend, model, ok, cost (nullable), tokens in/out (nullable), turns (nullable), seconds, summary, failure reason) and `current.json` (running phase, start time, rolling
60-line activity log written on every tool call). Telemetry must never break a run (swallow I/O errors).

## 7. Email (notify.py, visibility.py)

SMTP via `DEVAGENT_SMTP_USER/PASSWORD/HOST/PORT` (Gmail App Password), recipient `DEVAGENT_EMAIL_TO`
(default `jwalin.pandya+devagent@gmail.com`); if unset or failing, append to `notifications.log` instead.
**Email commands** via IMAP: unread mails with "devagent" in the subject whose first body line is
`<secret> <command>` (`status | resume | confirm | stop | submit <kind>: <text> | add <feature>`); run only if
BOTH the sender is in `DEVAGENT_COMMAND_FROM` AND the secret (>= 12 chars, `DEVAGENT_COMMAND_SECRET`) matches in
constant time; strip quoted replies; reply with the result; disabled unless all settings exist.
Treat `submit`/`add` text as untrusted input that becomes an agent prompt: length-limit it.

## 8. Security and safety requirements

- The worker runs the LLM agent with all permissions/approvals bypassed: say so prominently in README, recommend a VM/sandbox or
  dedicated Windows user and a repo-scoped git-host token, and offer `--safe`. Tokens are only ever sent to the
  host registered for their provider (exact host match), never to a host taken from job text.
- Never print or persist secrets (tokens, SMTP password, dashboard password, email secret).
- Dashboard bound to loopback only; no CORS; no eval; no external scripts or fonts.
- Validate every job field on the server; never build shell commands from job text (use argument lists).
- Atomic writes for all state files; tolerate a corrupt/missing state file.

## 9. Windows scripting gotchas

When generating or patching code through the shell, avoid string-escaping bugs (a `\n` inside an f-string being
turned into a real newline broke several files): write source files with the editor/Write tool, keep HTML in its
own `.html` file, and run `py -3.15 -c "import ast;ast.parse(open(f).read())"` on every file after edits.
Use `taskkill /T /F` to stop a process tree. Use `tasklist /FI "PID eq N"` to test liveness.

## 10. Build order and acceptance tests

1. `gitprovider.py`, `notify.py`, `telemetry.py`: unit-test with fakes (no real tokens/SMTP). For providers, run a
   local fake HTTP server per API style (GitHub-like and Gitea-like) and assert: correct endpoints and auth headers,
   token never appears in the remote URL/log output, a URL on a different host is refused, plain-git reports
   "unsupported" for create/PR. If Docker or a local Gitea binary is available, also do one real push to it.
2. `devagent.py`: dry run on a trivial request with `--max-budget 1 --no-push`; verify commits, BOARD.md,
   events.jsonl, SUMMARY parsing, exit codes 75/76/77 via the `mock` backend; then one real phase per installed backend
   (claude, codex) on a trivial request, skipping any that is not installed/logged in and saying so. Unit-test the codex
   JSONL parser and each classifier against recorded sample output.
3. `jobs.py`: submit/validate/order/cancel/retry/persistence tests, including concurrent writers.
4. `scheduler.py`: simulate worker exits 0/75/76/77/1 with a stub worker; verify cadence, 30-minute retry,
   3-day window, 1-hour no-response exit, instant wake on state change, single-instance PID guard, `stop`.
5. `dashboard.py` + `dashboard.html`: test 401 without auth, 403 without the custom header, input validation,
   job submit/cancel through the API, then screenshot the page with demo data in light and dark at desktop and
   phone width and fix layout issues. No console errors.
6. `visibility.py`: digest content, email-command parsing (quoted replies, wrong sender, wrong secret).
7. README.md: setup, env var table, every CLI command, security notes, troubleshooting (403 on push, OAuth expired,
   usage-limit behaviour, how to restart after reboot with Task Scheduler).

Definition of done: I can run `py -3.15 scheduler.py start`, open the dashboard, submit two jobs of different
kinds with different priorities, watch the first run live, cancel/pause/reprioritise from the UI, see usage-limit
waits recover on their own, receive the emails described above, and every behaviour in this prompt is covered by a
test or a documented manual check. Report what you could not verify.
