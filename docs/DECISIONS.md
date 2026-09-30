# Decision log

Format of a new entry: `## D<n>. Title` (e.g. `## D123. …`, no dash) → date and stage, context, decision, reason, affected
SPEC sections, **status** (`proposed` / `accepted <date>` / `rejected`). The number is the next one after the highest ID on
`main`. Entries are never deleted; a replaced decision is marked "superseded by D<n>". Details of every accepted decision are
part of `docs/SPEC.md`. Only the owner sets `accepted` or `rejected` (`AGENTS.md` §2).

All decisions D1–D106 below were **accepted on 2026-09-29** unless marked as superseded.

## Initial decisions

Made by the owner while preparing the specification, after a requirements interview and independent reviews.

| # | Decision | Reason |
|---|---|---|
| D1 | Own application instead of assembling existing open-source monitors | One model "signal + deadline", one UI, jobs and backups with the required semantics |
| D2 | Primary alerts — an own Telegram bot through the Bot API directly. External AI agents are not on the critical path | A previous attempt depended on changes to a third-party agent acting as a relay and was never finished |
| D3 | The deliverable is a self-contained Docker image with a runtime contract (SPEC §16). Deployment is outside the repository | Independence from the hosting platform |
| D4 | All configuration through the web UI, stored in the database. The repository is universal, without configuration or deployment details | Configuration is not mixed with code; image upgrades do not touch settings |
| D5 | MVP: external checks, jobs and backups, Linux/Docker and Windows/Hyper-V collectors (vendor-specific backup integration removed by D72) | Covers the main risks of a small infrastructure |
| D6 | External agents get data through a read-only JSON API with a token (pull) | No delivery logic toward agents and no edge cases around it |
| D7 | Backups: start/finish (success/warning/failed) + an optional `data_time` against `max_data_age`. A multi-component backup is several jobs | A simple protocol; freshness by data time, not by report time |
| D8 | The admin UI only over a VPN and with a password, a single admin. TOTP later | A small attack surface; a trusted admin → no SSRF filtering of targets |
| D9 | Ingest (collectors, jobs) — public HTTPS with tokens on a separate port | No VPN needed on every host |
| D10 | Built-in SQLite snapshots with rotation + configuration export/import. Off-host copies are a deployment duty | The configuration lives in the database and must survive the loss of the host |
| D11 | Simple charts (24 h / 7 d / 30 d) — **moved after v1.0 by D73** | — |
| D12 | GitHub + GitHub Actions → image in GHCR (clarified by D48: public repository and image) | Independence from the monitored infrastructure |
| D13 | Python 3.12: FastAPI + SQLite + Jinja/HTMX, one process | Simplicity and readability for the owner and agents |
| D14 | Critical and warning — immediately, plus recovery; reminders for unacknowledged critical every 4 h; a daily summary at 09:00 | The daily summary is a safeguard against indefinite silence and a liveness check of the channel |
| D15 | Dead-man — an external healthchecks.io-compatible service | A watcher of the watcher outside this server |
| D16 | Ack / Mute 1h buttons and the `/status` command in Telegram — in the MVP | Reaction from a phone without the VPN |
| D17 | Own logs: JSON files rotated by a total size limit set in the UI (default 100 MiB); access through the UI and the read API | Debugging by humans and agents |
| D18 | The product is named `smon` and is not tied to any infrastructure; the repository contains no names of organizations or their hosts | Reuse in other projects |

## Review norms

- **Review 1** (simplicity and reliability) closed three paths to a silent false OK: a deadline for every signal including host
  sub-signals; a loop supervisor and loop liveness in health and the dead-man; level-triggered notifications. It also simplified
  the model: no outbox, canary checks, import with merge or encryption of secrets in the database.
- **Review 2** (detail) fixed the code layout and layers, core signatures, the database schema, keys, KIND_POLICY, the engine
  and sender pseudocode, message templates, collector report v1, the settings registry, API, UI and conventions.
- **Review 3** (per SPEC section) — D25–D71. **Review 4** (the whole documentation) — D72–D88. **The final review**
  (consistency, simplicity, process) — D89–D102; its second round — D103. **The multi-aspect review** (architecture,
  security, agent readiness) — D104–D106.

## Decisions during preparation (questions of the first implementer)

## D19. The `cli` layer is above `main`
- **Context:** by SPEC §3.2 `cli.main()` calls `main.serve()`, while §3.3 listed `smon.main | smon.cli` as siblings, forbidding imports between them.
- **Decision:** `smon.cli` is a separate top layer, `smon.main` is below it. **Affected:** SPEC §3.3.

## D20. Loop liveness is published in `RuntimeState`
- **Context:** problems like `loop sweep stale 75s` need the loop age, and `web` and `deadman` must not know about `loops`.
- **Decision:** `Supervisor(specs, clock, runtime)` writes `runtime.loop_last_ok[name] = clock.monotonic()` after every successful
  tick and fills `runtime.loop_stale_after` at startup; `health.evaluate(...)` computes stale loops, their age and problems
  (signature and location refined by D25); there is no `Supervisor.stale()`.
- **Reason:** one point of truth in `AppContext`; layers stay intact; the monotonic clock rules out false staleness on clock jumps.
- **Affected:** SPEC §3.2, §4, §13.1, §13.3.

## D21. Routing errors: 404 and 405
- **Decision:** an unknown path → 404 `not_found`, a wrong method → 405 `method_not_allowed` with `Allow`; on :8080 and for
  `/api/*` on :8081 — the uniform JSON body; other :8081 paths — an HTML page in the layout. **Affected:** SPEC §11.1.

## D22. Login-form CSRF — a double-submit cookie — **superseded by D89**
- **Context:** before login there is no session, while the SPEC requires CSRF for every non-GET on :8081.
- **Decision:** cookie `smon_login_csrf` (random; HttpOnly, SameSite=Strict, Secure per `SMON_ADMIN_HTTPS`; 1 h) + a hidden
  form field with the same value; `hmac.compare_digest`. After login `sessions.csrf_token` is used. **Affected:** SPEC §15.

## D23. Process lifecycle events
- **Decision:** `process.started`, `process.stopping`, `process.stopped` are in the event catalogue. **Affected:** SPEC §14.

## D24. The `alembic check` procedure — **superseded by D84**

## Review 3 (per SPEC section) — D25–D61

| # | Decision | Affected |
|---|---|---|
| D25 | `health.evaluate(runtime, safe_mode, now, now_mono)` — a pure function in `domain/health.py` (the epoch is needed for `undelivered_since`); there is no `ops/health.py`. Refines D20 | §3, §4, §13.3 |
| D26 | Layers reordered: `notify \| ingest \| probes` above `ops \| queries`; `settings \| audit` above `db`; `constants` is a separate bottom layer | §3.3 |
| D27 | `CheckConfig`/`CheckParams` live in `schemas/checks.py`; new modules `domain/schedule.py`, `domain/netaddr.py`, `domain/text.py`, `loops/probe_scheduler.py`, `loops/sweep.py` (`metrics.py` removed by D73) | §3.2 |
| D28 | `RuntimeState`: `tg_send_error` (sender) and `tg_poll_error` (poller) instead of one field; `probe_now`, `snapshot_retry_at`, `snapshot_last_error`; an owner table of the fields | §3.4, §4.1 |
| D29 | "Check now" through `runtime.probe_now`; the result goes the normal way with debounce; no synchronous probe in web | §7.5, §12.7 |
| D30 | The Telegram client is built outside a transaction from a `TelegramSettings` snapshot (the cache per `(token, api_base)` is superseded by D91); a nested `Database.tx()` raises `RuntimeError` | §4.5, §5.1 |
| D31 | Token verification — `catalog.tokens.authenticate`, shared by ingest and the read API; `last_used_at` is written by catalog (signature refined by D87) | §3.4, §11 |
| D32 | Audit is written by the module that owns the change; `audit.write()` also emits the `audit` log event; no separate log events for config changes, logins, acks or silences; a fixed `action` vocabulary | §3.4, §14.4, §15.5 |
| D33 | All tables are created by migration `0002` in S1; parent tables are never recreated in later stages | §5.3, ROADMAP |
| D34 | Migrations run with foreign keys off, in one transaction, with `foreign_key_check` before commit; a schema test against `create_all()` (the mechanism refined by D87) | §5.3 |
| D35 | The snapshot runs on a separate autocommit connection outside `tx()` (VACUUM INTO cannot run in a transaction); a space check; a UTC stamp; a retry at most once an hour (weekly rotation removed by D82) | §13.5 |
| D36 | Log protection against secrets: root = WARNING, `logs.level` only for `smon`, `httpx`/`httpcore` = WARNING, `TelegramError` and dead-man errors without URLs, token redaction in the formatter, `hide_parameters=True`, `env.py` without `fileConfig`; a closed field list | §14 |
| D37 | Deadlines are shifted by the monitor's downtime (`down_from` from `internal.sweep_at`) instead of a grace from the process start — frequent restarts do not postpone deadlines forever (the no-data timer has no shift — D78) | §6.6 |
| D38 | Sweep order `reconcile → apply → open_no_data`; the engine writes fields only when they differ; an incident keeps the last bad reason; invariants I1–I5 | §6.5, §13.2 |
| D39 | Jobs: `anchor = max(last_begin_at + grace, last_finished_at, enabled_at, schedule_changed_at)`; an open run disables missed (its deadline is overrun); start also calls `evaluate_job`; normative texts | §8.4 |
| D40 | Backup-software session handling — **superseded by D72** | — |
| D41 | `bad_n = 2` for service, container and vm (transient states during reboots and deployments); `hosts.stale_after_s ≥ 600` | §6.3, §5.4 |
| D42 | Hosts: an empty storage list with a storage expectation → critical; a section in `errors[]` takes precedence over its data; `sent_at` is set just before the POST and not checked for the future; normative texts (the OS updates part moved after v1.0 by D73) | §9 |
| D43 | **Owner:** "resolved" and "removed from monitoring" of an announced incident are not suppressed by silences or parents | §10.1 |
| D44 | The end of a mute/silence for an announced, open, unacknowledged incident → one reminder (any severity) | §10.1 |
| D45 | A due daily summary is a sender candidate and counts toward the backlog; a broken channel produces `/fail` in a quiet period too | §10.2 |
| D46 | All messages are split by `split_parts` with UTF-16 length; after a failed part the rest is not sent; 1 s between parts; every update advances the offset | §10 |
| D47 | **Owner:** the configuration export includes the hashes of active tokens | §13.9 |
| D48 | **Owner:** the repository and the image are public; tags `X.Y.Z` and `sha-<7>` without `latest`; amd64 + arm64 | §16.1 |
| D49 | **Owner:** an allowlist of the admin port, `SMON_ADMIN_ALLOW` | §15.2, §16.3 |
| D50 | Unknown fields: `extra="forbid"` for job events, check params and import; ignored only in the collector report | §1.2, §7.1, §8.2 |
| D51 | `proxy_headers=False` + `domain.netaddr.client_ip` (the rightmost untrusted address of XFF) (the ingest IP bucket removed by D80) | §11.2, §15.4 |
| D52 | Sessions: the database stores the sha256 of the id; `set-password` deletes all sessions (the sliding expiry is superseded by D89) | §5.4, §15.2 |
| D53 | CSP without inline code; confirmation through `data-confirm`; CSRF for htmx only through `hx-headers`; root-relative URLs | §12.1, §15.2 |
| D54 | Pagination only for growing lists (incidents, runs, audit, status_history); the `key=` filter with `silence_matches` semantics; OpenAPI disabled | §11 |
| D55 | The image HEALTHCHECK is a liveness check (any `/healthz` answer); full health — the dead-man and the UI | §16.1 |
| D56 | Strict env validation (an unknown `SMON_*` → exit 2); fail fast on `/data` permissions; only `EXPOSE 8080`; an image smoke test in CI | §16 |
| D57 | Collectors keep to the schema themselves (a duplicate key or a limit → the section goes to `errors[]`); `systemctl show` in one call; the `smon-job` token is never in argv; one config per job (runspaces and OS update sources superseded by D81 and D73) | §9.2, §17 |
| D58 | The host report limit is 512 KiB | §11.2 |
| D59 | Settings: a single write path `settings.set_many` with section validation and type coercion (used by the UI and import); ranges and formats of every key; a "Takes effect" column (signature refined by D87) | §2.3 |
| D60 | Hook `ctx.on_settings_changed(keys)`: logs → apply; Telegram → reset backoff and errors (+ `tg_offset` on a new token); dead-man → reset its state | §2.3, §3.2 |
| D61 | Files and query results larger than 1 MiB are never read fully into memory (a streaming snapshot gzip) | §2.2, §13.5 |

## Development process — D62–D71

| # | Decision | Affected |
|---|---|---|
| D62 | **Owner:** an orchestrator agent delegates every checkpoint to an implementer sub-agent in its own branch/worktree, gets a review from a separate reviewer sub-agent after a green CI and merges the PR itself (squash). One implementer at a time | AGENTS §1, ROADMAP |
| D63 | **Owner:** the repository history starts from the finished documentation; commits use a GitHub noreply address; the documentation contains no names or contacts of people | AGENTS §3, §7 |
| D64 | **Owner:** MIT license (copyright holder "smon contributors") | LICENSE |
| D65 | **Owner:** when the SPEC does not match the real behaviour of a library, the agent records a D record with status `proposed` and edits the SPEC right away; the owner accepts or rejects it at stage acceptance (mechanics refined by D86) | AGENTS §2 |
| D66 | Checkpoints inside stages; a checkpoint = a PR with a green CI; acceptance and a tag once per stage; CI on push and PR with an image smoke test | ROADMAP |
| D67 | S0 creates stubs of all layer modules (`.importlinter` is written once, `import-linter>=2`), the SAFE_MODE core and the generic `/settings` | ROADMAP S0 |
| D68 | Property tests of the invariants with `hypothesis` (a dev dependency); direct time outside `clock.py` is banned by ruff `TID251` | AGENTS §5, §6 |
| D69 | Export by stage — **superseded by D92** | ROADMAP, §13.9 |
| D70 | Getting CI green by weakening checks is forbidden (noqa/type: ignore without a reason, skip/xfail, linter configuration, golden adjustments, editing a tagged migration) — only through a D record | AGENTS §4 |
| D71 | Acceptance labels: [test] (an integration test on `FakeClock`), [stand] (accelerated parameters), [owner] | ROADMAP |

## Review 4 (the whole documentation) — D72–D88

| # | Decision | Affected |
|---|---|---|
| D72 | **Owner:** no vendor-specific backup integration. A job is any task that reports `start`/`finish` through the generic API; smon decides whether it started and finished on time, the result and the data freshness. Calls are embedded where needed (wrappers, post-job hooks). A dedicated collector for backup software is a backlog item. Removed: the backup-session report section and its ingest module, the job fields `source`, `host_id` and the backup product's job name, host-time clamping, host-deletion restrictions. Supersedes D40 | §1, §5.4, §8, §9, §11 |
| D73 | **Owner:** metrics history and charts (D11) and OS update monitoring are moved after v1.0. Removed from v1.0: `series`/`samples`, `metrics.py`, the `updates` kind and section, `thresholds.updates_max_age_days`, the Windows update task | §1.3, §2, §5, §6, §9, §17, ROADMAP |
| D74 | **Owner:** drift is limited to one stage: stage n+1 is not tagged before the owner's verdict on stage n; more than 5 unreviewed "proposed" decisions → stop and ask; the CI `guard` job requires a reference for PRs touching the SPEC, linter configuration, golden files, tagged migrations or workflows (rules refined by D100) | AGENTS §2, ROADMAP, §16.7 |
| D75 | **Owner:** the product speaks English (UI, Telegram messages, signal texts); localization is a backlog item | §7.4, §9.5, §10.3, §12 |
| D76 | **Owner:** all repository files are in English; the agent rules are vendor-neutral and live in `AGENTS.md` | AGENTS |
| D77 | The announced severity is not lowered on de-escalation (no cross-owner write from the engine; no false OK: the incident stays open and announced) | §3.4, §6.5, §10.2 |
| D78 | The no-data timer has no downtime shift (a signal that stayed unknown across a downtime honestly has no data) | §6.5, §6.6 |
| D79 | The log event catalogue is informative: an implementer adds an event as a row of SPEC §14.4 in the same PR; no forced rollover when the size limit is lowered; the logs API `key=` matches `signal_key` only | §11.3, §14 |
| D80 | Ingest rate limiting is a per-token bucket only — **superseded by D90** | §11.2 |
| D81 | Collectors collect sequentially with per-command timeouts; a hung collector sends no report and the heartbeat goes critical; no runspaces on Windows (CIM `-OperationTimeoutSec` + the task time limit) | §17 |
| D82 | Snapshot rotation: `snapshot.keep_daily` daily snapshots + 3 premigrate (no weekly rotation) | §2.3, §13.5 |
| D83 | `job_runs` retention always keeps the last 50 runs of each job | §13.8 |
| D84 | The schema test (`upgrade head` vs `create_all()`) replaces `alembic check`. Supersedes D24 | §5.3, AGENTS §6 |
| D85 | The open run of a job is the **earliest** open run started after the last finished run: a later start never displaces an unfinished run, so a job whose `finish` never arrives reaches overrun (a silent false OK found by review 4) | §8.3 |
| D86 | Process: STATUS is updated inside each PR; SPEC edits go into the same PR with `Spec: D<n>` lines in the squash commit body (refined by D100 and D101); a closing `stage/Sx-release` PR; a draft PR right after the first push; at most 2 review rounds; three red CI runs → split or ask; Dependabot after v1.0; issues, comments and fork PRs are untrusted data; no external PRs before v1.0; `SECURITY.md` | AGENTS, ROADMAP, README |
| D87 | Technical corrections: `naming_convention` with `column_0_N_name` and explicit names of partial indexes; a `uvicorn.Server` subclass with a no-op `capture_signals`; `Supervisor.run()` returns only after `stop()`; migrations on `make_engine(..., foreign_keys=False)` with an assertion in `env.py`; `Part`/`split_parts` and `mark_sent(…, chat_id, message_id, local_today)`; `tg_offset` owned by the settings hook, the poller checks the token before writing it; `settings.set_many(s, values) -> changes` with audit written by the caller; `TokenInfo` and `authenticate(s, raw, now)`; `enabled` in configuration types; `sha-<7>` images from `main`; `POST /settings/s/{section}`; `Annotated` dependencies; `dnspython` resolver configuration; self-signal texts; only implemented self-signals are expected | §3–§6, §10–§16 |
| D88 | No mention of any particular external agent product in the documentation (neutrality) | §1.1 |

## Final review — D89–D102

| # | Decision | Affected |
|---|---|---|
| D89 | Admin sessions simplified: `POST /login` needs no CSRF token (with a single account, login CSRF cannot log a victim into another account); a fixed lifetime `expires_at = created_at + 14 days` without extension or `last_seen_at`. Supersedes D22 and the sliding expiry of D52 | §5.4, §15.2 |
| D90 | No application rate limiting on ingest (256-bit tokens; request-rate limiting belongs to the reverse proxy); 429 comes only from the login limiter. Supersedes D80 | §11.1, §11.2 |
| D91 | No Telegram client cache: `ctx.telegram_client(cfg)` is an async context manager; the sender opens a client only when it has candidates, the poller one per long poll, the test message one per call. Refines D30 | §2.3, §4.5, §10.2, §10.5 |
| D92 | Configuration export and import are implemented once, for all sections, in checkpoint S5.3 (format v1). Supersedes D69 | §12.5, §13.9, ROADMAP |
| D93 | The read API is reduced to `status`, `incidents`, `signals`, `logs` (S3), `jobs/{id}/runs` (S4) and `hosts/{id}` (S5); an endpoint returns 404 before its stage; object statuses are read through `/signals` | §11.3 |
| D94 | Shutdown and signals: no `conn.interrupt()` and no retention stop flag; a snapshot or retention in progress is ended by Docker's SIGKILL, which is safe; `signal.signal` sets a flag during startup, `loop.add_signal_handler` takes over right before `serve()` | §13.4, §13.5, §13.8, §16.4 |
| D95 | Audit: `target` is `<kind>:<id>`; one record per incident; details are the names of changed fields or keys (no old/new values), the duration for a mute | §15.5 |
| D96 | Determinism: blocking work goes only through `ctx.run_blocking` (a synchronous call in integration tests); `FakeClock` yields `FAKE_YIELDS = 50` times after every step; `ctx.disk_usage`; self-signal messages contain no time or age, so a repeated sweep writes nothing; normative OK texts of self-signals | §3.5, §4.4, §13.2 |
| D97 | The dead-man pings only after 120 s of process uptime, so a crash-looping monitor never pings (a silent false OK found by the final review); the uptime is shown in the Overview, the daily summary and `/api/v1/status` | §2.2, §10.4, §11.3, §12.5, §13.3 |
| D98 | A failed premigrate snapshot → no migration, `process.failed`, exit 1 (a migration is irreversible); Alembic's `Config()` is built in code (`script_location="smon.db:migrations"`), `alembic.ini` is for developers only | §5.3, §16.4 |
| D99 | Timestamps from collectors and wrappers have at most 6 fractional digits; the wrappers document `bash -o pipefail` for pipelines (a pipeline reports its last command's code — a silent false OK); accepted coverage boundaries: intermittent failures and a crash-looping container that is `running` at report time | §1.3, §9.1, §17 |
| D100 | CI `guard`: a separate workflow on `pull_request` (including `edited`, so a PR body edit re-runs only this check); looks only at modified or deleted protected files; passes with a `Spec: D<n>` or `Planned: Sx.n` line in the PR body; D records are written `D<n>` without a dash; a PR template; a `windows` CI job from S4.2 (real Windows PowerShell 5.1, PSScriptAnalyzer); `push` runs only on `main`; the collector and Windows suites are excluded from the default pytest run; required jobs are never skipped by `if:`; release images are tagged by semver only. Refines D74 and D86 | §16.1, §16.7, AGENTS §4, ROADMAP |
| D101 | Process: the owner channel (a blocking question or readiness for acceptance → a docs PR to `main` plus the session's final message; the owner answers in a session or with their own commit; agents never write an owner verdict or set `accepted`); the implementer updates STATUS before `gh pr ready`; `gh pr merge --admin` is forbidden; only CI runs after `gh pr ready` count toward the red-CI limit; an open `stage/*` PR is unfinished work to continue; after 2 review rounds the orchestrator fixes the finding itself or asks; the next checkpoint starts only after the previous one is merged; the acceptance record lists the SPEC changes of the stage; a stage is tagged right after its closing PR is merged once the owner verdict on the previous stage is recorded; restart recovery from remote `stage/*` branches and `Review round` lines in the PR body; oversized checkpoints are split into `Sx.n.m`. Refines D86 | AGENTS §1, §2, §8, ROADMAP |
| D102 | S0 is split into three checkpoints (core; process; web, Docker and release) and S1 into four (core; probes and the Telegram client; UI; notifications); every [test] acceptance scenario is written in the checkpoint that completes its modules; the read list of every stage is completed; S5 and S6 scenarios that need VMs are [owner] with [test] duplicates on fixture reports; [stand] uses an image built locally from the accepted commit; reminders are implemented in S1.4, the poller and the daily summary in S2 | ROADMAP |

## Final review, round 2 — D103

| # | Decision | Affected |
|---|---|---|
| D103 | Contract corrections: sender and poller `run_once` take an `open_client` callable and `process_started_at` (the client is opened only when there are candidates; the summary shows the uptime); the maintenance tick owns `snapshot_retry_at` / `snapshot_last_error` and `take()` raises; retention writes `internal.retention_date` before deleting; `ctx.disk_usage` is synchronous and called before the sweep transaction; `tg_send_error` texts for network errors and a missing client; the poller pauses on any other `TelegramError`; `enabled_at = now` on every re-enable; `mute_incidents` sets `reason = "mute {duration}"`; the job ok text for a first run still in progress; `key=*` in the SQL filter; the audit writers and settings callers corrected | §3.4, §3.5, §4.2, §4.3, §4.5, §5.2, §8.4, §10.2, §10.5, §11.1, §11.3, §13.2, §13.5, §13.8 |

## Multi-aspect review — D104–D106

| # | Decision | Affected |
|---|---|---|
| D104 | **Owner:** UI cosmetics are guidance, not requirements: the navigation order and the settings submenu, macro names and colors, the composition of detail pages and the list columns. Routes, forms, confirmations, PRG, banners, state display, htmx and CSP stay normative. A deviation from guidance is a STATUS note, never a blocking finding or a D record | §12, AGENTS §1 |
| D105 | **Owner:** the Windows collector has no automated tests of its logic: PSScriptAnalyzer in CI and verification on real hosts by the owner in the S6 acceptance | §17.3, ROADMAP S6 |
| D106 | Security and process corrections: `guard` runs from the base branch and also watches `tool.pytest`, dependency lists, every `ruff.toml` / `pytest.ini`, `devtools/guard.py` and `.github/`; `Planned: Sx` for stages without checkpoints; actions pinned by commit SHA, `persist-credentials: false`, no `${{ github.event.* }}` in `run:`, `uv` image by digest; per-job permissions in `release.yml`; the collector archive and the `VERSION` constant; agents never change repository settings; a failed [test]/[stand] scenario gets a fix checkpoint; owner verdict values; a merged release PR without its tag is tagged on recovery; `process_started_at` is taken after migrations; `{N}` in deadline texts is the configured window; login redirects without `next`; the import size limit | §6.6, §10.4, §11.1, §12.7, §12.8, §15.5, §16.1, §16.4, §16.7, AGENTS, ROADMAP, STATUS |
