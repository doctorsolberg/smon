# smon — Roadmap

Process (details — `AGENTS.md` §1): an **orchestrator** agent runs the stages in order. It delegates every checkpoint to an
**implementer** sub-agent working in its own git worktree on branch `stage/Sx.n`. The implementer implements, runs all checks,
pushes and opens a draft PR at the first push. The orchestrator waits for a green CI, has a **reviewer** sub-agent review the PR,
sends findings back if needed and merges the PR itself (squash). The acceptance run and a release tag happen once per stage;
the owner verdict follows (`AGENTS.md` §1–§2).

References like §N point to sections of `docs/SPEC.md`. The Read list of a stage is the mandatory minimum of the SPEC for the
implementer. Every stage also reads §1.2, §3.3–3.5 and `AGENTS.md`; work on the database or the UI also reads §5.1–5.2 and
§12.1–12.4.

## General Definition of Done

**Checkpoint (PR):**
1. CI is green (SPEC §16.7): ruff, mypy, import-linter, pytest (including the schema test, §5.3), `guard`, image build and
   smoke test; from S4.2 the `windows` job; from S5.2 the collector tests under Python 3.8.
2. If models changed — a migration (§5.3). A migration not yet included in a tag may be edited; one included in a tag may not.
3. Checks were not weakened (`AGENTS.md` §4). Modifications of files protected by `guard` carry a `Spec: D<n>` or
   `Planned: Sx.n` line in the PR body.
4. `docs/STATUS.md` is updated **in the PR itself** (the implementer's last commit before `gh pr ready`); decisions outside
   the SPEC are recorded as D records with status `proposed` in `docs/DECISIONS.md`; SPEC edits go into the same PR, and the PR body and the
   squash commit body contain a line `Spec: D<n> …` for each of them.
5. Every new loop has its `LoopSpec` in `loops/ticks.py` (`build_loops`).
6. The review (by a reviewer sub-agent) passed; findings are closed.
7. Each [test] acceptance scenario of the stage is written as an integration test in the checkpoint that completes the
   modules it needs; the orchestrator names these scenarios when delegating.

**Stage (additionally):**
8. The acceptance run: the orchestrator runs the [test] scenarios and executes the [stand] scenarios; the record is
   `docs/acceptance/Sx.md` (template — `AGENTS.md` §8, including the "SPEC changes" block). Scenarios labelled [owner] are
   turned into owner questions `Q-Sx-N` and do not block the tag unless marked "blocking". The closing PR adds no code.
   A failed [test] or [stand] scenario: the orchestrator records it, adds a fix checkpoint `Sx.<next>` to the ROADMAP and
   repeats the acceptance run after the fix is merged; a closing PR never contains a failed [test] or [stand] scenario.
9. The closing PR of the stage `stage/Sx-release` contains the acceptance record, STATUS and the `project.version` bump. Right
   after its merge the orchestrator tags that commit `v{project.version}` (S0 → `v0.1.0`, S1 → `v0.2.0`, …, S6 → `v0.7.0`,
   S7 → `v1.0.0`), provided the owner verdict on the previous stage is recorded; `release.yml` publishes the image `X.Y.Z`
   (amd64 + arm64). Checks that need the tag (e.g. an anonymous `docker pull`) are appended to the acceptance record by a
   follow-up docs PR.
10. **Drift limit (D74, D101):** while the owner verdict on stage n is pending (the "Owner verdict" line in
   `docs/acceptance/Sn.md`), the orchestrator may complete stage n+1, including merging its closing PR, but must not tag it or
   start stage n+2. With more than 5 unreviewed `proposed` D records it stops and asks the owner.

**Acceptance labels:**
- **[test]** — an integration test `tests/integration/…` on `FakeClock` (the whole `AppContext`, `FakeClock.run_for`,
  `FakeTelegram`, a substituted `run_probe`, a synchronous `run_blocking`). Scenarios with deadlines of minutes or hours are
  checked this way.
- **[stand]** — `devtools/compose.dev.yml` with accelerated parameters (`interval_s=30`, `retry_interval_s=10`) and the smon
  image built locally from the accepted commit (needs Docker where the orchestrator runs). devtools run in compose from that
  image with `./devtools:/devtools:ro` (`python /devtools/<x>.py`). Stand objects are created through the admin UI with
  `curl` (a cookie jar, the `csrf_token` taken from the form, a created token read from the 200 response); from S5.3 a stand
  may import a prepared export file.
- **[owner]** — needs real infrastructure (including test VMs) or an owner decision; the orchestrator writes the steps and
  the expected result, the owner runs them.

## Dependency placement

| What | Stage | Why |
|---|---|---|
| stubs of all layer modules (§3.2), `.importlinter`, `guard`, the PR template | S0.1 | the layers and the process are set up once |
| settings registry, `set_many`, `updated_at`, `init_internal` | S0.1 | the log limit is needed in S0 acceptance |
| `AppContext` (`run_blocking`, `disk_usage`, `on_settings_changed`), `RuntimeState`, `domain/health`, `schemas/errors`, SAFE_MODE core, `build_loops` | S0.2 | health, 404/405, env validation, shutdown |
| `SMON_ADMIN_ALLOW`, the generic `/settings` | S0.3 | the admin UI |
| **all tables** (migration `0002`) | S1.1 | parent tables are never recreated (D33) |
| `self:telegram` and the undelivered banner | S1 | S1 acceptance |
| parent logic in the engine | S1.1 | tested on synthetic keys; S5 adds only the host evaluator |
| `notify/telegram`, `ctx.telegram_client` | S1.2 | the Telegram test message in S1.3 |
| `summary.collect` + `SummaryData` (Overview), `render_test` | S1.3 | Overview and the Telegram test are in S1.3 |
| all sender rules of §10.1 including reminders and the end of a mute or silence | S1.4 | one place for level-triggered logic |
| `probe_now`, `sweep_at`/`down_from` | S1 | "check now" and the downtime shift are in S1 |
| the poller; daily summary rendering and sending | S2 | `collect` already exists |
| read tokens, the read API (`status`, `incidents`, `signals`, `logs`) | S3 | integrations |
| job tokens, `domain/schedule`, jobs in `catalog.expected` and the sweep, `jobs/{id}/runs`, ingest 503 in SAFE_MODE | S4.1 | the first public ingest |
| host tokens, hosts in `catalog.expected`, `hosts/{id}`, the collector release archive | S5 | — |
| export and import of all sections | S5.3 | all configuration objects exist (D92) |

---

## S0 — Skeleton

**Goal:** a runnable container with a working admin login, the process foundation, CI and release.

**Read:** §1, §2.3, §3, §4.1 (RuntimeState, TelegramSettings), §4.4, §4.6, §5.1–5.4 (tables 0001), §11.1, §12.1–12.3,
§12.5 (S0), §12.7 (settings widgets), §13.1, §13.3, §13.4, §13.6, §14, §15.2, §15.4, §15.5, §16.

**Checkpoints:**
- **S0.1 — core:** `pyproject.toml` (ruff, mypy, pytest configuration), `uv.lock`, `.importlinter`; the package and all layer
  stubs; `config` (env validation), `constants`, `clock`, `log`, `fmt`; `db` (`make_engine`, `Database`, nested-tx guard,
  table models 0001, migration `0001`, `env.py` with the FK-off engine, Alembic `Config()` in code); `settings` (registry,
  `get`, `set`, `set_many`, `updated_at`, `init_internal`), `audit`; `domain/types` (`RuntimeState`, `TelegramSettings`),
  `schemas/errors`; `tests/conftest.py` (including the `linux_only` marker), `tests/fakes.py` (`FakeClock` §4.4),
  `tests/factories.py`; `.github/workflows/ci.yml` (checks, without the image job), `.github/workflows/guard.yml`,
  `devtools/guard.py`, `.github/pull_request_template.md`; GitHub Actions pinned by commit SHA (SPEC §16.7).
- **S0.2 — process:** `context` (`AppContext`, `run_blocking`, `disk_usage`, `on_settings_changed`; later checkpoints extend
  it); `cli` (`serve`, `db upgrade`), `main` (startup order, flock, signals, the uvicorn subclass,
  `FIRST_COMPLETED`); `domain/health`, `domain/netaddr`; `loops/supervisor`, `loops/ticks.build_loops` (empty); `web/dispatch`
  (request_id, `X-Request-ID`, request log), empty `ingest_app`/`admin_app` with error handlers (404/405 JSON/HTML), `/healthz`;
  the SAFE_MODE core (no supervisor, `/healthz` → 503); `tests/e2e`.
- **S0.3 — web and delivery:** `web/auth` (password hashing, login, fixed-lifetime sessions, logout, login limiter),
  `smon admin set-password`, `web/csrf`, `web/forms`,
  `web/allowlist`, `web/deps`; layout, login page, generic `/settings` (`POST /settings/s/{section}`), Overview placeholder,
  the SAFE_MODE banner; `Dockerfile`, `.dockerignore`, `deploy/docker-compose.example.yml` (§16.6), the image job of `ci.yml`
  (smoke test, `sha-<7>` on `main`; `Planned: S0.3`), `release.yml`.

**Out of scope:** checks, signals, Telegram; any UI except the layout, login, settings and the Overview placeholder.

**Mandatory tests:**
- flock (a second instance → exit 1); startup order; `main` with `FIRST_COMPLETED` → exit 1 on an unexpected task exit;
  `Supervisor.run()` with no loops returns only after `stop()`; SIGTERM → exit 0 ≤ 8 s (e2e subprocess, verifies the uvicorn
  subclass); SIGTERM during migrations → exit 0: `tests/e2e/driver.py` replaces `smon.main.upgrade` with a stub that writes a
  ready file in `tmp_path` and blocks for 2 s, then calls `smon.main.serve()` (no test-only `SMON_*` variables);
- port dispatch (the UI is not reachable on :8080); 404/405 as JSON and HTML; `X-Request-ID` on every response, including 500;
- CSRF (session token, htmx header; `POST /login` is exempt); sessions: sha256 in the database, the fixed 14-day lifetime,
  logout, `set-password` deletes sessions; the login limiter works before the password check;
- `SMON_ADMIN_ALLOW` → 403 (`ASGITransport(client=…)`); `client_ip`;
- env validation: an unknown `SMON_*` → exit 2; an unknown database revision → exit 1; no write access to `/data` → exit 2;
- database: two threads with `BEGIN IMMEDIATE` (the second waits); a nested `tx()` → `RuntimeError`; the schema test (§5.3);
  a batch migration of a parent keeps its CASCADE children (synthetic tables); `PRAGMA foreign_keys == 0` in migrations;
- logs: `env.py` without `fileConfig` (after migrations `smon.*` still logs); token redaction; the closed field list (`dropped`);
  the event name format; the `/data/logs` size limit;
- supervisor: a failing tick → the loop is stale in `domain.health.evaluate`; `CancelledError` propagates;
- settings: `set_many` (validation, type coercion, only changed keys, returned changes), `updated_at`, `on_settings_changed`;
- SAFE_MODE: the supervisor does not start, `/healthz` → 503, the banner;
- `guard`: unit tests of `devtools/guard.py` (added files ignored, a changed `tool.ruff` table detected, `Spec:` and
  `Planned:` lines accepted).

**Acceptance:**
1. [stand] A container with a named volume and `SMON_ADMIN_HTTPS=0` → `smon admin set-password --stdin` → login on :8081;
   no UI on :8080.
2. [stand] A second instance on the same volume does not start; the message is clear.
3. [stand] The password and settings survive a restart.
4. [stand] `docker stop` → exit 0 in < 10 s.
5. [stand] A `curl` loop against :8080 with `logs.max_total_mb=10` → the size of `/data/logs` stays within the limit.
6. [stand] The CI image passes the smoke test; after the tag and after the owner has made the GHCR package public — an
   anonymous `docker pull` and `docker manifest inspect` show amd64 and arm64 (appended after the tag).

---

## S1 — Core: checks → incidents → Telegram

**Goal:** the first useful feature — external checks with notifications.

**Read:** §1, §2.2–2.3, §3–§7, §10.1–10.4 (without sending the daily summary), §11.1, §12, §13.1–13.4, §14, §15.2, §15.5, §16.4.

**Checkpoints:**
- **S1.1 — core:** migration `0002` (**all remaining tables of §5.4**); `domain/*` in full (`status`, `kinds`, `keys`, `titles`,
  `text`, `types`); `schemas/checks`; `evaluators/checks` (+ OK and TLS texts), `deadlines`, `self` (`self:telegram` only);
  `engine/*`; `catalog/checks`, `catalog/expected`; `loops/sweep`; `queries` (including `suppression`); the golden file
  `tests/golden/check_texts.txt`.
- **S1.2 — probes and the Telegram client:** `probes/*` (+ `texts`, the golden file `tests/golden/probe_texts.txt`),
  `loops/probe_scheduler` (`probe_now`); `notify/telegram` (client and errors), `ctx.telegram_client`.
- **S1.3 — UI:** `notify/summary.collect`, the types of `notify/render` (`SummaryData`, `Item`, `Part`) and `render_test`;
  the test message; UI: Overview (via `summary.collect`), Checks (CRUD + "check now"), Incidents (+ incident page), `/signal`,
  Silences, Settings → Telegram (+ test), Audit.
- **S1.4 — notifications:** `notify/render` (templates §10.3, `render_block`, `split_parts`), `notify/sender` with **all rules
  of §10.1** (including reminders and the end of a mute or silence), the sender tick; banners; `devtools/fake_telegram`
  (Bot API: `sendMessage`, `getUpdates`, `answerCallbackQuery`; checks the token; message journal; an HTTP endpoint to emulate
  button presses), `devtools/flaky_target` (`/set?status=500`), `devtools/compose.dev.yml`; `tests/integration`.

**Mandatory tests:**
- engine: table tests (debounce, `unknown` without debounce, W↔C as one class, escalation clears the ack, no-data with and
  without a parent, reconcile → removed, a result dropped for a disabled check, the last bad reason);
  **property tests of invariants I1–I5** with `hypothesis`; a repeated sweep without changes = 0 UPDATEs; the downtime shift (§6.6);
  sweep order `reconcile → apply → open_no_data`;
- sender: the severity snapshot, suppression by silence and parent, "resolved" is not suppressed, announcing after a silence
  ends, the reminder interval, the reminder at the end of a mute, splitting (UTF-16, part header, `Part.block_idx`), marking
  after each part, stopping after a failed part, backoff and 429, `undelivered_since`; a client is opened only with candidates;
- golden tests of the templates §10.3 and the texts §7.4 (probe errors, OK and TLS texts);
- probes against local servers (`respx`, `trustme`): whole-probe timeout, an expired certificate, DNS normalization;
- the log contains no bot token (a send through `respx` with an error);
- the batch migration `0002` on real tables keeps the children.

**Acceptance:**
1. [test] flaky-target → 500: critical ≤ 8 min (§2.2), back to 200 → resolved. [stand] the same with accelerated parameters.
2. [stand] A wrong bot token during an outage → the "undelivered" banner; after the fix — one message with the current state.
3. [test] A silence during an outage → no message; when the silence ends, "opened" arrives.
4. [stand] A restart with an open incident → no duplicates.
5. [test] The monitor is stopped for 30 min → after the start, 0 messages within 15 min while the targets are healthy.
6. [stand] The log shows the INFO chain for a signal key: `signal.status_changed` → `incident.opened` → `notify.sent`.

---

## S2 — Telegram interaction

**Read:** §2.3, §3.4 (RuntimeState), §4.2–4.3, §4.5, §5.4 (`tg_messages`), §6.1, §6.3, §10, §13.1–13.3, §14.4, §15.5.

**Modules:** `notify/poller`, the poller tick; the daily summary (`render_summary`, `render_status`) and its sending in the
sender.

**Mandatory tests:** poller (chat and user_id filter, ack and mute only on the message's incidents, the offset in one transaction
with the action, an exception on an update does not block the queue, the token check before writing the offset, 409 →
`tg_poll_error`, `poll_failing_since` and reset on success, a client per long poll, `/status` with a bot suffix); an ack from
Telegram stops the reminders; the daily summary (time condition, catch-up, the `summary_sent_at` window, all sections, golden,
the summary as a candidate of the undelivered queue).

**Acceptance:**
1. [stand] Ack from (fake) Telegram → the incident has `acked_at` and an audit record. [test] The ack stops the reminders.
2. [test] Mute 1h → silence → after it ends, a reminder if the problem persists.
3. [test] The daily summary lists acknowledged incidents, silences, unknown signals and disabled objects.
4. [stand] A button pressed by a foreign user_id → "Access denied".
5. [stand] `/status` answers with the current state.
6. [owner] A real bot: the test message, Ack, `/status`.

---

## S3 — Reliability, read API and the first real deployment

**Read:** §2.3, §3.4, §4.1, §4.3, §5.1, §5.3, §5.4, §6.6, §7.1, §11.1, §11.3, §12, §13, §14.6, §15.1, §15.5, §16.

**Checkpoints:**
- **S3.1 — reliability:** `ops/deadman` (the first ping only after 120 s of uptime), `ops/snapshots`, `ops/retention`;
  `evaluators/self` in full (all four self-signals implemented); the premigrate snapshot (a failure → no migration, exit 1);
  restore (`docs/restore.md`, created here); a dead-man receiver in `devtools/flaky_target` (`/ping/*` with a request journal).
- **S3.2 — integrations:** `catalog/tokens` (read), `web/api` (read API §11.3: `status`, `incidents`, `signals`, `logs`),
  `ops/logs_reader` and the UI "Logs"; UI: tokens, snapshots.

**Mandatory tests:** health (a stale loop, undelivered > 600 s, SAFE_MODE); dead-man (no ping during the first 120 s of
uptime, ping and fail, an error → `self:deadman`, `deadman_url_set_at`); self-signals (fixed `disk_usage`, a repeated sweep
writes nothing, the OK texts); snapshot (autocommit connection, the space check, integrity, streaming gzip, UTC stamp, rotation
in `general.timezone`, retry at most once an hour); a failed premigrate snapshot → exit 1 and the database is not migrated;
read API (401 without a token, 405 on writes, the `key` filter, 404 for the job and host endpoints before their stages);
logs API (filters, `truncated`, `{"raw":…}`); retention in batches, the last 50 runs per job kept.

**Acceptance:**
1. [stand] `docker stop` → the dead-man receiver stops getting pings; [owner] a real healthchecks.io alert within ≈ 7 min.
2. [test] The sweep tick fails → the loop is stale → a `POST …/fail` request (respx).
3. [test] Telegram unavailable > 10 min → `/fail`.
4. [test] A process that exits 60 s after every start never pings the dead-man.
5. [stand] Restore drill: snapshot → a new container in SAFE_MODE → the fake Telegram and dead-man journals are empty; the
   configuration and incidents are in place.
6. [stand] A read token reads the status and logs; without a token — 401; writes — 405.
7. [owner] The first real deployment per the owner's deployment plan: admin over VPN ("connection refused from outside"),
   real external checks.

---

## S4 — Jobs and backups

**Read:** §3.4, §3.5, §4.1–4.3, §5.2, §5.4, §6.3–6.6, §8, §11.1–11.3, §12.1–12.7, §13.2, §14.4, §15.1, §15.5, §16.7, §17.4.

**Checkpoints:**
- **S4.1 — server:** `domain/schedule`; `schemas/jobs`; `evaluators/jobs`; `catalog/jobs`, `catalog/tokens` (job); `ingest/auth`,
  `ingest/jobs`, `ingest.record_reject`; the start/finish routes; ingest 503 in SAFE_MODE; `catalog/expected` and
  `loops/sweep` extended with jobs (§6.6, §13.2: without this a job that stops running never becomes missed);
  `GET /api/v1/jobs/{id}/runs`; UI Jobs (§12.5).
- **S4.2 — wrappers:** `wrappers/smon-job`, `wrappers/smon-job.ps1` (usage text with the `pipefail` note), `tests/wrappers`,
  `tests/wrappers/windows`; the `windows` CI job (`Planned: S4.2`).

**Mandatory tests:** `next_slot` strictly greater, `min_gap`/`max_gap`; table tests of `evaluate_job` (anchor, early and late
start, a manual run, finish-only, a long run, an open run disables missed, **three starts without a finish → overrun**,
enabling, a schedule change, interval, overrun, failed and warning with `warning_as`, stale with the default `data_time`,
texts); ingest (processing order §11.2, codes, `last_reject`, duplicate, the first result wins, finish without `run_id`,
`extra="forbid"`); the wrappers keep the exit code (bash on ubuntu; Windows PowerShell 5.1 with `-File`, negative codes, a
command that cannot start), the token is not in argv, retries only on network/5xx/429, timestamps with 6 fractional digits.

**Acceptance** (stand, `start_grace_s = 120`):
1. [stand] A wrong field → 422, the reason is shown on the job page.
2. [test] `exit 1` → critical; cron stopped → missed; `sleep` longer than `max_runtime_s` → overrun; the wrapper killed mid-run → overrun.
3. [test] Success with an old `data_time` → stale.
4. [test] A schedule change → no retroactive alarm.
5. [stand] The monitor is unavailable → the job finishes with its own exit code, then the monitor records missed.
6. [owner] Pilot: 1–2 real backup jobs per the owner's deployment plan (a wrapper or a post-job hook calling the API).

---

## S5 — Linux + Docker

**Read:** §2.3, §3.4, §4.1–4.3, §5.4, §6.1–6.6, §9, §11.1–11.3, §12.1–12.7, §13.9, §14.4, §15.1, §16.7, §17.1–17.2.

**Checkpoints:**
- **S5.1 — server:** `schemas/report_v1`; `evaluators/hosts`; `catalog/hosts` (+ expectations, discovery); `catalog/tokens`
  (host); `ingest/reports`; `catalog/expected` extended with hosts and expectations (§6.6); `GET /api/v1/hosts/{id}`;
  UI Hosts.
- **S5.2 — collector:** `collectors/linux/*`, `tests/collectors` (+ `ruff.toml` with `target-version = "py38"`); the release
  archive `smon-collectors-X.Y.Z.tar.gz` and `SHA256SUMS`, the `VERSION` constant in the scripts (SPEC §16.7); the Python 3.8
  CI job (`Planned: S5.2`).
- **S5.3 — export and import:** `schemas/export_v1`, `catalog/export` for all sections (settings, checks, jobs, hosts with
  expectations, tokens); UI `/settings/export`, `/settings/import` (§13.9).

**Mandatory tests:** report validation (limits, uniqueness keys, RFC 3339, `schema_version`, `errors[]`); `evaluate_report`
for every kind and every section rule (present, absent, empty, in `errors[]`, `errors[]` precedence), texts (golden);
discovery (defaults); suppression of children while the heartbeat is down; collector — parser unit tests on fixtures
(`/proc`, mountinfo, `systemctl show`, Docker API) under Python 3.8, `py_compile` 3.8, ruff py38; export → import → export
byte-identical; import into a non-empty instance → 409; an unknown field → 422; restored tokens authenticate.

**Acceptance:**
1. [owner] The collector installed on a test VM → the host appears, discovery offers expectations.
2. [test] Fixture reports: a disk past the threshold → warning, then critical → freed → resolved. [owner] The same with
   `fallocate` on the test VM.
3. [test] Fixture reports: an expected container stopped or unhealthy → critical after the second report. [owner] The same on
   the test VM.
4. [test] An expected disk unmounted → critical "missing"; a section disappears → unknown, then "no data" (≈ 45 min).
5. [test] Reports stop → exactly one alert (heartbeat), children suppressed. [owner] The collector timer stopped on the test VM.
6. [test] A host comes back with a critical disk → the message arrives.
7. [test] No access to the Docker socket → unknown, a "no data" warning 30 min later.
8. [stand] Export → import into an empty instance → a second export is identical.
9. [owner] Pilot: 1–2 real Linux hosts per the owner's deployment plan.

---

## S6 — Windows + Hyper-V

**Read:** §2.3, §9, §11.2, §16.7, §17.1, §17.3.

**Checkpoints:**
- **S6.1 — probe:** `collectors/windows/probe.ps1` — a read-only discovery on every target host (the PowerShell version: older
  Windows Server releases may ship 4.0, WMF 5.1 is required; TLS 1.2 in .NET; the Hyper-V and Storage modules). The owner runs
  it and reports the results without host names; a finding that changes the collector becomes a D record with status
  `proposed`. After S6.1 is merged the orchestrator records the blocking question `Q-S6-1` (run `probe.ps1` and report);
  S6.2 starts after the answer.
- **S6.2 — collector:** `collectors/windows/*`; PSScriptAnalyzer for the collector in the `windows` CI job (`Planned: S6.2`).

**Mandatory tests:** PSScriptAnalyzer clean (the `windows` job); one Windows-shaped fixture report (`os_family=windows`,
services, vms, storage, commit memory) passes the server validation. The collector's mapping has no automated tests (D105): it
is verified by the owner in the acceptance below.

**Acceptance:**
1. [owner] On a Hyper-V host: an expected running VM stopped → critical.
2. [owner] On a Windows VM: an expected service stopped → critical.
3. [test] A fixture report with a section in `errors[]` → unknown with a text. [owner] A section without permissions on a real host.
4. [owner] Pilot: a Hyper-V host and 2–3 Windows VMs per the owner's deployment plan.

---

## S7 — Rollout and v1.0

Mostly calendar time spent observing; little development.

**Work:** the full catalogue per the owner's deployment plan (owner); threshold tuning over 1–2 weeks; `docs/operations.md`
(created here; including the dead-man setup: period 1 min, grace 5 min), `docs/upgrade.md`, updating `docs/restore.md`; a
security review — a separate reviewer session using the checklist of §15 and §14.5; tag `v1.0.0`.

**Acceptance:**
1. [owner] All enabled objects are green or have accepted reasons.
2. [owner] At least one full weekly cycle of jobs has passed.
3. [owner] The restore drill has been repeated on the real instance.

---

## Backlog after v1.0

In priority order: alerts on container restart and OOM growth (§1.3); metrics history and charts (24 h / 7 d / 30 d); OS
update monitoring (a summary-only kind; the report accepts new sections without a `schema_version` bump); a dedicated collector
for a backup product's session API; Dependabot; localization; network device collectors; Linux mdadm in storage; TOTP; explicit
dependencies between checks; canary checks; threshold hysteresis; a private CA and STARTTLS for checks; OpenAPI on :8081;
pushing events to external agents; application-specific probes; multiple admin accounts; a guard against two hosts reporting
under one `host_id` (a cloned VM).
