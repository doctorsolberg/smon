# smon — Specification

- Version: 2.2.
- Status: normative document for development. Everything stated here is a requirement, not a wish, except text explicitly
  marked as guidance (§12).
- Reading order for an agent: `AGENTS.md` → `docs/STATUS.md` → the sections of this document listed for the current stage
  in `docs/ROADMAP.md`.
- A choice not covered by this document that affects the schema, a contract, a signal key, a message text or a module
  boundary is recorded in `docs/DECISIONS.md` (see `AGENTS.md` §2).

Contents:
1. Purpose and principles
2. Requirements
3. Components, code layout, layers, write ownership
4. Core contracts
5. Data model
6. Signals: keys, kind policy, semantics, engine
7. Checks and probes
8. Jobs
9. Hosts and collector report v1
10. Notifications: sender, templates, daily summary, poller
11. API
12. UI
13. Reliability: loops, health, dead-man, snapshots, SAFE_MODE, restore, retention, export/import
14. Logging
15. Security
16. Runtime contract
17. Collectors and wrappers

---

## 1. Purpose and principles

### 1.1 Purpose

`smon` (Simple Monitor) is a small self-contained monitoring service for a small infrastructure run by a single operator:

- active checks of network services: HTTP(S), TCP, TLS, DNS;
- monitoring of scheduled jobs and backups: missed run, failure, overrun, stale data. Jobs report `start`/`finish`
  through a generic HTTP API; the calls are embedded wherever needed (wrapper scripts, post-job hooks of any tool);
- telemetry from lightweight Linux/Docker and Windows/Hyper-V collectors (push);
- incidents and Telegram notifications, a daily summary, an external dead-man;
- an administrator web UI and a read-only API for external agents and integrations.

The main goal is to notice in time that a service, a backup or the monitor itself has stopped working.
It is not an observability platform.

The product is not tied to any particular infrastructure; the repository is public. The repository contains no names of
organizations, their domains or hosts, and no names or contacts of people; examples use placeholders (`example.com`,
`host-1`, `backup-job`).

### 1.2 Principles

1. **Everything is a signal with a deadline.** Every object is reduced to status streams, and every signal has a freshness
   deadline:
   - for checks, host heartbeats and host expectations: `window_s` (§6.6);
   - for jobs: the schedule (missed, overrun, stale; §8.4);
   - self-signals are recomputed by every sweep, and the liveness of the sweep itself is watched by the dead-man (§13.3).

   After the deadline, silence is never "OK": the signal becomes `unknown` (for a heartbeat: `heartbeat_severity`), and 1800 s
   later a "no data" incident is opened (§6.4). Before the first deadline the signal is "awaiting data"; a job stays ok
   while none of its reasons is active.
2. **Fail loud, not smart.** Invalid data is rejected (4xx) and logged. If the request is authenticated with the object's
   token, the reason is shown on the object's page (§11.2); a 401 goes to the log only. Since no valid data arrives,
   the deadline fires. Strange data is not interpreted.
3. **Only a silent false OK is unacceptable.** A silent false OK is a real problem that, for longer than the §2.2 targets,
   is shown as ok or is reported by none of: an immediate message, the daily summary, the dead-man. Not a silent false
   OK: `unknown` in the UI before the no-data incident fires; summary-only kinds (§6.3); suppression by a parent or a silence
   (visible in the daily summary). A rare false alarm and a duplicate message are acceptable (at-least-once delivery).
4. **One place for incident logic.** Evaluators are pure functions "data → status". Debounce, incidents, suppression and
   notifications live only in the engine and the sender.
5. **Edge-case policy.** A new edge case is first checked against two questions: does it reduce to "no fresh data → alert"
   or "rejected → visible"? If yes, no special code is written.
   - Special handling is allowed only if the case causes a silent false OK or systematic false alarms. Systematic means
     alarms that occur during normal operation of the object without any failure and repeat on every run or cycle.
     A one-off coincidence is not systematic.
   - Special handling requires a D<n> record in `docs/DECISIONS.md` (the scenario; why it does not reduce to the two
     questions; which false OK or alarm it causes) and a test reproducing the scenario. Otherwise it becomes a backlog line.
6. **The monitor never acts on the infrastructure.** No commands on hosts, no remediation, no remote access.

**Precedence:** if a detailed section of this document contradicts a principle, code follows the detailed section, and the
contradiction is recorded in `docs/STATUS.md` under "Questions" with a proposed D<n>.

### 1.3 Out of scope

- Commands on hosts, remediation, SSH/WinRM, code execution from the UI.
- Filtering check targets by address (SSRF): there is a single trusted administrator working over a VPN; an allowlist of
  the admin port is the second line of defence (§15). The risk is accepted deliberately.
- Exactly-once delivery, HA, multiple admin accounts (the Telegram buttons can be pressed by everyone in
  `allowed_user_ids`, §10.5), log collection from hosts, APM, Prometheus/Grafana.
- Vendor-specific backup integrations (e.g. reading backup-software sessions): jobs report through the generic API;
  a dedicated collector is a backlog item.
- Verifying backup restorability or content: status and `data_time` are enough.
- Matching events to schedule slots with per-run precision.
- Own DST handling: `cronsim` and `zoneinfo` semantics as is; a false "missed" on the night of a transition is acceptable (§1.2 p.3).
- Canary checks and detection of the monitor's own network loss: the dead-man fires. After recovery, batches of "opened"
  and "resolved" arrive; jobs whose events were lost show false missed/overrun until their next run. This is an accepted
  cost; no special code.
- Alerts on container restart count growth and OOM kills — display only. Accepted boundaries: a container in a crash loop that
  is `running` at report time stays green; an intermittent failure that never yields `bad_n` consecutive bad results (e.g. a
  load balancer with one dead backend) stays green. Do not add special code for them (backlog: restart growth alerts).
- After v1.0 (backlog): metrics history and charts; OS update monitoring; localization (UI and messages are in English).
- Previously considered and rejected (do not "improve"):
  - a message queue or outbox (§10.1);
  - import with merge (§13.9);
  - encryption of secrets in the database (§15);
  - retries and spooling in collectors (§17.2);
  - pushing events to external agents (D6);
  - configuring objects through files or env (D4, §2.3);
  - TLS termination in the application (§16);
  - off-host copying of snapshots by the application (D10);
  - dependencies between checks (backlog).

---

## 2. Requirements

### 2.1 Functional

**External checks** (§7): `http`, `tcp`, `tls`, `dns`. Defaults: interval 300 s; while a status change is being confirmed
(either direction) — retry after 60 s; timeout 10 s for the whole probe. Alarm after 3 consecutive failures, recovery after
2 successes — fixed (KIND_POLICY). Latency is stored. "Check now" from the UI.

**Jobs and backups** (§8):
- schedule — cron + IANA time zone, or an interval;
- `start`/`finish` events from wrappers, scripts or post-job hooks of any tool;
- job status — the worst of missed, overrun, failed/warning, stale.

**Hosts** (§9): a thin collector sends facts; thresholds are configured on the server. Expected disks, services, containers
and VMs are picked in the UI from the last report (discovery) or added manually.

**Incidents and notifications** (§6, §10):
- one open incident per signal;
- ack stops reminders; escalation clears the ack;
- a silence has a mandatory end; observation continues; only new alarms and reminders are suppressed — "resolved" for
  an announced incident is sent immediately; when a mute/silence ends, one reminder is sent if the problem persists;
- Telegram: messages on open, escalation, recovery and removal from monitoring; critical reminders; daily summary;
  `[Ack] [Mute 1h]` buttons; `/status` command;
- the daily summary is a mandatory safeguard against indefinite silence.

**Self-monitoring** (§13): self-signals `self:telegram`, `self:snapshot`, `self:disk`, `self:deadman`; `/healthz`; dead-man.

**Admin UI** (§12) — available only over a VPN (e.g. Tailscale), password-protected, with an address allowlist:
overview; CRUD of checks, jobs and hosts; incidents, silences, logs, settings, tokens, audit; configuration export and import.

**Integrations** (§11): a read-only API with a `read`-scoped token; the dead-man through a healthchecks.io-compatible URL.

**Own reliability** (§13): loop supervisor, health and dead-man; SQLite snapshots with rotation; configuration export/import
(including token hashes); `SMON_SAFE_MODE`; retention.

**Logging** (§14): JSON files with rotation by a total size limit configured in the UI; viewing in the UI and through the API.

### 2.2 Non-functional

| Requirement | Target |
|---|---|
| Capacity | 20 hosts, 50 checks, 30 jobs; RAM < 256 MiB; one process. Files and query results larger than 1 MiB are never read fully into memory (the snapshot is gzipped as a stream; exception: the log tail ≤ 20 MiB) |
| Web outage detection | with default check parameters and no semaphore queue: `interval + 2·retry + 3·(timeout + 5) + 5 ≈ 470 s` (≤ 8 min) |
| Host loss detection | `stale_after_s` since the last report (≤ 15 min by default) + ≤ 15 s sweep + ≤ 5 s sender |
| Item loss detection | missing from a received section — next report (service, container, VM — second report, ≤ 10 min; memory and clock — third); no data for a section — `unknown` after `stale_after_s`, the "no data" notification 30 min later (≈ 45 min) |
| Monitor crash or hang detection | via the dead-man (period 1 min, grace 5 min; the first ping only after 120 s of uptime, so a crash loop never looks alive): crash or full hang of the process ≈ 7 min; stale probe/sweep/sender/poller loops — 60–120 s + ≤ 60 s until `/fail`; **exception** — maintenance (stale after 2 h) |
| Broken Telegram channel detection | ≤ 10 min after the first failed send; in a quiet period — no later than `summary_time + 10 min` |
| Data intake | 202 only after the database commit; a database failure → 503 on ingest; unhealthy — through the failed mandatory sweep write (≤ 75 s) |
| Time | all deadlines use the server receive time. Timestamps in payloads — RFC 3339 with an offset only; naive → 422. `data_time` more than 5 min in the future → 422 |
| Retention (defaults) | status history, runs (last 50 per job are always kept), closed incidents, audit, silences — 365 days |

### 2.3 Settings (`settings` registry)

Everything editable in the UI is stored in the `settings` table (§5). The registry is `Setting(key, type, default, secret,
validate)` in `smon/settings.py`; `section` is the key prefix; keys with `not secret and section != "internal"` are exported.
Cross-field rules live in `SECTION_RULES` (threshold pairs).

- **Single write path** `settings.set_many(s, values) -> dict[str, tuple[old, new]]`: validates the effective state of
  the whole section, coerces values to their types (float → float, list[int] → sorted unique), writes only changed keys,
  raises a validation error (→ 422 with a message at the field) otherwise, and returns the changes. Callers (the settings
  route, import) write the audit record themselves and, after commit, call `ctx.on_settings_changed(changes.keys())`.
  `internal.*` keys are written with `settings.set(s, key, value)` (no section validation, no audit);
  `settings.init_internal(s, now)` does `INSERT OR IGNORE` of `internal.instance_created_at`.
- A key outside the registry on write or import → 422; on read from the database (after an upgrade) it is ignored with a
  WARNING log event `settings.ignored`.
- Values are read from the database on every tick; there is no cache.
- **Hook `ctx.on_settings_changed(keys)`** (called by web after commit of a settings save and after an import):
  - `logs.*` → `apply_log_settings`;
  - `telegram.*` → `runtime.sender_backoff_until = 0`, `sender_attempts = 0`, `tg_send_error = tg_poll_error = None`,
    `poll_failing_since = None` (`undelivered_since` is not reset: only a real delivery clears it);
    a new `telegram.bot_token` also resets `internal.tg_offset` (own short transaction);
  - `deadman.ping_url` → `deadman_last_ok_at = deadman_last_error = None`.
- Secrets are write-only in the UI ("set / not set" + a "clear" checkbox); they never appear in the UI, the API, logs
  or exports.

| Key | Type and validation | Default | Purpose | Takes effect |
|---|---|---|---|---|
| `general.timezone` | IANA (`ZoneInfo(v)` without an exception) | `"UTC"` | time display, daily summary, snapshot and retention schedule, `display_tz` of interval jobs | next render and tick; a summary, snapshot or retention run may shift or repeat once (acceptable); job time zones are not affected |
| `general.instance_name` | str 1..40, no control characters | `"smon"` | title of the daily summary and `/status` | immediately |
| `telegram.bot_token` | str, **secret** | `""` | bot token | next sender and poller tick (a running long poll lives ≤ 25 s; the poller's token check drops its result); hook `on_settings_changed` |
| `telegram.chat_id` | str `^-?\d+$` | `""` | notification chat | next tick; already announced incidents are not re-announced in the new chat (they stay visible in the summary and reminders) |
| `telegram.thread_id` | int > 0 \| null | `null` | `message_thread_id` (forum topic) | next send |
| `telegram.allowed_user_ids` | list[int] (comma-separated input, each `^\d+$`, ≤ 20); an empty list — UI warns "buttons and /status will not work" | `[]` | who may press buttons and call `/status` | next update |
| `notify.reminder_interval_h` | int 1..24 | `4` | reminders for unacknowledged critical incidents | next sender tick |
| `notify.summary_time` | `^([01]\d\|2[0-3]):[0-5]\d$` | `"09:00"` | daily summary time (in `general.timezone`) | rule "`local_now ≥ time` and not yet sent today": if already sent today — tomorrow; if the new time has passed — immediately |
| `deadman.ping_url` | `^https?://`, ≤ 500, trailing `/` stripped; **secret** | `""` | healthchecks.io-compatible check URL (period 1 min, grace 5 min) | next deadman tick; save time = `deadman_url_set_at`; hook `on_settings_changed` |
| `logs.level` | `DEBUG\|INFO\|WARNING` | `"INFO"` | level of the `smon` logger | immediately (`apply_log_settings`) |
| `logs.max_total_mb` | int 10..2000 (MiB) | `100` | total size limit of log files | immediately (`apply_log_settings`, §14.1) |
| `retention.history_days` | int 30..3650 | `365` | status_history, closed incidents, job_runs, audit_log, silences | next retention run |
| `snapshot.time` | `^([01]\d\|2[0-3]):[0-5]\d$` | `"03:30"` | daily snapshot time | same rule as the summary |
| `snapshot.keep_daily` | int 1..60 | `7` | rotation | rotation after the next daily snapshot |
| `thresholds.disk_warn_free_pct` / `disk_crit_free_pct` | float, `1 ≤ crit < warn ≤ 99` | `15` / `8` | default disk thresholds (percent, not fractions) | next host report; not recomputed retroactively |
| `thresholds.mem_warn_avail_pct` / `mem_crit_avail_pct` | float, `1 ≤ crit < warn ≤ 99` | `10` / `5` | default memory thresholds | next report (with debounce — third) |
| `thresholds.commit_warn_pct` | float 50..100 | `90` | Windows commit charge | next report |
| `thresholds.clock_warn_skew_s` | int 60..3600 | `300` | clock skew | next report |

Threshold overrides in expectations are validated against the effective pair `override ?? global` (§9.4); when global
thresholds change, overrides are not re-validated: an inverted pair cannot cause a false OK because critical is checked first.

`internal.*` keys are not shown in the UI and not exported; their writers — §3.4:

| Key | Type | Default |
|---|---|---|
| `internal.tg_offset` | int | `0` |
| `internal.summary_sent_date` | str `YYYY-MM-DD` (in `general.timezone`) \| null | null |
| `internal.summary_sent_at` | int \| null | null |
| `internal.sweep_at` | int \| null (also `down_from` at startup, §6.6) | null |
| `internal.retention_date` | str \| null | null |
| `internal.instance_created_at` | int | written by `settings.init_internal` |

The boundary "UI or env": env is only the runtime contract (§16.3). Fixed numbers (loop periods, self-signal thresholds,
limits, timeouts) live in `smon/constants.py`, not in settings.

---

## 3. Components, code layout, layers, write ownership

### 3.1 Data flow

```text
 collectors / job wrappers ──HTTPS──► :8080 ingest ─┐
 probe loop ─► http/tcp/tls/dns ────────────────────┤ evaluators (pure functions)
 sweep loop (15 s): deadlines, jobs, self ──────────┘        ▼
                                          SignalUpdate(key,status,message,title,value,origin)
                                                             ▼
                                  ENGINE (in the caller's transaction, BEGIN IMMEDIATE):
                                  debounce → signal → status_history → incident open/update/resolve
                                                             ▼
                          SQLite (WAL) ◄─────────────────────┤
 Telegram ◄── sender loop (5 s): "announced" vs "current"    ┘   + daily summary
 Telegram ──► poller loop: Ack / Mute 1h / /status
 dead-man URL ◄── deadman loop (60 s): ping | /fail
 maintenance loop (60 s): snapshots, retention
 VPN ──► :8081 admin UI + read API

 main(): see §16.4 — env → flock → migrations → AppContext → Supervisor(loops) + uvicorn (one Server, two sockets)
```

### 3.2 Package layout

```text
src/smon/
  __init__.py          __version__ (from package metadata)
  __main__.py          python -m smon → cli.main()
  cli.py               argparse: serve | admin set-password [--stdin] | db upgrade. No business logic
  main.py              serve(), upgrade(): startup order §16.4; signals; waiting for supervisor and server
  context.py           AppContext: config, db, clock, runtime, process_started_at, down_from;
                       telegram_client(cfg), run_blocking(fn, *args), disk_usage(path), on_settings_changed(keys) (§2.3)
  config.py            RuntimeConfig (frozen) from SMON_* with validation (§16.3); read once in main
  constants.py         ALL fixed numbers (periods, self thresholds, limits, timeouts) and IMPLEMENTED_SELF (§6.6)
  clock.py             Clock (Protocol), SystemClock
  log.py               JSON formatter with a closed field list, setup_logging(), apply_log_settings(), contextvars
  fmt.py               pure formatters and parsers: bytes, pct, duration, parse_duration, ts, ts_short
  settings.py          settings registry: get, get_all, set, set_many, init_internal, thresholds(s), telegram(s)
  audit.py             write(s, actor, action, target, details) → audit_log row + log event `audit`
  domain/              pure types and rules, no IO
    status.py          Status(StrEnum), rank(), is_bad(), worst()
    keys.py            key builders and parser, parent_key(), silence_matches(), key_filter()
    kinds.py           Kind(StrEnum), KindPolicy, KIND_POLICY
    titles.py          signal titles
    text.py            clip(s, n) — truncation with "…"; parse_expected_status(); normalize_dns()
    schedule.py        next_slot(), slots(), min_gap(), max_gap() — wrapper around cronsim (§8.1)
    health.py          evaluate(runtime, safe_mode, now, now_mono) → (healthy, problems)
    netaddr.py         client_ip(peer, xff, trusted), ip_allowed(ip, allow)
    types.py           §4.1 types
  schemas/             pydantic models of external contracts
    report_v1.py       collector report (§9)
    jobs.py            start/finish (§8)
    checks.py          CheckParams (union by type), CheckConfig (§7)
    api.py             read API responses (§11)
    errors.py          ApiError, error body
    export_v1.py       export format (§13.9)
  evaluators/          pure functions → SignalUpdate
    checks.py  hosts.py  jobs.py  deadlines.py  self.py
  db/
    base.py            make_engine(), Database: PRAGMA, BEGIN IMMEDIATE hook, tx(), nested-tx guard, naming_convention
    models.py          ALL ORM tables (the single source of the schema)
    migrations/        env.py (no fileConfig), versions/NNNN_<slug>.py
  engine/              the only place that changes signals, status_history, incidents, silences
    core.py            apply(), open_no_data(), reconcile(), signal_states()
    actions.py         ack(), mute_incidents(), create_silence(), cancel_silence()
  catalog/             configuration CRUD, loading into domain types, expected_signals(), export/import
    checks.py  hosts.py  jobs.py  tokens.py (create, revoke, authenticate)  expected.py  export.py
  ops/
    deadman.py  snapshots.py  retention.py  logs_reader.py
  queries.py           read-only queries shared by UI, read API and the daily summary (including suppression())
  ingest/              intake of reports and job events without HTTP
    auth.py  reports.py  jobs.py
  probes/              IO probes: CheckConfig → ProbeResult; never write to the DB, never raise
    __init__.py (run_probe, make_ssl_context)  http.py  tcp.py  tls.py  dns.py  texts.py (error texts §7.4)
  notify/
    telegram.py        TelegramClient (Protocol), HttpTelegramClient, TelegramError
    render.py          pure templates: Item, Part, SummaryData, render_*, split_parts()
    sender.py          select_candidates(), mark_sent(), run_once()
    poller.py          run_once()
    summary.py         collect() → SummaryData
  loops/
    supervisor.py      LoopSpec, Supervisor
    probe_scheduler.py probe scheduler (§7.5)
    sweep.py           sweep tick (§13.2)
    ticks.py           build_loops(ctx) → list[LoopSpec]; the remaining ticks
  web/
    dispatch.py        outer ASGI layer: request_id, X-Request-ID, request log, dispatch by scope["server"][1]
    ingest_app.py      FastAPI for :8080
    admin_app.py       FastAPI for :8081
    deps.py  errors.py  auth.py  csrf.py  forms.py  allowlist.py
    api/               read API
    ui/routes/*.py  ui/templates/  ui/static/
```

### 3.3 Layers

Imports go only downwards through the list. The rule is enforced by `import-linter` (layers contract) in CI. Modules on the
same line separated by `|` do not import each other.

```text
smon.cli
smon.main
smon.web | smon.loops
smon.context
smon.notify | smon.ingest | smon.probes
smon.ops | smon.queries
smon.catalog
smon.engine
smon.settings | smon.audit
smon.db
smon.evaluators
smon.schemas
smon.domain
smon.fmt | smon.clock | smon.config | smon.log
smon.constants
```

- Modules below `context` never import it: they receive explicit arguments (`db`, `Session`, `clock`, `TelegramClient`,
  `RuntimeState`, settings snapshots). `AppContext` is unpacked only by `loops/*`, `web` and `main`.
- `domain` may import `cronsim` and `zoneinfo` (pure libraries).

### 3.4 Write ownership

**Tables:**

| What | Written by |
|---|---|
| `signals`, `status_history`, `incidents` (except announce fields), `silences`, `incidents.acked_*` | `engine` only |
| `incidents.announced_severity`, `announced_at`, `resolve_announced`; `tg_messages` | `notify.sender` only |
| `checks`, `hosts` (configuration fields), `host_expectations`, `jobs` (configuration fields), `tokens` (including `last_used_at`) | `catalog` only |
| `job_runs`, `hosts.last_report*`, `hosts.os_family`, `*.last_reject*` | `ingest` only |
| `settings` | `smon.settings` only (callers: web, `main.upgrade`, `catalog.export`, sender, poller, sweep, `ops.retention`, the settings hook) |
| `audit_log` | `smon.audit` only |
| `admin`, `sessions` | `web.auth` and `cli` (`set-password`) |
| deletion by retention | `ops.retention` only (FK cascades do not count as a violation) |

**`internal.*` keys:**

| Key | Written by |
|---|---|
| `internal.tg_offset` | poller; reset by `context.on_settings_changed` when `telegram.bot_token` changes |
| `internal.summary_sent_date`, `internal.summary_sent_at` | sender |
| `internal.sweep_at` | sweep |
| `internal.retention_date` | retention |
| `internal.instance_created_at` | `settings.init_internal` (called by `main.upgrade()` after migrations) |

**`RuntimeState` fields (in memory):**

| Field | Written by |
|---|---|
| `loop_last_ok`, `loop_stale_after` | supervisor |
| `undelivered_since`, `pending_count` | sender only |
| `sender_backoff_until`, `sender_attempts`, `tg_send_error` | sender; reset by `context.on_settings_changed` |
| `tg_poll_error`, `poll_failing_since` | poller; reset by `context.on_settings_changed` |
| `deadman_last_ok_at`, `deadman_last_error` | deadman; reset by `context.on_settings_changed` |
| `probe_now` | web adds, the probe scheduler takes |
| `snapshot_retry_at`, `snapshot_last_error` | the maintenance tick (`loops/ticks.py`) |

**Audit** is written by the module that owns the change, passing `actor`: `catalog.*`, `engine.actions`, `web.auth`,
the settings, import, export and Telegram-test routes (`settings.update`, `config.import`, `config.export`, `telegram.test`),
`cli` (`admin.password_set`). The `action` vocabulary — §15.5.

### 3.5 Code rules

1. **Pure modules** (`domain`, `evaluators`, `notify.render`, `fmt`) do no IO, do not read clocks and do not log.
   They accept frozen dataclasses or pydantic models, **never ORM objects**.
2. **DI.** A single `AppContext` is created in `main`. It is passed to the apps via `app.state.ctx` and to loops as an
   argument. There are no module-level singletons except loggers and contextvars (`request_id`, `loop`, `log_fields`).
   In-memory state that belongs to an app (the login limiter's failed-login counters, §15.2) lives in that app's `app.state`.
3. **No repository layer.** Queries live in the owning module; shared read queries live in `queries.py`.
4. **HTTP routes are thin:** parse → `with ctx.db.tx() as s:` → call `ingest`, `catalog`, `engine`, `settings`, `queries`,
   `ops` or `notify` → respond. An ingest route uses up to three short transactions: authentication (tx1) → body read
   (await) → intake (tx2); a rejection after tx1 is recorded by `ingest.record_reject` in its own short transaction.
   Dependencies use `Annotated[T, Depends(...)]` (ruff B008).
5. **Async only at the boundaries:** `run_probe`, `TelegramClient` methods, sender/poller `run_once`, the deadman tick,
   `Clock.sleep`, `Supervisor.run`, HTTP handlers. Everything that accepts a `Session` is synchronous.
6. **Blocking work** (snapshot, retention, log reading) goes only through `ctx.run_blocking(fn, *args)` (`asyncio.to_thread`
   in production, a synchronous call in `tests/integration`), so that integration tests on `FakeClock` are deterministic.
   `ctx.disk_usage(path)` is synchronous (one `statvfs` of the local `/data`; a fixed value in tests).

---

## 4. Core contracts

The signatures are normative. Changes go through `docs/DECISIONS.md`. All dataclasses are `@dataclass(frozen=True, slots=True)`
except `RuntimeState`; time is `int` epoch seconds UTC unless stated otherwise.

### 4.1 Domain types

```python
# domain/status.py
class Status(StrEnum):
    OK = "ok"; WARNING = "warning"; CRITICAL = "critical"; UNKNOWN = "unknown"
def rank(s: Status) -> int          # ok=0, unknown=1, warning=2, critical=3
def is_bad(s: Status) -> bool        # warning | critical
def worst(*s: Status) -> Status      # by rank; no arguments → OK

# domain/kinds.py
class Kind(StrEnum):   # check, job, heartbeat, disk, memory, service, container, vm, storage, clock,
                       # self_telegram, self_snapshot, self_disk, self_deadman
class KindPolicy: bad_n: int; ok_n: int; notify: Literal["immediate", "summary"]; has_parent: bool
KIND_POLICY: Mapping[Kind, KindPolicy]          # table §6.3

# domain/keys.py
class KeyParts(NamedTuple): object_key: str; kind: Kind; subject: str
def parse_key(key: str) -> KeyParts             # invalid key → ValueError
def check_key(id) / job_key(id) / host_key(id) / host_sub_key(host_id, kind, subject="") / self_key(name) -> str
def parent_key(key: str) -> str | None
def silence_matches(target: str, key: str) -> bool
def key_filter(k: str, key: str) -> bool        # = silence_matches; the `key=` filter in UI and API (§11.1)

# domain/types.py
Origin = Literal["data", "sweep"]   # data: data arrived (ingest, probe); sweep: computed from time

class SignalUpdate:
    key: str
    status: Status       # OBSERVED status; the engine decides the effective one
    message: str         # English; sweep updates contain no current time or age; the engine clips it to 300
    title: str           # from domain.titles; the engine clips it to 120
    value: float | None = None   # unit fixed per kind: check=latency_ms, disk=free_pct,
                                 # memory=available_pct, clock=skew_s
    origin: Origin = "data"

class ExpectedSignal:
    key: str
    title: str
    window_s: int | None = None      # None — the deadline is computed by evaluate_job or the sweep (job, self)
    base_at: int = 0                 # what to count from while there is no data (§6.6)
    deadline_status: Status = Status.UNKNOWN
    deadline_message: str = ""

class SignalState: key: str; status: Status; since: int; last_update_at: int | None

class ProbeResult:
    ok: bool
    error: str | None           # text from the catalogue §7.4
    latency_ms: float | None
    http_status: int | None = None
    tls_days_left: int | None = None     # -1 — the certificate has expired
    tls_not_after: int | None = None
    dns_answers: tuple[str, ...] = ()

class HostConfig: id: str; name: str; enabled: bool; enabled_at: int; stale_after_s: int; heartbeat_severity: Status
class Expectation: id: int; kind: Kind; subject: str; expected_state: str | None;
                   warn_pct: float | None; crit_pct: float | None; reserve_bytes: int | None; created_at: int
class Thresholds: disk_warn_free_pct: float; disk_crit_free_pct: float; mem_warn_avail_pct: float;
                  mem_crit_avail_pct: float; commit_warn_pct: float; clock_warn_skew_s: int

class JobConfig:
    id: str; name: str; enabled: bool; schedule_kind: Literal["cron", "interval"]
    cron: str | None; tz: str | None; interval_s: int | None
    display_tz: str              # tz for cron, general.timezone for interval (filled in by catalog)
    enabled_at: int; schedule_changed_at: int
    start_grace_s: int; max_runtime_s: int; max_data_age_s: int | None
    warning_as: Literal["warning", "ok"]
class RunInfo: status: Literal["success", "warning", "failed"]; finished_at: int; exit_code: int | None; message: str | None
class JobRunsView:                # all fields optional; None terms are skipped in max()
    last_begin_at: int | None     # max(coalesce(started_at, finished_at))
    last_finished_at: int | None  # max(finished_at)
    open_run_started_at: int | None   # §8.3
    last_finished: RunInfo | None     # run with the largest (finished_at, id)
    last_good_data_time: int | None   # max(data_time) over success|warning
    last_reject_at: int | None; last_reject_reason: str | None

class SelfState:
    process_started_at: int; instance_created_at: int
    undelivered_since: int | None; tg_send_error: str | None; tg_poll_error: str | None
    newest_snapshot_at: int | None; snapshot_last_error: str | None
    data_free_bytes: int | None; data_total_bytes: int | None     # None — statvfs failed → self:disk unknown
    deadman_configured: bool; deadman_url_set_at: int | None
    deadman_last_ok_at: int | None; deadman_last_error: str | None

class TelegramSettings: bot_token: str; chat_id: str; thread_id: int | None; allowed_user_ids: frozenset[int]
class TokenInfo: id: int; kind: Literal["host", "job", "read"]; host_id: str | None; job_id: str | None; prefix: str

@dataclass
class RuntimeState:                       # in memory, mutable, no IO; field owners — §3.4
    undelivered_since: int | None = None
    pending_count: int = 0
    tg_send_error: str | None = None      # "{status} {description}"; reset on a successful send
    tg_poll_error: str | None = None      # 401/409 or network errors > 5 min; reset on a successful getUpdates
    poll_failing_since: int | None = None # start of the current series of poller network errors
    deadman_last_ok_at: int | None = None
    deadman_last_error: str | None = None
    sender_backoff_until: int = 0
    sender_attempts: int = 0
    loop_last_ok: dict[str, float] = field(default_factory=dict)      # clock.monotonic() of the last successful tick
    loop_stale_after: dict[str, float] = field(default_factory=dict)
    probe_now: set[str] = field(default_factory=set)
    snapshot_retry_at: int = 0
    snapshot_last_error: str | None = None

# domain/health.py
def evaluate(runtime: RuntimeState, safe_mode: bool, now: int, now_mono: float) -> tuple[bool, list[str]]
```

`CheckConfig` and `CheckParams` live in `schemas/checks.py` (frozen pydantic, §7.1):
`CheckConfig(id, name, type, params, enabled, enabled_at, interval_s, retry_interval_s, timeout_s, failure_severity, updated_at)`.

### 4.2 Evaluators and engine

```python
# evaluators/*: no IO, no clocks, deterministic
def evaluate_check(check: CheckConfig, r: ProbeResult, now: int) -> SignalUpdate
def evaluate_report(host: HostConfig, exp: Sequence[Expectation], thr: Thresholds,
                    report: ReportV1, received_at: int) -> list[SignalUpdate]   # heartbeat first
def evaluate_job(job: JobConfig, runs: JobRunsView, now: int, origin: Origin) -> SignalUpdate
def next_expected_at(job: JobConfig, runs: JobRunsView) -> int | None           # slot of the missed rule (§8.4)
def evaluate_deadlines(expected: Mapping[str, ExpectedSignal], signals: Mapping[str, SignalState],
                       now: int, process_started_at: int, down_from: int) -> list[SignalUpdate]  # origin="sweep"; §6.6
def evaluate_self(st: SelfState, now: int) -> list[SignalUpdate]                 # origin="sweep"; only IMPLEMENTED_SELF and configured

# engine/core.py: never commit; the caller owns the transaction
@dataclass
class EngineResult:
    status: dict[str, Status]   # effective status after apply for every input key
    opened: list[int]; escalated: list[int]; resolved: list[int]

def apply(s: Session, updates: Sequence[SignalUpdate], now: int) -> EngineResult
def open_no_data(s: Session, now: int) -> list[int]
def reconcile(s: Session, expected_keys: Collection[str], now: int) -> list[int]
def signal_states(s: Session) -> dict[str, SignalState]

# engine/actions.py (actor: "admin" | "tg:<user_id>" | "system" | "cli"); they write audit themselves
def ack(s, incident_ids, actor, now) -> int              # open ones only; a repeated ack does not change acked_*; → number of open among ids
def mute_incidents(s, incident_ids, duration_s, actor, now) -> int   # a silence per signal_key of each open incident; → number of silences
    # reason = "mute {duration}" (e.g. "mute 1h"); one incident.mute audit record per incident, no separate silence.create
def create_silence(s, target, ends_at, reason, actor, now) -> int    # created_at=now; ends_at<=now or an invalid target → ValueError
def cancel_silence(s, silence_id, actor, now) -> None                # cancelling a cancelled silence is a no-op
```

### 4.3 Cross-module signatures

```python
catalog.expected.expected_signals(s, now) -> dict[str, ExpectedSignal]         # self-signals from constants.IMPLEMENTED_SELF
catalog.checks.enabled(s) -> list[CheckConfig];  catalog.checks.get(s, id) -> CheckConfig | None
catalog.jobs.enabled(s) -> list[JobConfig];      catalog.jobs.get(s, id) -> JobConfig | None
catalog.jobs.runs_view(s, job_id) -> JobRunsView
catalog.hosts.get(s, id) -> tuple[HostConfig, list[Expectation]] | None
catalog.tokens.authenticate(s, raw, now) -> TokenInfo | None    # lookup by sha256; revoked → None; last_used_at at most once a minute
ingest.record_reject(s, obj_kind, obj_id, reason, now) -> None  # own short transaction (§11.2)
settings.thresholds(s) -> Thresholds;  settings.telegram(s) -> TelegramSettings | None   # None if token or chat_id is empty
settings.set(s, key, value) -> None;   settings.set_many(s, values) -> dict[str, tuple[Any, Any]];  settings.init_internal(s, now) -> None
settings.updated_at(s, key) -> int | None
queries.suppression(s, now) -> Suppression       # .is_suppressed(key), .silenced_until(key), .parent_ok(key)
async probes.run_probe(check: CheckConfig, now: int, ssl_ctx: ssl.SSLContext | None = None) -> ProbeResult
notify.sender.select_candidates(s, now, reminder_s, summary_due: bool) -> list[Item]
notify.sender.mark_sent(s, items, now, *, chat_id: str, message_id: int, local_today: str) -> None
OpenClient = Callable[[TelegramSettings | None], AbstractAsyncContextManager[TelegramClient | None]]   # the tick passes ctx.telegram_client
async notify.sender.run_once(db, clock, open_client: OpenClient, runtime, health_fn, process_started_at: int) -> None
async notify.poller.run_once(db, clock, open_client: OpenClient, runtime, health_fn, process_started_at: int) -> None
notify.summary.collect(s, now, tz, healthy, problems, runtime, deadman_configured, process_started_at) -> SummaryData
ops.snapshots.take(db_path, backups_dir, kind, now) -> Path;  # raises on failure  ops.snapshots.newest_at(backups_dir) -> int | None
ops.snapshots.rotate(backups_dir, keep_daily, tz) -> None
ops.retention.run(db, now, tz, history_days) -> dict[str, int]
async ops.deadman.ping(client, url, healthy, problems) -> str | None          # None on 2xx, otherwise the error text (no URL)
ops.logs_reader.read(logs_dir, level, key, contains, since, limit) -> tuple[list[dict], bool]
```

**Notification types** (`notify/render.py`, pure):

```python
ItemKind = Literal["opened", "escalated", "reminder", "resolved", "removed", "summary"]
class Item: kind: ItemKind; incident_id: int | None; signal_key: str; title: str; message: str
            severity_sent: Status | None; old_severity: Status | None; opened_at: int | None; resolved_at: int | None
            # summary: incident_id=None and empty fields; at most one per tick
class Part: text: str; block_idx: tuple[int, ...]
class IncidentLine: id: int; severity: Status; title: str; message: str; opened_at: int; resolved_at: int | None
                    acked_by: str | None; silenced_until: int | None; parent_ok: bool | None
class SignalLine: key: str; title: str; status: Status; message: str; since: int; parent_ok: bool | None
class SummaryData:
    instance_name: str; healthy: bool; problems: tuple[str, ...]; uptime_s: int
    deadman_configured: bool; deadman_last_ok_at: int | None; pending_count: int
    counts: Mapping[Status, int]; open_incidents: tuple[IncidentLine, ...]; closed_unannounced: tuple[IncidentLine, ...]
    silences: tuple[tuple[str, int, str], ...]      # (target, ends_at, reason)
    disabled: tuple[str, ...]; no_data: tuple[SignalLine, ...]; summary_only: tuple[SignalLine, ...]
    awaiting: tuple[str, ...]; last_sweep_at: int | None; version: str
def render_block(item: Item, now: int, tz: str) -> str
def render_summary(d: SummaryData, now: int, tz: str) -> list[str]      # blocks
def render_status(d: SummaryData, now: int, tz: str) -> list[str]       # blocks
def render_test(instance_name: str, now: int, tz: str) -> str
def split_parts(blocks: Sequence[str], limit: int = 4000) -> list[Part]
```

`health_fn: Callable[[], tuple[bool, list[str]]]` is passed by the tick; it calls `domain.health.evaluate` with its `ctx`.
`run_once` reads `cfg`, `general.timezone` and `deadman_configured` in its own transaction. The sender enters
`open_client(cfg)` only if `select_candidates` returned items; the poller enters it once per long poll. The tick passes
`ctx.telegram_client` and `ctx.process_started_at` (needed by `summary.collect` for the uptime line).

### 4.4 Clock

```python
class Clock(Protocol):
    def now(self) -> int: ...                  # epoch seconds UTC
    def monotonic(self) -> float: ...
    async def sleep(self, seconds: float) -> None: ...
```

`tests/fakes.py: FakeClock(start=1767571200)` (2026-01-05T00:00:00Z, a Monday) — manual virtual time:
- `sleep(s)` awaits a future resolved when `now + s` is reached (`s <= 0` → `asyncio.sleep(0)`);
- `advance(s)` moves both time sources and wakes everyone whose deadline has come;
- `async run_for(s)` advances time through the nearest sleeper deadlines; after every step it yields control
  `FAKE_YIELDS = 50` times (`await asyncio.sleep(0)`) so that woken tasks run to their next `await`;
- `now()` = int of the internal float.
- Integration tests use `ctx.run_blocking` as a synchronous call (§3.5), so no real threads run while virtual time jumps.

### 4.5 Telegram client

```python
class TelegramError(Exception):
    kind: Literal["network", "http"]
    status: int | None
    description: str          # description from the Bot API response or the exception class name; NEVER the URL or str(exc) of httpx
    retry_after: int | None

class TelegramClient(Protocol):
    async def send_message(self, chat_id: str, thread_id: int | None, text: str, buttons: bool) -> int  # message_id
    async def get_updates(self, offset: int, timeout_s: int) -> list[dict[str, Any]]
    async def answer_callback(self, callback_id: str, text: str) -> None
```

- `run_once` reads `settings.telegram(s)` in its own transaction; the client is created **outside** the transaction through
  `open_client` (§4.3): `ctx.telegram_client(cfg)` returns an async context manager yielding a `TelegramClient` (or None if `cfg is None` or in
  SAFE_MODE). **No client cache:** the sender opens a client only when it has candidates, the poller — one per long poll,
  the test message — one per call.
- Timeouts: `send_message`, `answer_callback` — 10 s; `get_updates` — `timeout_s + 10`.
- `HttpTelegramClient` never calls `raise_for_status()` and never includes the URL in errors (the URL contains the token).
- Buttons are fixed: `[Ack]` `callback_data="ack"`, `[Mute 1h]` `callback_data="mute1h"`.
- Unit tests use `FakeTelegram` (`tests/fakes.py`); compose uses `devtools/fake_telegram` (the same HTTP API).

### 4.6 Supervisor

```python
@dataclass(frozen=True)
class LoopSpec:
    name: str
    period_s: float
    stale_after_s: float | None     # None — not part of health
    tick: Callable[[], Awaitable[None]]
    on_stop: Callable[[], Awaitable[None]] | None = None   # called after the loop is cancelled (e.g. cancelling probe tasks)

class Supervisor:
    def __init__(self, specs: Sequence[LoopSpec], clock: Clock, runtime: RuntimeState) -> None: ...
    async def run(self) -> None: ...       # returns ONLY after stop(), even with an empty specs list
    def stop(self) -> None: ...
```

Loop liveness is published in `RuntimeState` (D20); there is no `stale()` method.

---

## 5. Data model

### 5.1 Transaction model (normative)

- A **synchronous** SQLAlchemy 2 `Session` is used. `AsyncSession` and aiosqlite are forbidden.
- All database work happens on the event loop thread. Every endpoint and `Depends` that touches the database is declared
  `async def` only: FastAPI runs synchronous `def` handlers in a threadpool, which would move database work to another thread.
- A transaction is opened only as `with ctx.db.tx() as s:` — `BEGIN IMMEDIATE`, commit on exit, rollback on exception.
  **A nested `tx()` on the same thread raises `RuntimeError`** (otherwise the second `BEGIN IMMEDIATE` waits for itself
  for `busy_timeout`).
- **No `await` inside `tx()`.** The order is always: first `await` (probe, body read, Telegram), then `tx`.
- No request-scoped sessions through `Depends`. Only snapshots (dataclasses, pydantic models) and scalars leave a transaction.
- On the loop thread — only short transactions and queries with LIMIT. The worst loop freeze is `busy_timeout` (5 s) while
  retention (through `ctx.run_blocking`) holds the write lock; therefore a retention batch is short, with a commit per batch.
- Only the snapshot, retention and log reading run off the loop, through `ctx.run_blocking` (§3.5). Retention uses `db.tx()` on
  its own connection; the snapshot uses a separate autocommit connection **outside** `tx()` (`VACUUM INTO` cannot run inside
  a transaction, §13.5); log reading does not use the database.
- `OperationalError` on ingest → 503 `unavailable`.

**`db/base.py` recipe (normative):**

```python
def make_engine(path: str, *, foreign_keys: bool = True) -> Engine:
    engine = create_engine(f"sqlite:///{path}", connect_args={"check_same_thread": False},
                           hide_parameters=True)        # SQL parameters never reach exception texts or logs
    @event.listens_for(engine, "connect")
    def _on_connect(dbapi_conn, _):
        dbapi_conn.isolation_level = None              # pysqlite does not open its own transactions
        cur = dbapi_conn.cursor()
        for p in ("journal_mode=WAL", "synchronous=FULL", f"foreign_keys={'ON' if foreign_keys else 'OFF'}",
                  "busy_timeout=5000"):
            cur.execute(f"PRAGMA {p}")                 # PRAGMA outside any transaction
        cur.close()
    @event.listens_for(engine, "begin")
    def _on_begin(conn):
        conn.exec_driver_sql("BEGIN IMMEDIATE")
    return engine
SessionFactory = sessionmaker(make_engine(path), expire_on_commit=False)
# Database.tx(): nested-call check → with SessionFactory.begin() as s: yield s
```

Test: two threads; the second `tx()` waits for the first instead of failing with `SQLITE_BUSY`.

### 5.2 Conventions

| What | Rule |
|---|---|
| Time | every `*_at`, `ts`, `since` — `INTEGER` epoch seconds UTC. Core code uses `int` too; `datetime` only at the boundaries: RFC 3339 parsing, cron, rendering |
| Units | durations `*_s` — INT; bytes `*_bytes` — INT; percentages `*_pct` — REAL |
| enum | `TEXT` + a **named** `CHECK`; `StrEnum` in Python. Do not use `sa.Enum`. A batch migration loses unnamed CHECKs |
| boolean | `sa.Boolean` |
| JSON | `sa.JSON`; nullable — `sa.JSON(none_as_null=True)`; written only as `model_dump(mode="json")`; changed only by assigning a new object |
| DEFAULT | `server_default` plus the same ORM default |
| Configuration IDs | `TEXT` slug `^[a-z0-9][a-z0-9-]{0,39}$`, entered on creation, immutable |
| Service-row IDs | `INTEGER PRIMARY KEY` |
| FK | configuration children — `ON DELETE CASCADE`. State tables (`signals`, `status_history`, `incidents`, `silences`) have no FK to configuration: they link by key text, history survives deletion |
| `updated_at` | set only by `catalog`, explicitly, without `onupdate`; ingest writes (`last_report`, `last_reject*`) do not change it. On creation `enabled_at = schedule_changed_at = updated_at = created_at`; `enabled_at = now` whenever `enabled` changes from false to true (checks, hosts, jobs) |
| naming | `MetaData(naming_convention={"ix": "ix_%(table_name)s_%(column_0_N_name)s", "uq": "uq_%(table_name)s_%(column_0_N_name)s", "ck": "ck_%(table_name)s_%(constraint_name)s", "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s", "pk": "pk_%(table_name)s"})`. Index names are global in SQLite; partial indexes get explicit names |

### 5.3 Migrations

- Alembic with `render_as_batch=True`; `env.py` **does not call `fileConfig`**, and `alembic.ini` has no logging sections
  (otherwise the `smon.*` loggers get disabled).
- **Migrations run on a separate engine `make_engine(path, foreign_keys=False)`** (FK off in the connect hook, i.e. before
  any transaction; a `PRAGMA foreign_keys` inside a transaction has no effect). `env.py` asserts `PRAGMA foreign_keys == 0`
  before `run_migrations`; all revisions run in one transaction (`transactional_ddl=True`); before commit
  `PRAGMA foreign_key_check` must return nothing, otherwise rollback and exit 1. Reason: batch mode recreates a table via
  DROP, and with FKs on SQLite would cascade-delete children.
- Batch rules: a column referenced by a CHECK may be dropped or renamed only after `drop_constraint`.
- File `NNNN_<slug>.py`, revision = `"NNNN"`. No downgrade. A migration included in a release tag is never edited.
- **Revisions:**
  - `0001` (S0) — `settings`, `admin`, `sessions`, `audit_log`;
  - `0002` (S1) — **all other tables of §5.4**. Stages S3–S6 add logic only; parent tables are never recreated.
- Migrations are applied by `main.upgrade()` (§16.4) — both in `serve` and in `smon db upgrade`. `main.upgrade()` builds the
  Alembic `Config()` in code (`script_location="smon.db:migrations"`); `alembic.ini` exists only for developers and is not in the
  image. A `premigrate` snapshot (from S3) is taken only if the current revision is not `None` and differs from head; **if the
  snapshot fails, nothing is migrated**: `process.failed`, stderr `premigrate snapshot failed: {error}`, exit 1 (a migration is
  irreversible; the resulting restart loop is loud). A revision unknown to the code →
  exit 1 `database schema NNNN is newer than smon X.Y.Z; restore a premigrate snapshot or use a newer image`.
- **Schema test** (`tests/db`, mandatory): a database after `upgrade head` is compared with a database from
  `metadata.create_all()` by `PRAGMA table_xinfo`, indexes (including `sqlite_master.sql`), `foreign_key_list` and CHECK texts.
  Another test: a batch migration of a parent table keeps the rows of its CASCADE children (synthetic tables in S0,
  real tables from S1).

### 5.4 Tables

Columns without a `NULL` mark are `NOT NULL`.

```text
-- 0001 (S0)
settings        key TEXT PK | value JSON | updated_at INT
admin           id INT PK CHECK(id=1) | password_hash TEXT | updated_at INT
sessions        id_hash TEXT PK (sha256 hex of the cookie value) | csrf_token TEXT | created_at INT | expires_at INT
                INDEX(expires_at)
audit_log       id INT PK | at INT | actor TEXT | action TEXT (§15.5) | target TEXT | details JSON NULL (no secrets)
                INDEX(at)

-- 0002 (S1)
checks          id TEXT PK | name TEXT (≤100) | type TEXT CHECK(http,tcp,tls,dns) | params JSON
                | interval_s INT DEFAULT 300 CHECK(>=30) | retry_interval_s INT DEFAULT 60 CHECK(>=10)
                | timeout_s INT DEFAULT 10 CHECK(1..60)
                | failure_severity TEXT DEFAULT 'critical' CHECK(warning,critical)
                | enabled BOOL | enabled_at INT | created_at INT | updated_at INT
hosts           id TEXT PK | name TEXT (≤100) | enabled BOOL | enabled_at INT | stale_after_s INT DEFAULT 900 CHECK(>=600)
                | heartbeat_severity TEXT DEFAULT 'critical' CHECK(warning,critical)
                | os_family TEXT NULL CHECK(linux,windows)
                | last_report_at INT NULL | last_report JSON NULL
                | last_reject_at INT NULL | last_reject_reason TEXT NULL
                | created_at INT | updated_at INT
host_expectations id INT PK | host_id TEXT FK→hosts ON DELETE CASCADE
                | kind TEXT CHECK(disk,memory,service,container,vm,storage,clock)
                | subject TEXT DEFAULT ''
                | expected_state TEXT NULL CHECK(running,off)
                | warn_pct REAL NULL | crit_pct REAL NULL        -- override for disk and memory
                | reserve_bytes INT NULL                          -- disk only
                | created_at INT | updated_at INT | UNIQUE(host_id,kind,subject)
                CHECK vm_state: (kind='vm') = (expected_state IS NOT NULL)
                CHECK singleton_subject: (kind IN ('memory','storage','clock')) = (subject='')
                CHECK pct_scope: (warn_pct IS NULL AND crit_pct IS NULL) OR kind IN ('disk','memory')
                CHECK reserve_scope: reserve_bytes IS NULL OR (kind='disk' AND reserve_bytes > 0)
jobs            id TEXT PK | name TEXT (≤100) | enabled BOOL | enabled_at INT
                | schedule_kind TEXT CHECK(cron,interval) | cron TEXT NULL | tz TEXT NULL | interval_s INT NULL CHECK(>=60)
                | schedule_changed_at INT
                | start_grace_s INT DEFAULT 1800 CHECK(>=60) | max_runtime_s INT DEFAULT 14400 CHECK(>0)
                | max_data_age_s INT NULL
                | warning_as TEXT DEFAULT 'warning' CHECK(warning,ok)
                | last_reject_at INT NULL | last_reject_reason TEXT NULL
                | created_at INT | updated_at INT
                CHECK schedule: (schedule_kind='cron' AND cron IS NOT NULL AND tz IS NOT NULL) OR
                                (schedule_kind='interval' AND interval_s IS NOT NULL)
job_runs        id INT PK | job_id TEXT FK→jobs ON DELETE CASCADE | run_id TEXT (≤64)
                | started_at INT NULL | finished_at INT NULL | status TEXT NULL CHECK(success,warning,failed)
                | exit_code INT NULL | data_time INT NULL | message TEXT NULL (≤1000) | created_at INT
                UNIQUE(job_id,run_id); INDEX(job_id,started_at); INDEX(job_id,finished_at); INDEX(job_id,data_time)
tokens          id INT PK | kind TEXT CHECK(host,job,read) | host_id TEXT NULL FK→hosts ON DELETE CASCADE
                | job_id TEXT NULL FK→jobs ON DELETE CASCADE | label TEXT
                | prefix TEXT | sha256 TEXT UNIQUE | created_at INT | last_used_at INT NULL | revoked_at INT NULL
                CHECK host_scope: (kind='host') = (host_id IS NOT NULL)
                CHECK job_scope:  (kind='job')  = (job_id IS NOT NULL)

signals         key TEXT PK (≤300) | object_key TEXT | kind TEXT CHECK(<all Kind values>) | subject TEXT DEFAULT ''
                | title TEXT | status TEXT CHECK(ok,warning,critical,unknown) | since INT | message TEXT
                | value REAL NULL | last_update_at INT NULL
                | streak_class TEXT NULL CHECK(ok,bad) | streak INT DEFAULT 0
                INDEX(object_key)
status_history  id INT PK | signal_key TEXT | at INT | old_status TEXT | new_status TEXT | message TEXT
                INDEX(signal_key,at); INDEX(at)
incidents       id INT PK | signal_key TEXT | title TEXT | severity TEXT CHECK(warning,critical) | message TEXT
                | no_data BOOL DEFAULT 0 | opened_at INT | resolved_at INT NULL
                | resolve_reason TEXT NULL CHECK(recovered,removed)
                | acked_at INT NULL | acked_by TEXT NULL
                | announced_severity TEXT NULL CHECK(warning,critical) | announced_at INT NULL
                | resolve_announced BOOL DEFAULT 0
                CHECK resolved: (resolved_at IS NULL) = (resolve_reason IS NULL)
                UNIQUE INDEX ix_incidents_open_signal_key (signal_key) WHERE resolved_at IS NULL
                INDEX(signal_key,opened_at); INDEX(resolved_at)
                INDEX ix_incidents_resolve_pending (resolved_at)
                      WHERE resolved_at IS NOT NULL AND announced_severity IS NOT NULL AND resolve_announced=0
silences        id INT PK | target TEXT | reason TEXT | ends_at INT CHECK(ends_at>created_at)
                | created_by TEXT | created_at INT | cancelled_at INT NULL
                INDEX(ends_at)                      -- a silence starts at created_at
tg_messages     chat_id TEXT | message_id INT | sent_at INT | incident_ids JSON | PK(chat_id,message_id)
```

- **Deleting a host** cascades to its tokens and expectations; its signals and incidents leave through `reconcile`
  (`removed`); `status_history` and silences remain.
- `tg_messages.chat_id` is a string of the same form as in callbacks (`str(message.chat.id)`, §10.5).

---

## 6. Signals: keys, kind policy, semantics, engine

### 6.1 Keys

Keys are built and parsed **only** by `domain/keys.py`.

```text
check:<id>                   kind=check
job:<id>                     kind=job
host:<id>                    kind=heartbeat     (heartbeat key = object key = parent of the sub-signals)
host:<id>:<kind>:<subject>   kind ∈ disk | service | container | vm
host:<id>:<kind>             kind ∈ memory | storage | clock   (subject = '')
self:<name>                  kind=self_<name>,  name ∈ telegram | snapshot | disk | deadman
```

- `object_key`: `check:<id>`, `job:<id>`, `host:<id>`, `self:<name>`.
- **The subject is not escaped.** Colons (`C:`), `/`, `$`, spaces and dots are allowed. Parsing is unambiguous: `host_id`
  is a slug without `:`, `kind` is a fixed word, the subject is everything after the third `:` (`host:web-1:disk:C:` → `C:`).
  Use `partition`, not `split`. The maximum key length is 256 < 300.
- Subject: 1..200 characters, no control characters. Case is preserved; comparison is exact.
- Subject normalization is done by the collector: containers — without the leading `/`; Linux services — the unit name
  without `.service`; Windows: service — `.Name`, disk — `DeviceID` (`C:`).
- **A silence target** is `*`, an object key or a full signal key successfully parsed by `parse_key`; anything else → 422
  (otherwise `host` would silence all hosts, and a typo would silence nothing).

```python
def silence_matches(target: str, key: str) -> bool:
    return target == "*" or key == target or key.startswith(target + ":")
```

The `+ ":"` rule keeps `host:web` from matching `host:web2`. The `key=` filter in the UI and API has the same semantics (`key_filter`).

### 6.2 Titles (`domain/titles.py`)

| Signal | Title |
|---|---|
| check | `{name} ({type})` |
| job | `job {name}` |
| heartbeat | `{host} · heartbeat` |
| disk | `{host} · disk {subject}` |
| memory | `{host} · memory` |
| service | `{host} · service {subject}` |
| container | `{host} · container {subject}` |
| vm | `{host} · VM {subject}` |
| storage, clock | `{host} · storage`, `{host} · clock` |
| self | `monitor · Telegram`, `monitor · snapshots`, `monitor · /data disk`, `monitor · dead-man` |

`{host}` and `{name}` are the object's `name` field (≤ 100). `titles` clips the result to 120 characters with "…".

### 6.3 Kind policy (`domain/kinds.py`, KIND_POLICY)

| kind | Bad statuses | bad_n | ok_n | Deadline (no data) | Parent | notify |
|---|---|---|---|---|---|---|
| check | `failure_severity`; tls: warning < 21 days, critical < 7 days or expired | 3 | 2 | no result for max(3×interval, 900 s) → unknown | — | immediate |
| job | critical: failed, missed, overrun, stale; warning: warning result | 1 | 1 | computed by `evaluate_job` | — | immediate |
| heartbeat | `heartbeat_severity` | 1 | 1 | no report for `stale_after_s` → `heartbeat_severity` | — | immediate |
| disk | warning / critical / critical "missing" | 1 | 1 | not refreshed for `stale_after_s` → unknown | host | immediate |
| memory | warning / critical | 3 | 3 | → unknown | host | immediate |
| service | critical | 2 | 1 | → unknown | host | immediate |
| container | critical | 2 | 1 | → unknown | host | immediate |
| vm | critical | 2 | 1 | → unknown | host | immediate |
| storage | critical / unknown | 1 | 1 | → unknown | host | immediate |
| clock | warning | 3 | 3 | → unknown | host | immediate |
| self_telegram | critical | 1 | 1 | — | — | **summary** |
| self_snapshot | critical | 1 | 1 | — | — | immediate |
| self_disk | warning / critical | 1 | 1 | — | — | immediate |
| self_deadman | warning | 1 | 1 | — | — | immediate |

- **Summary-only kinds never have incidents.** Their status is visible in the UI and in the daily summary (section
  "Summary-only", while `unknown` goes to "No data"); they are never sender candidates and never count toward the undelivered queue.
  Notifying about a broken Telegram through Telegram is pointless — it is covered by the banner and `/fail` (§13.3).
- CPU and restart count are not signals: display only.
- `bad_n = 2` for service, container and vm: transient states during a normal reboot or deployment (delayed start,
  container re-creation) would otherwise cause systematic false alarms (§1.2 p.5); the delay is +5 min.
- All host sub-signals except the heartbeat exist **only** as `host_expectations` rows (§9.4).

### 6.4 Status semantics

- **`unknown` = "could not observe":** the probe did not run, the collector section reported an error, the data is stale.
  **A probe that ran and failed is bad, not unknown.** Timeout, DNS error and connection refused are bad.
- **An incident is closed** only by `ok` after `ok_n` confirmations. A transition critical → unknown keeps the incident open
  with its severity.
- **"No data":** an own `unknown` (not caused by the parent) for longer than 1800 s → a warning incident with `no_data=1`.
- **Debounce:**
  - `unknown` and updates with `origin=sweep` apply immediately;
  - `ok` with no open incident applies immediately;
  - everything else goes through `bad_n`/`ok_n`. warning and critical are one class "bad": from ok, W and C observations
    accumulate a common streak and the last observation is applied; **a severity change of an already bad signal also needs
    `bad_n` consecutive observations** (W↔C flapping does not jump the incident or clear the ack).
- **Only `origin=data` refreshes `last_update_at` and `value`.**
- **Parent:** the engine **does not overwrite** children's statuses. Children become unknown by their own `stale_after_s`.
  The parent affects only suppression in the sender (§10.1) and the "no data" timer.
- **An incident keeps the last bad reason:** `inc.message` is updated only by a bad observation; `inc.title` always.
- A `signals` row appears on the first `apply` — from data or from a passed deadline (then unknown without a `status_history`
  row). Until then the UI shows "awaiting data".
- `status_history` is written only on a status change. Message changes are not recorded.
- **The engine assigns fields only when they differ** from the current values: a repeated sweep with no changes produces
  zero UPDATEs in `signals` and `incidents` (tested with a `before_cursor_execute` counter). The only unconditional sweep
  write is `internal.sweep_at`.
- The engine clips `message` to 300 and `title` to 120 characters with "…".
- CRUD does **not** close incidents itself. Disabling, deleting and removing an expectation are reflected by the next sweep
  (≤ 15 s) through `reconcile`.
- A probe or ingest that completes after the object was disabled checks `enabled` in the same transaction before `apply`.
  If the object is disabled, the result is dropped.

### 6.5 Engine: normative pseudocode

```text
apply(s, updates, now):
  for u in updates:                                     # input order; the heartbeat comes first
    u = clip(u)                                         # message ≤ 300, title ≤ 120
    sig = get(u.key) or create(u.key, status=UNKNOWN, since=now, streak=0)   # object_key/kind/subject via parse_key
    pol = KIND_POLICY[sig.kind]
    set_if_changed(sig.title, u.title)
    if u.origin == "data":                              # ONLY data refreshes the deadline
        set_if_changed(sig.last_update_at, now); set_if_changed(sig.value, u.value)
    inc = open_incident(u.key)
    new = decide(sig, u, pol, has_inc = inc is not None)
    if new is None:                                     # status does not change
        set_if_changed(sig.message, u.message)
        if inc:
            set_if_changed(inc.title, u.title)
            if is_bad(u.status): set_if_changed(inc.message, u.message)   # the last bad reason
        continue
    old = sig.status
    sig.status, sig.since, sig.message = new, now, u.message
    add status_history(old, new); log signal.status_changed
    on_change(sig, pol, inc, new, now)

decide(sig, u, pol, has_inc):
  obs = u.status
  if obs == sig.status:                         reset(sig); return None
  if obs == UNKNOWN or u.origin == "sweep":     reset(sig); return obs      # no debounce
  cls = "ok" if obs == OK else "bad"
  if cls == "ok" and not has_inc:               reset(sig); return OK       # nothing to close
  sig.streak = sig.streak + 1 if sig.streak_class == cls else 1; sig.streak_class = cls
  if sig.streak >= (pol.ok_n if cls == "ok" else pol.bad_n): reset(sig); return obs
  return None

reset(sig): set_if_changed(sig.streak, 0); set_if_changed(sig.streak_class, None)

on_change(sig, pol, inc, new, now):
  if pol.notify == "summary": return                    # summary-only kinds have NO incidents
  if new in (WARNING, CRITICAL):
     if inc is None: open Incident(signal_key, severity=new, message=sig.message, title=sig.title, opened_at=now)
     else:
        if rank(new) > rank(inc.severity): inc.acked_at = inc.acked_by = None        # escalation clears the ack
        inc.severity = new; inc.no_data = False; inc.message = sig.message; inc.title = sig.title
  elif new == OK and inc: resolve(inc, "recovered", now)
  # new == UNKNOWN: the incident is untouched, severity and message stay

open_no_data(s, now):                     # called by the sweep after apply
  for sig in signals where status == UNKNOWN and KIND_POLICY[kind].notify == immediate and no open incident:
     p = get(parent_key(sig.key)) if parent_key(sig.key) else None
     if p and p.status != OK: continue                  # unknown because of the parent
     start = max(sig.since, p.since if p else 0)
     if now - start >= 1800:
        open Incident(severity=WARNING, no_data=True, message="no data: " + sig.message, title=sig.title)

reconcile(s, expected_keys, now):          # called by the sweep FIRST; the only path to "removed from monitoring"
  for sig in signals where key ∉ expected_keys:
     if inc := open_incident(sig.key): resolve(inc, "removed", now)
     delete sig                                          # status_history stays

resolve(inc, reason, now): inc.resolved_at = now; inc.resolve_reason = reason
```

The announced severity is not lowered on de-escalation: after C→W→C the incident stays announced as critical and no second
"escalation" is sent (the incident is open all the time; the escalation W→C clears the ack, so reminders resume).

**Invariants** (property tests with `hypothesis` on random `apply`/sweep sequences, mandatory in S1):
- I1: an open incident ⇒ the signal status ≠ ok;
- I2: an immediate kind in warning or critical ⇒ an open incident exists;
- I3: a summary-only kind has no incidents;
- I4: after a sweep the `signals` keys ⊆ expected;
- I5: after a sweep there is no immediate signal in unknown without an incident for longer than 1800 s while its parent is ok.

### 6.6 Deadlines and the downtime shift

- Deadlines are computed by the sweep on the fly from the configuration and `last_update_at`. There is no stored `expect_by`.
- **Shift by the monitor's downtime:** before starting the loops `main` reads `internal.sweep_at` into `ctx.down_from`
  (if the key is absent — `process_started_at`). For a signal:

  ```text
  base     = max(sig.last_update_at ?? 0, e.base_at)
  shift    = max(0, process_started_at − max(base, down_from))
  deadline = base + window_s + shift
  now > deadline  →  SignalUpdate(status=e.deadline_status, message=e.deadline_message, origin="sweep")
  ```

  A 30-min downtime shifts deadlines by 30 min (no avalanche after downtime); a 10-s restart shifts them by 10 s. Frequent
  restarts do not postpone deadlines forever (that would be a silent false OK). The "no data" timer (`open_no_data`) has no
  shift: a signal that stayed unknown across a downtime honestly has no data.
- Jobs and self-signals have `window_s = None`: their status is computed by `evaluate_job` and `evaluate_self`. Jobs have
  no shift: a missed run is real.
- `catalog.expected_signals(s, now)` builds the expected signals:

| Signal | window_s | base_at | deadline_status | deadline_message |
|---|---|---|---|---|
| enabled check | max(3×interval_s, 900) | `enabled_at` | unknown | `check has not run for {N} min` |
| heartbeat of an enabled host | `stale_after_s` | `enabled_at` | `heartbeat_severity` | `no reports for {N} min` + `; reports arrive but are rejected: {reason}` if `last_reject_at > last_report_at` |
| expectation of an enabled host | `stale_after_s` | max(`created_at`, host `enabled_at`) | unknown | `no fresh data for {N} min` |
| enabled job | None | — | — | — |
| `self:telegram`, `self:snapshot`, `self:disk` | None — only those in `IMPLEMENTED_SELF` | — | — | — |
| `self:deadman` | None — only if implemented and `deadman.ping_url` is set | — | — | — |

`{N}` = `window_s // 60` (configuration, never the elapsed time, so a repeated sweep writes nothing); a NULL
`last_report_at` counts as −∞ in the `last_reject_at > last_report_at` comparison.

`IMPLEMENTED_SELF` is a constant in `constants.py`: S1 — `{telegram}`; from S3 — all four. Self-signals and jobs use the
`ExpectedSignal` defaults (§4.1). The sweep applies only updates whose keys are expected.

---

## 7. Checks and probes

### 7.1 Parameters and validation (`schemas/checks.py`)

`CheckParams` is a pydantic discriminated union by `type`, `extra="forbid"`, frozen. The database stores
`model_dump(mode="json")` with all defaults (a stable export round trip). `type` never changes after creation.

```text
http: {url, method: GET|HEAD = GET, expected_status: "200-299", body_contains: str|null,
       follow_redirects: bool = true, verify_tls: bool = true}
tcp:  {host, port}
tls:  {host, port = 443, server_name: str|null}          # 21/7-day thresholds are constants
dns:  {name, rtype: A|AAAA|CNAME|MX|TXT|NS, resolver: IP|null, expected: [str]|null}
```

| Field | Rule |
|---|---|
| `url` | absolute http/https with a host, ≤ 2000 characters, **no userinfo** (otherwise 422: a password would end up in exports and audit) |
| `host` | an IP literal (IPv6 without brackets) or a hostname ≤ 253 characters without spaces, `/` or a scheme |
| `port` | 1..65535 |
| `server_name` | a hostname |
| `expected_status` | `^\d{3}(-\d{3})?(,\d{3}(-\d{3})?)*$`, codes 100..599, lo ≤ hi; the normalized string is stored; the parser is a pure function in `domain/text.py` |
| `body_contains` | 1..200 characters; together with `method=HEAD` → 422 "body_contains requires GET" |
| `name` (dns) | `dns.name.from_text`, `_` allowed; the name is absolute, search domains are not applied |
| `resolver` | an IP address, port 53 |
| `expected` | `null` or ≥ 1 item (an empty form field = null; `[]` → 422); stored normalized (§7.2); for A/AAAA an address family mismatch → 422 |

Targets are not filtered by address (§1.3). Custom headers, authentication and request bodies are not supported.

### 7.2 Probe semantics (`probes/*`)

**General:**
- `async def run_probe(check: CheckConfig, now: int, ssl_ctx=None) -> ProbeResult`. The probe fully decides `ok` and `error`.
- The whole probe (DNS, connect, TLS, redirects, body) runs under `async with asyncio.timeout(timeout_s)` after acquiring
  the semaphore; `TimeoutError` and `httpx.TimeoutException` → `timeout {N} s`.
- The probe catches `Exception` (not `BaseException`: `CancelledError` propagates); a cancelled probe does not apply.
- Latency is measured with `time.perf_counter()` inside the semaphore: http — until the headers of the final response
  (including all hops); tcp — until the connection is established; tls — until the handshake completes; dns — until the answer.
  No response or connection → `latency_ms=None`.
- **Certificate trust:** a single builder `probes.make_ssl_context()` = `ssl.create_default_context()` (system store) for
  http and tls; tests pass their own `ssl_ctx` (`trustme`). A private CA is a backlog item.

**http:**
- a new `httpx.AsyncClient(trust_env=False, verify=ssl_ctx|False, follow_redirects=…, max_redirects=20)` per probe, without
  a keep-alive pool (a reused connection would check neither DNS nor the certificate);
- User-Agent `smon/<version>`;
- `follow_redirects=true` — the final response code is checked; more than 20 redirects → bad;
  `false` — the first response is checked, a 3xx is matched against `expected_status`;
- the body is read only if the code matched and `body_contains` is set: the first 1 MiB after decompression, a byte-wise
  search (`s.encode() in body`), case-sensitive;
- success = code ∈ `expected_status` and the substring is found (if set);
- an http check does not track certificate expiry — a separate tls check does (hint in the UI).

**tcp:** `asyncio.open_connection`; success — the connection is established.

**tls:**
- `asyncio.open_connection(host, port, ssl=ssl_ctx, server_hostname=server_name or host)`; the chain and the name are always verified;
- for an IP host no SNI is sent and the IP SAN is checked; a certificate issued for a name needs `server_name` (hint in the UI);
- only implicit TLS is supported; STARTTLS is a backlog item;
- expiry: `ssl.cert_time_to_seconds(getpeercert()["notAfter"])`, `tls_days_left = floor((notAfter − now) / 86400)`;
- **an expired certificate** (`SSLCertVerificationError.verify_code == 10`) → `tls_days_left=-1`, `tls_not_after=None`, `ok=False`.

**dns:**
- `dns.asyncresolver.Resolver(configure=resolver is None)`; with a configured resolver — `nameservers=[resolver]`, port 53;
  `lifetime` = the remaining timeout;
- for A/AAAA/MX/TXT/NS the rrset of the final name (`Answer.rrset`) is compared; for `rtype=CNAME` — only the first hop;
- **normalization** (one pure function `normalize_dns(rtype, s)` for answers and `expected`):
  A/AAAA — `str(ipaddress.ip_address(x))`; CNAME/NS — lowercase without the trailing dot; MX — `"{pref} {host}"`;
  TXT — `b"".join(strings).decode("utf-8", "replace")`, lowercase;
- `dns_answers` are sorted and deduplicated; with `expected` the **sets are compared for equality**; without `expected`
  success = at least one record.

### 7.3 Evaluator (`evaluate_check`)

- `ok` → OK, except tls: `tls_days_left < 7` → critical, `< 21` → warning.
- not `ok` → `failure_severity`. Exception: an expired certificate (`tls_days_left == -1`) → critical regardless of `failure_severity`.
- The date in TLS texts is UTC (the evaluator has no time zone).

### 7.4 Texts (normative catalogue, `probes/texts.py`, golden test)

Error texts are produced by the probe (`probes/texts.py`); OK and TLS-threshold texts are constants in `evaluators/checks.py`
(the layers forbid evaluators from importing probes). Golden files cover both, one per module:
`tests/golden/check_texts.txt` (evaluator texts, S1.1) and `tests/golden/probe_texts.txt` (probe errors, S1.2). `error` is
clipped to 200 characters.

| Situation | Text |
|---|---|
| unexpected code | `HTTP {code} (expected {expected_status})` |
| substring missing | `substring "{s≤40}" not found in response` |
| timeout | `timeout {N} s` |
| connection refused | `connection refused` |
| DNS error while connecting (gaierror) | `DNS: cannot resolve {host}` |
| redirects | `too many redirects` |
| other connection error | `connection error: {strerror}` |
| TLS | `TLS: certificate expired` \| `TLS: certificate is not valid for {name}` \| `TLS: untrusted certificate` \| `TLS: error: {verify_message}` |
| DNS check | `DNS: name does not exist` \| `DNS: no {rtype} records` \| `DNS: server returned an error` \| `DNS: got {a, b}, expected {c}` |
| OK | `OK, {ms} ms`; tls — `OK, certificate valid for {d} more days (until {YYYY-MM-DD}), {ms} ms` |
| TLS threshold | `TLS: certificate expires in {d} days ({YYYY-MM-DD})` |
| anything else | `error: {Type}: {str}` |

### 7.5 Scheduling (`loops/probe_scheduler.py`)

- In memory every enabled check has `next_due` (by `clock.monotonic()`) and an `in_flight` flag; probe tasks are kept in the
  scheduler's set and cancelled in `LoopSpec.on_stop`.
- The probe tick (5 s): re-reads enabled checks (`catalog.checks.enabled`); new ones get `next_due=0`, deleted and disabled
  ones are dropped from the map; a changed `updated_at` resets `next_due` (saving the form triggers an immediate probe).
- Ids from `runtime.probe_now` ("check now", §12) get `next_due=0` unless the check is `in_flight`; the set is cleared.
- For every check with `next_due <= now_mono` and not `in_flight` a task is created; `next_due = now_mono + retry_interval_s`
  is set immediately (protection against repeats if the task fails). At most 10 probes run concurrently (semaphore).
- The task: `await run_probe(...)` → `with tx`: check `enabled` and `updated_at` (otherwise the result is dropped and `next_due`
  is untouched) → `evaluate_check` → `apply` → `next_due = now_mono + (retry_interval_s if the observed status ≠ the effective
  one after apply, else interval_s)`. `in_flight` is cleared in `finally`. Task errors are logged (`loop.error`).
- The `retry_interval_s` step continues while a status change is being confirmed in either direction; a steady bad state uses `interval_s`.
- After startup all enabled checks run immediately (the startup wave is bounded by the semaphore).
- Latency is stored only in `signals.value` (metrics history is a backlog item).

---

## 8. Jobs

A job is any scheduled task that reports `start` and `finish` through the generic API (§8.2). smon decides whether it
started on time, finished in time, what the result was and whether the data is fresh. The calls are embedded wherever needed:
the `smon-job` wrappers (§17.4), a script's own `curl`, or a post-job hook of any scheduler or backup tool. smon has no
vendor-specific integrations.

### 8.1 Configuration and validation

- `schedule_kind=cron`: `cron` is parsed by `cronsim`, `tz` by `zoneinfo` (`CronSimError` or an unknown tz → 422).
- `schedule_kind=interval`: `interval_s ≥ 60`.
- `start_grace_s ≥ 60` and `start_grace_s < min_gap / 2`, where `min_gap` is the minimal interval between the next 20 slots
  from the moment of saving (for interval — `interval_s`).
- `max_runtime_s > 0` — mandatory.
- `max_data_age_s` — optional; if set, `max_data_age_s > max_gap + start_grace_s` (`max_gap` over the same 20 slots):
  otherwise stale would fire on every cycle.
- `name` ≤ 100.
- `enabled_at` is updated when enabled, `schedule_changed_at` when cron, tz or interval change.
- `domain/schedule.py` is the only wrapper around `cronsim`:
  - `next_slot(cron, tz, after: int) -> int` — the first slot **strictly greater** than `after`
    (`next(CronSim(expr, datetime.fromtimestamp(after, ZoneInfo(tz))))`; cronsim truncates to the minute and adds a minute);
  - `slots(cron, tz, after, n) -> list[int]`; `min_gap(cron, tz, after, n=20) -> int`; `max_gap(...)`.
  It is used by the evaluator, validation, the UI preview and the "expected by" column (§12.6). Test: `after` exactly on a slot
  → the next slot.

### 8.2 Events (ingest)

Bodies are validated with `extra="forbid"`. Timestamps are RFC 3339 strings with an offset (`AwareDatetime`, strings only).

**Start:** `POST /api/v1/jobs/{id}/start`, body `{"run_id"?: "[A-Za-z0-9._-]{1,64}"}` (an empty body = `{}`).
- Without `run_id` the server generates a uuid4 hex.
- A new run (`started_at = now`) → 202 `{"result":"accepted","run_id":…}`.
- A run with this `run_id` already exists (in any state) → 200 `{"result":"duplicate","run_id":…}`.
- In the same transaction: `evaluate_job(origin="data")` + `apply` (an open run immediately clears missed, §8.4).

**Finish:** `POST /api/v1/jobs/{id}/finish`,
body `{"status":"success|warning|failed","run_id"?,"exit_code"?:int,"data_time"?:RFC3339,"message"?:str≤1000}`.
- `run_id` given and the run is finished → 200 `duplicate`. **The first result wins.**
- `run_id` given and the run does not exist → a run with `started_at=NULL` is created.
- `run_id` not given → the latest run by `(started_at, id)` with `finished_at IS NULL` is used; if there is none, a new run is created.
- `finished_at = now`, `data_time = data_time ?? now`.
- `data_time` more than 300 s in the future → 422.
- In the same transaction: `evaluate_job(origin="data")` and `engine.apply`. Response 202 `{"result":"accepted","run_id":…}`.

Authorization: the token of this job (`kind=job`, `job_id`).

### 8.3 JobRunsView (`catalog.jobs.runs_view`)

- `last_begin_at = max(coalesce(started_at, finished_at))` over all runs;
- `last_finished_at = max(finished_at)`;
- `open_run_started_at` = **the earliest** `started_at` among open runs (`finished_at IS NULL`, `started_at IS NOT NULL`)
  started after `last_finished_at`; if no run has finished yet — among all open runs. A later start does not displace an
  earlier open run, so a job whose `finish` never arrives reaches overrun (test: 3 starts without finish with a 1-h period
  and `max_runtime_s` = 4 h → overrun). A single lost finish is covered by the next finished run;
- `last_finished` = `RunInfo` of the run with the largest `(finished_at, id)`;
- `last_good_data_time` = max `data_time` over runs with status success or warning;
- `last_reject_at`, `last_reject_reason` — from `jobs`.

### 8.4 Evaluator (`evaluate_job`)

`anchor = max(last_begin_at + start_grace_s, last_finished_at, enabled_at, schedule_changed_at)` (None terms are skipped).

| Reason | Condition | Status |
|---|---|---|
| missed | **only if there is no open run** (`open_run_started_at is None`): cron — `now > next_slot(anchor) + start_grace_s`; interval — `now > anchor' + interval_s + start_grace_s`, where `anchor' = max(last_begin_at, last_finished_at, enabled_at, schedule_changed_at)` | critical |
| overrun | `open_run_started_at` is set and `now > open_run_started_at + max_runtime_s` | critical |
| failed / warning | status of `last_finished`: failed → critical; warning → warning (or ok with `warning_as=ok`) | critical / warning |
| stale | `max_data_age_s` is set and `now − (last_good_data_time ?? enabled_at) > max_data_age_s` | critical |

- The status is the worst of the active reasons.
- An open run disables missed: otherwise a job longer than its period would cause a systematic false alarm. Its deadline is overrun.
- One `anchor` rule covers an early and a late start, a manual run, finish-only jobs, enabling and a schedule change without
  retroactive alarms. `last_finished_at` in the anchor keeps a long run from silently swallowing the next slot.
- If a job sends only `finish`, `start_grace_s` must cover the execution time (hint in the UI).
- `next_expected_at(job, runs)` = `next_slot(anchor)` for cron, `anchor' + interval_s` for interval; it may be in the past.
- Until a job has received events, its status is ok while no reason is active.

**Texts (normative).** Time is always `DD.MM HH:MM` in `display_tz`; the message lists the active reasons separated by
`"; "`, the engine clips it (≤ 300):

| Reason | Text |
|---|---|
| missed cron | `missed run {DD.MM HH:MM} ({tz})` (slot = `next_slot(anchor)`) |
| missed interval | `no run for more than {duration(interval_s + start_grace_s)}` |
| overrun | `running for more than {duration(max_runtime_s)} (started {DD.MM HH:MM} {tz})` |
| failed / warning | `last run {status}[, exit {code}][: {message[:100]}]` |
| stale | `data older than {duration(max_data_age_s)} (latest {DD.MM HH:MM} {tz})` \| `no successful data` |
| ok | `last run {status} {DD.MM HH:MM}` (from the last finished run); no finished run but an open one — `running since {DD.MM HH:MM} {tz}`; `no events yet` only when the job has no runs |
| last rejection | `; last event rejected: {reason}` — if `last_reject_at > (last_begin_at ?? −∞)` |

A job that is killed together with its wrapper leaves its run open → overrun. A `finish` with a new `run_id` while an older
run is open creates a finished run; the older open run started before it no longer counts (§8.3). The UI shows such runs as "open".

---

## 9. Hosts and collector report v1

### 9.1 Envelope

| Field | Type | Required |
|---|---|---|
| `schema_version` | int, = 1 | yes |
| `collector_version` | str 1..32 (= the release version `X.Y.Z`) | yes |
| `sent_at` | RFC 3339 with an offset, at most 6 fractional digits (the collector formats `yyyy-MM-ddTHH:mm:ss.ffffffZ`); set **immediately before the POST** | yes |
| `os_family` | `"linux"` \| `"windows"` | yes |
| `hostname` | str 1..255 | yes |
| `os` | str 1..200 | yes |
| `boot_time` | RFC 3339 with an offset (display of uptime only) | no |

`sent_at` and `boot_time` are **not** checked for being in the future: a fast host clock produces a clock warning, not a 422.

### 9.2 Sections

Every section is optional. A missing section means "not collected". Non-applicable sections are **omitted**, not reported in
`errors`. `errors[]` is only for real collection failures. **The collector itself keeps to the schema** (§17): it clips text
to the limits and skips items with invalid numbers (without reporting them in `errors[]`); on a duplicate key, too many items
or a partial failure it omits the whole section and reports it in `errors[]`.

```text
cpu:        {busy_pct: 0..100}                                                                     (display only)
memory:     {total_bytes > 0, available_bytes 0..total_bytes, commit_used_bytes?: >= 0, commit_limit_bytes?: > 0}
            # commit — both fields or none; Linux does not send commit (Committed_AS > CommitLimit is normal with overcommit)
disks:      [≤ 64]   {mount 1..200, fs ≤ 32, total_bytes > 0, free_bytes 0..total_bytes}          key: mount
services:   [≤ 2000] {name 1..200, state: running|stopped|starting|stopping|paused|failed|other,
                      start_mode?: auto|manual|disabled|other, display_name?: ≤ 256}                  key: name
containers: [≤ 500]  {name 1..200, state: running|restarting|paused|exited|created|dead|removing,
                      health: healthy|unhealthy|starting|none, restart_count >= 0, oom_killed: bool, image ≤ 256}  key: name
vms:        [≤ 200]  {name 1..200, state: running|off|saved|paused|starting|other}                   key: name
storage:    [≤ 200]  {name 1..200, type: pool|virtual_disk|physical_disk, health: healthy|warning|unhealthy|unknown}  key: (type, name)
errors:     [≤ 20]   {section: str ≤ 32, error ≤ 500}   # unknown section names are ignored; repeats are joined with "; "
```

**Server-side validation:**
- Unknown fields are ignored (compatibility with newer collectors; sections may be added later without a `schema_version` bump).
- An unsupported `schema_version`, a duplicate key inside a section or any schema violation → 422 for the whole report
  (fail loud; the reason is visible in `last_reject` and in the heartbeat text).
- `hosts.last_report` is the `model_dump(mode="json")` of the validated report, not the raw body.
- `os_family` from the report is written to `hosts.os_family`; `hostname` and `collector_version` are shown on the host page.

### 9.3 State mapping (done by the collector)

| Source | Value | State |
|---|---|---|
| Linux: `systemctl show -p Id,LoadState,ActiveState,UnitFileState` in one call for the units from `list-units --type=service --all`; only `LoadState=loaded`; ActiveState | active | running |
| | activating, reloading | starting |
| | deactivating | stopping |
| | inactive | stopped |
| | failed | failed |
| | other (maintenance, refreshing) | other |
| Linux: UnitFileState | enabled, enabled-runtime | auto |
| | disabled, masked | disabled |
| | other (static, indirect, generated) | other |
| Windows: `Status` | Running | running |
| | Stopped | stopped |
| | StartPending, ContinuePending | starting |
| | StopPending, PausePending | stopping |
| | Paused | paused |
| Windows: `StartType` | Automatic | auto (trigger-start → other) |
| | Manual | manual |
| | Disabled | disabled |
| | other | other |
| Hyper-V: `Get-VM`.State | Running, Off, Saved, Paused, Starting | the same |
| | anything else | other |

Windows: per-user services (`ServiceType & 0x40`) are not reported; a service with `TriggerInfo` gets `start_mode=other`.

**File system filter:**
- Linux: allowlist `fstype ∈ {ext2, ext3, ext4, xfs, btrfs, zfs, vfat, exfat, f2fs}`; network file systems are excluded
  (`statvfs` may hang); mount points under `/var/lib/{docker,containerd,kubelet}`, `/run`, `/snap` are excluded. Deduplicated
  by `st_dev`, keeping the shortest mount path. `total = f_blocks · f_frsize`, `free = f_bavail · f_frsize` (as `df`).
- Windows: `Win32_LogicalDisk` with `DriveType=3`.

**Storage:** name = `{FriendlyName} ({SerialNumber | DeviceId})`; primordial pools are not reported. The storage section does
not see hardware RAID degradation (a coverage boundary). Linux mdadm is a backlog item.

### 9.4 Expectations and discovery

- All host sub-signals except the heartbeat exist **only** as `host_expectations` rows. An expectation's kind and subject are
  immutable; thresholds and `expected_state` can be edited.
- **Discovery** (`/hosts/{id}/report`) is built from `hosts.last_report` and only **adds** expectations. Preselected by default:
  - services with `start_mode=auto` and `state=running` (auto services that are not running are unchecked and highlighted
    "auto, not running");
  - containers in `running` state (others are unchecked, with their state shown);
  - all disks (they already passed the file system filter);
  - all VMs: `expected_state = off` if the VM is off, otherwise `running`;
  - singleton kinds memory and storage — if their section is present in the report **or listed in `errors[]`**; clock — always.
- **Manual addition** — kind + subject (§6.1) and `expected_state` for vm (default running), also before the first report:
  if the section is present → critical "missing", if the section is absent → the deadline.
- Removing an expectation deletes the row; the signal leaves through `reconcile` as "removed from monitoring".
- A change of `os_family` or a reinstalled collector needs no special handling: old expectations become "missing" or "no data".
- **Threshold validation** when saving an expectation or the settings: the effective pair (override + global)
  `1 ≤ crit < warn ≤ 99`, `reserve_bytes > 0`. The evaluator does not validate thresholds: it checks critical first.

### 9.5 Host evaluator (`evaluate_report`)

Order: first the heartbeat (ok on any accepted report, message `report received`), then the expectations. For each expectation:
- the section is listed in `errors[]` (even if present) → unknown `collection error: {error}`;
- the section is absent → no update (the deadline will fire);
- the section is present, the item is missing → critical `missing from report`;
- the item is present → the rule.

| kind | Section | Rule | value |
|---|---|---|---|
| disk | disks | `free_pct < crit` → critical; `free_pct < warn` or `free_bytes < reserve_bytes` → warning | free_pct |
| memory | memory | `avail_pct < crit` → critical; `< warn` → warning; with both commit fields — `used/limit·100 > commit_warn_pct` → warning; the worst wins | avail_pct |
| service | services | ok only if `running` | — |
| container | containers | ok if `running` and `health != unhealthy` | — |
| vm | vms | ok if `state == expected_state` | — |
| storage | storage | empty list → critical `no storage items`; any warning/unhealthy → critical with the list of bad items; otherwise any unknown → unknown `state unknown: {names}`; otherwise ok | — |
| clock | envelope | `\|sent_at − received_at\| > clock_warn_skew_s` → warning | skew_s (signed: `sent_at − received_at`) |

Thresholds: the override from the expectation row, otherwise `thresholds.*` (§2.3). Thresholds are compared with the
unrounded value.

**Texts (normative, golden tests; bytes via `fmt.bytes`, base 1024, "GiB"):**

| kind | Text (ok — the same facts without the reason) |
|---|---|
| heartbeat | `report received` |
| disk | `{pct:.1f}% free ({free} of {total})` [+ `; below reserve {reserve}`] |
| memory | `{pct:.1f}% memory available` [+ `; commit {c:.0f}%`] |
| service, container | `state: {state}`[`, unhealthy`] |
| vm | `state {state}, expected {expected}` |
| storage | `problems: {name} — {health}, …` \| `no storage items` \| `state unknown: {names}` \| `ok: {n} items` |
| clock | `clock skew {±N} s` |

Several reasons are joined with "; ". All texts go through `domain.text.clip`.

### 9.6 Report example

```json
{"schema_version":1,"collector_version":"1.0.0","sent_at":"2026-09-29T09:00:00+00:00",
 "os_family":"linux","hostname":"host-1","os":"Ubuntu 24.04",
 "cpu":{"busy_pct":12.5},"memory":{"total_bytes":8589934592,"available_bytes":1932735283},
 "disks":[{"mount":"/","fs":"ext4","total_bytes":123480309760,"free_bytes":62277025792}],
 "services":[{"name":"docker","state":"running","start_mode":"auto"}],
 "errors":[{"section":"containers","error":"docker socket: permission denied"}]}
```

---

## 10. Notifications: sender, templates, daily summary, poller

### 10.1 Model (level-triggered)

There is no message queue. An incident stores what the human has already been told: `announced_severity` (null until
announced), `announced_at` and `resolve_announced`. Every 5 s the sender compares this with the current state.

**Suppression** (`queries.suppression(s, now)`) is evaluated at selection time and applies **only to new alarms and reminders**:

```text
suppressed(key) = any(silence_matches(t.target, key) for t in active silences)
                  or (parent_key(key) and parent.status != OK)
# active silence: cancelled_at IS NULL and ends_at > now; no parent row → treated as ok
```

**Candidates** (in one `tx`; the result is `Item` snapshots (§4.3), not ORM objects; one Item per incident, the first matching rule):

```text
open, not suppressed:
   announced_severity IS NULL                                  → opened
   rank(severity) > rank(announced_severity)                   → escalated
   severity = critical, acked_at IS NULL,
     now − announced_at ≥ reminder_interval_h·3600             → reminder
   announced, acked_at IS NULL, ∃ silence m: silence_matches(m.target, key)
     and announced_at < end(m) ≤ now, where end(m) = min(ends_at, cancelled_at ?? ends_at)
                                                               → reminder   # end of a mute/silence: "remind me later", any severity
resolved, announced_severity NOT NULL, resolve_announced = 0   → resolved | removed (by resolve_reason); NOT suppressed
resolved, announced_severity IS NULL                           → no message (listed in the daily summary as "closed without notification")
daily summary due (§10.4)                                      → summary
```

`old_severity` is `announced_severity` at selection time (for `OLD_MARK`).

**Order within a message:** critical opened and escalated → warning opened and escalated → reminder → resolved → removed.
Within a group — by `opened_at`, then `id`. The daily summary is a separate message after the incidents.

### 10.2 Sending (`notify/sender.py`)

- `run_once` reads `cfg = settings.telegram(s)` in its transaction and selects candidates; only if there are candidates does it
  enter `open_client(cfg)` (§4.3) outside the transaction and send. No client while candidates exist is an error "Telegram is not
  configured" (counts toward the undelivered queue). Incidents are sent first (one `split_parts`), then the summary (its own
  `split_parts`).
- `render.split_parts(blocks, limit=4000) -> list[Part]`, `Part(text, block_idx)`: blocks are joined with an empty line and
  split only between whole blocks. A block longer than the limit is split at `\n`; a line longer than the limit is clipped to
  limit − 1 with "…". Length is counted in UTF-16 code units (`len(s.encode("utf-16-le")) // 2`) including the part header
  `({i}/{n})`, which is the first line and only when n > 1. Invariant: no part exceeds the limit. The daily summary and
  `/status` are split the same way.
- Plain text, no `parse_mode`.
- An Item is marked after the part that contains its **last** fragment. The `[Ack] [Mute 1h]` buttons and a `tg_messages` row
  (with the ids of the open incidents in the part) are added only if the part contains opened, escalated or reminder items.
- 1 s between parts (`await clock.sleep(1)`; the Telegram per-chat rate).
- After **each part** succeeds — a separate `tx` with `mark_sent(s, items, now, chat_id=…, message_id=…, local_today=…)`:
  - opened, escalated, reminder: `announced_severity = item.severity_sent`, `announced_at = now`;
  - resolved, removed: `resolve_announced = 1`;
  - summary: `internal.summary_sent_date = local_today`, `internal.summary_sent_at = now`.
- If part k fails, parts k+1… are not sent in this tick. `delay = min(10 · 2^(attempts−1), 300)` s; on 429 —
  `max(delay, retry_after)`. Until `sender_backoff_until` a tick returns immediately. Nothing is dropped. A successful tick →
  `sender_attempts = 0`.
- Errors: `runtime.tg_send_error` = `"{status} {description}"` for `kind=http`, `"network: {description}"` for
  `kind=network`, `"Telegram is not configured"` when there is no client (this case also advances `sender_attempts` and the
  backoff); reset on a successful send.
- **Undelivered queue:** `runtime.undelivered_since` = the time of the first failed attempt in the current failure series; reset when
  a tick sent everything or there was nothing to send. `undelivered_age > 600` → health unhealthy (`/fail`, §13.3), a UI banner.
  `runtime.pending_count` = the number of candidates after the tick. A due daily summary is a candidate like any incident, so
  a broken channel produces `/fail` in a quiet period as well — no later than `summary_time + 10 min`.
- A message that was sent but not marked (a crash between send and commit) is sent again after a restart — acceptable.

### 10.3 Templates (normative, `notify/render.py`, golden tests `tests/golden/*.txt`)

`render_block(item, now, tz)`; all `render.*` functions are pure and receive `now` and `tz` as arguments.

```text
MARK:  critical → 🔴   warning → 🟠   ok → ✅   removed → ⚪
ts(x): HH:MM if x is today (in tz), otherwise DD.MM HH:MM
age / duration: <60 s → "N s"; <1 h → "N min"; <1 day → "N h M min"; otherwise "N d M h"; a zero M is omitted; rounded down

opened:     {MARK} {title}
            {message}
            since {ts(opened_at)} · #{id}
escalated:  {MARK} {title} — escalated {OLD_MARK}→{MARK}
            {message}
            open since {ts(opened_at)}, {age(opened_at)} · #{id}
reminder:   {MARK} {title} — reminder, not acknowledged for {age(opened_at)}
            {message} · #{id}
resolved:   ✅ {title} — recovered at {ts(resolved_at)}, lasted {duration(opened_at..resolved_at)} · #{id}
removed:    ⚪ {title} — removed from monitoring at {ts(resolved_at)} · #{id}

button answers:  "Acknowledged: {n}"  |  "Muted until {HH:MM}: {n}"  |  "No open incidents"  |  "Access denied"
test message:    "smon: test message ({instance_name}, {ts(now)})"
```

`title` and `message` are already clipped by the engine (120/300); render clips again with "…" if needed.

### 10.4 Daily summary

Built by `notify.summary.collect(...)` (the same object feeds the UI Overview and `/status`), sent by the sender.
- Condition: `local_now ≥ notify.summary_time` and `internal.summary_sent_date != local_today`. One rule covers both the
  scheduled time and a catch-up summary after downtime. In a tick incidents go first, then the summary.
- If part k fails, the whole summary is sent again (a duplicate is acceptable).
- The "closed without notification" window: `resolved_at > coalesce(internal.summary_sent_at, now − 86400)`.
- A section is a block; one entity per line inside. Empty sections are omitted except the first four lines.
- The daily summary is **mandatory** and is the safeguard against indefinite silence: acks, silences and disabled objects are
  visible every day.

```text
{instance_name} daily summary · {DD.MM.YYYY HH:MM} ({tz})
Monitor: healthy, up {age(uptime)} | PROBLEMS: {problems joined with "; "}, up {age(uptime)}
Dead-man: last successful ping {ts} | not configured | no successful ping since start
Undelivered: {pending_count}

Open incidents ({n}):
{MARK} {title} — {message} · {age}[ · ack {acked_by}][ · silenced until {ts}][ · parent unavailable] · #{id}
Closed without notification ({n}):
{MARK|⚪} {title} — {ts(opened_at)}–{ts(resolved_at)}
Silences ({n}):
{target} — until {ts(ends_at)} · {reason}
Disabled ({n}):
{title}
No data ({n}):
{title} — since {ts(since)}[ (parent unavailable)]
Summary-only ({n}):
{MARK} {title} — {message}
OK: {ok_count} signals.
```

`/status` is the header `{instance_name} status · {ts}`, the "Monitor" and "Undelivered" lines, the counters line
`ok {n} · warning {n} · critical {n} · unknown {n}` and the "Open incidents" section in the same format; it is split with `split_parts`.

### 10.5 Poller (`notify/poller.py`)

- `getUpdates(offset=internal.tg_offset, timeout=25, allowed_updates=["message","callback_query"])` with the token of `cfg`,
  through a client opened for this one long poll.
- **Every** update advances the offset — from another chat, a non-command, without text. Processing:
  `for upd: try: tx(action + offset = update_id + 1) except: log loop.error; tx(offset = update_id + 1)`.
  An exception on one update does not block the queue.
- **Token check:** before writing the offset or executing an action, the poller verifies that `settings.telegram(s).bot_token`
  in that transaction still equals the token used for this `getUpdates`; otherwise the batch is dropped (a long poll with an
  old token must not write an offset that the new bot will never reach).
- Filter: `str(chat.id) == telegram.chat_id` (for a callback — `str(callback_query.message.chat.id)`) and
  `from.id ∈ allowed_user_ids`. A foreign callback → `answer_callback("Access denied")` + log `tg.rejected_user`;
  a foreign message is silently ignored.
- `ack` and `mute1h`: incidents are taken from `tg_messages(chat_id, message.message_id)`; they apply only to incidents still
  open (`engine.actions`, actor `tg:<user_id>`). `mute1h` creates a separate silence per `signal_key` of the message's open
  incidents, until `now + 3600`. `answer_callback` — after commit; its errors (e.g. 400 on an old query) are only logged
  (`notify.failed`, `kind=callback`).
- `/status`: the command is the first token of the text up to `@` (`text.split()[0].split("@")[0] == "/status"`); the bot
  suffix is not checked. The offset is committed first, then the answer is sent with `send_message` (to `telegram.thread_id`),
  without buttons, bypassing the sender. A failed answer is only logged (`notify.failed`, `kind=status`).
- Errors: 401 and 409 → `runtime.tg_poll_error = "{status} {description}"` and a 30-s pause. A network error or any other
  `TelegramError` (5xx, 429, 400) sets `poll_failing_since` (if unset) and pauses the poller (`retry_after` s on 429, 5 s
  otherwise); if `now − poll_failing_since > 300` → `tg_poll_error` as well. A successful `getUpdates` resets both fields. `deleteWebhook` is never called:
  409 means another consumer (or a webhook) — fail loud.
- `internal.tg_offset` is reset by `ctx.on_settings_changed` when a new `telegram.bot_token` is saved (update ids of a new bot
  are unrelated to the old ones).
- After `telegram.chat_id` changes, open incidents are not re-announced; the buttons of old messages answer "Access denied".
  UI hint: "after changing the chat, send /status". Anonymous group admins do not pass `allowed_user_ids` — also a hint.

---

## 11. API

### 11.1 General

- Prefix `/api/v1`. Fields are only added; breaking changes go to `/api/v2` only. Clients must ignore unknown fields.
- Timestamps in responses — `YYYY-MM-DDTHH:MM:SSZ` (exception: `last_report` is returned as stored).
- `request_id` (12 hex) is generated by the outer ASGI layer `web/dispatch.py`: it is put into `scope["state"]` and the log
  contextvar, and a `send` wrapper adds the `X-Request-ID` header to **every** response of both ports, including 500.
  An incoming header is ignored. If an exception escapes after the response has started, `dispatch` logs `http.error` once
  and swallows it (so `ASGITransport` tests with `raise_app_exceptions` still see the response).
- **Pagination (API)** — incidents and job runs: `?limit=` (default 100, 1..1000; out of range → 422) and `?before_id=`; `ORDER BY id DESC`, fetch limit + 1;
  response `{"items":[…],"next_before_id":int|null}`. Other lists (e.g. signals) are returned in full as `{"items":[…]}` —
  their size is bounded by capacity. Logs have their own contract (§11.3). UI lists that grow (closed incidents, runs, audit,
  status history) use the same `before_id` scheme with 50 items per page (§12.6).
- **The `key=` filter** everywhere has the semantics of `domain.keys.key_filter` (= `silence_matches`): an exact match or a
  prefix `k + ":"`; `key=host` returns all hosts. SQL without LIKE: `(:k = '*' OR key = :k OR substr(key, 1, length(:k) + 1) = :k || ':')`.
- OpenAPI and `/docs`, `/redoc`, `/openapi.json` are disabled on both ports (`docs_url=redoc_url=openapi_url=None`);
  the contract is this document. Test: 404.
- **Uniform error body:**

```json
{"error":{"code":"validation_error","message":"invalid report",
          "details":[{"field":"disks.0.free_bytes","reason":"must be <= total_bytes"}]},
 "request_id":"a1b2c3d4e5f6"}
```

| HTTP | code | When |
|---|---|---|
| 401 | `unauthorized` | missing or malformed header; the token does not start with `smon_`; sha256 not found; token revoked. Tokens of a deleted object are cascaded away → also 401. Response has `WWW-Authenticate: Bearer` |
| 403 | `forbidden` | the token is valid but its kind or object does not match the path (including a non-existent id); a read token on ingest; a host/job token on :8081 |
| 403 | `object_disabled` | the object is disabled |
| 403 | `admin_not_allowed` | the client IP is outside `SMON_ADMIN_ALLOW` (:8081, §15) |
| 403 | `csrf_failed` | CSRF check failed (:8081, HTML) |
| 404 | `not_found` | unknown path (routing, D21); unknown id in the read API |
| 405 | `method_not_allowed` | method not supported by the route; `Allow` header (D21) |
| 409 | `conflict` | login before a password is set; import into a non-empty instance |
| 413 | `payload_too_large` | body limit exceeded |
| 422 | `invalid_json`, `validation_error`, `unsupported_schema_version` | invalid data, including query parameters (`field` = `query.limit`) |
| 429 | `rate_limited` | login limiter (§15.2); plus `Retry-After` |
| 503 | `unavailable`, `safe_mode` | database unavailable (`OperationalError`) or SAFE_MODE |
| 500 | `internal` | anything else |

- Routing errors (D21): on :8080 and for `/api/*` paths on :8081 — the uniform JSON body; for other :8081 paths — a simple
  HTML page in the common layout.
- Handlers on both apps: `starlette.exceptions.HTTPException` (404/405; `exc.headers` are kept, otherwise `Allow` is lost),
  `RequestValidationError` (→ 422 `validation_error`), `ApiError`, `OperationalError` (→ 503), `Exception` (→ 500).
  `details` contain only loc (dot-joined) and msg, without `input`/`ctx`; at most 20 entries, reason ≤ 200 characters.
- **Clients** (collectors, wrappers): 202 and 200 `duplicate` mean success. Retry only on a network error, 429 (respecting
  `Retry-After`) and 5xx; other 4xx are not retried.

### 11.2 Ingest (:8080, public)

| Method and path | Body | Limit | Response |
|---|---|---|---|
| `POST /api/v1/hosts/{id}/report` | report v1 (§9) | 512 KiB | 202 `{"result":"accepted"}` |
| `POST /api/v1/jobs/{id}/start` | §8.2 | 16 KiB | 202 or 200 `duplicate` |
| `POST /api/v1/jobs/{id}/finish` | §8.2 | 16 KiB | 202 or 200 `duplicate` |
| `GET /healthz` | — | — | 200 or 503 `{"healthy":bool}` without details |

**Processing order (normative):**
1. SAFE_MODE → 503 `safe_mode`.
2. tx1: `catalog.tokens.authenticate` → None → 401; kind or object mismatch → 403 `forbidden`; `enabled=false` →
   403 `object_disabled` + `last_reject`.
3. Size: `Content-Length` above the limit → 413 without reading the body; otherwise `request.stream()` with a counter up to limit + 1.
4. JSON: `json.loads`; NaN/Infinity and non-UTF-8 → `invalid_json`; an empty body for start = `{}`, for others → `invalid_json`;
   a non-object top level → `validation_error`. `Content-Type` is ignored (`curl -d` sends form-urlencoded);
   `Content-Encoding` is not supported.
5. `schema_version` → 422 `unsupported_schema_version`; the schema (`model_validate`) → 422 `validation_error`.
6. Semantic checks (`data_time` > now + 300, etc.) → 422.
7. tx2: re-check `enabled`; duplicate → 200; intake and `apply` → 202 after commit.

- A rejection after tx1 (413, 422) with a valid token of this object is recorded by
  `ingest.record_reject(s, obj_kind, obj_id, reason, now)` in its own short transaction.
- Ingest routes take only `Request` (no pydantic body parameter: otherwise FastAPI parses the body before authentication and
  answers in its own format).
- Time in the body — only an RFC 3339 string with an offset (`AwareDatetime`; pydantic's lax mode would accept an epoch number).
- **No application rate limiting on ingest:** a 256-bit token cannot be brute-forced, collectors run from timers, and
  request-rate limiting belongs to the reverse proxy.
- **Client IP** — `domain.netaddr.client_ip(peer, xff, trusted)` (§15.4), used for logs.
- `last_reject_at` and `last_reject_reason` are written only with a valid token of this object for 403 `object_disabled`,
  413 and 422. Format: `"{http} {code}"` + `": {field}: {reason}"` of the first detail + ` (+N)` if there are several; ≤ 500 characters.

### 11.3 Read API (:8081; GET only)

**Authentication:** an `Authorization` header → only the token is checked (`kind=read`; invalid → 401 even with a live
cookie); no header → the admin session. No auth on `/api/*` → 401 JSON with `WWW-Authenticate: Bearer`, no redirect.
A Bearer on UI routes is ignored. There are no write methods in `/api/v1` (→ 405). `/healthz` on both ports needs no auth.

| Endpoint | Stage | Response |
|---|---|---|
| `GET /api/v1/status` | S3 | `{healthy, safe_mode, problems[], uptime_s, counts{ok,warning,critical,unknown}, open_incidents (int), unacked_critical (int: open critical with `acked_at IS NULL`), undelivered_count, undelivered_since, deadman{configured,last_ok_at,last_error}, telegram{send_error,poll_error}, version, now}`; counts — over `signals` rows; `undelivered_count` = `runtime.pending_count` |
| `GET /api/v1/incidents?state=open\|resolved\|all&key=` | S3 | items `{id, signal_key, title, severity, message, no_data, opened_at, resolved_at, resolve_reason, acked_at, acked_by, announced_severity, announced_at, silenced_until, parent_ok}`; `state` defaults to `open`; paginated |
| `GET /api/v1/signals?key=&status=` | S3 | items `{key, object_key, kind, subject, title, status, since, message, value, last_update_at, open_incident_id, silenced_until, parent_ok}`; `status` — a comma-separated list. Statuses of checks, jobs and hosts are read here (`key=check`, `key=job:<id>`, `key=host:<id>`) |
| `GET /api/v1/logs?level=&key=&contains=&since=&limit=500` | S3 | see below |
| `GET /api/v1/jobs/{id}/runs` | S4 | `{id, run_id, started_at, finished_at, status, exit_code, data_time, message}`; paginated |
| `GET /api/v1/hosts/{id}` | S5 | `{id, name, enabled, os_family, last_report_at, last_report, last_reject{at,reason}\|null}` |

- An endpoint exists from its stage on; before that the path returns 404.
- `silenced_until` — the maximum `ends_at` of the matching active silences, or null.
- `parent_ok`: null — no parent; no parent row → true (as in §10.1).
- Detail endpoints return the object without a wrapper; an unknown id → 404.

**Logs** (an exception to pagination):
- `items` — parsed JSON objects of log lines; a line that cannot be parsed is returned as `{"raw": "<line>"}`;
- `level` — the minimal level; `contains` — a case-insensitive substring of the raw line; `since` — RFC 3339 with an offset;
  `key` — `key_filter(key, signal_key)` of the line (lines without `signal_key` are found with `contains`);
- read from the end, at most 20 MiB in total across `smon.log`, `.1`…`.4`; the `limit` (1..5000) newest matches are
  returned, oldest first; a truncated first line is dropped;
- `truncated=true` if there are more matches than `limit` or the scan hit 20 MiB before the start of the oldest file.
- The UI "Logs" page calls the same `ops.logs_reader.read` function with the same parameters.

---

## 12. UI

**Normative and guidance.** Normative: §12.1, the banner area of §12.2, §12.4, the pages, routes and stages of §12.5, §12.7
(routes, form semantics, confirmations, PRG), §12.8 and any text used by other sections. Guidance (D104): the navigation order
and the settings submenu (§12.2), macro names and colors (§12.3), the composition of detail pages (§12.5), the list columns and
their order (§12.6). A deviation from guidance is a note in STATUS ("Notes for later stages"), never a blocking review finding
or a D record.

### 12.1 Stack and general rules

- Jinja2 (autoescape mandatory, `|safe` on data forbidden) + htmx 2.x (vendored, version in the file name) + own `app.css`
  + `app.js` ≤ 100 lines. No CSS/JS frameworks, no build step, no CDN.
- **Language:** English.
- **CSP compatibility (§15):** no inline scripts, handlers (`on*=`) or styles (`style=`, `<style>`); every `<script>` has `src`.
  - Confirmation — a `data-confirm="…"` attribute and a delegated submit/click handler in `app.js` (also used by the
    `confirm_post` macro); `hx-confirm` only on htmx elements.
  - `<meta name="htmx-config" content='{"allowEval":false,"allowScriptTags":false,"includeIndicatorStyles":false}'>`;
    the `.htmx-indicator` class is defined in `app.css`.
  - CSRF for htmx — only `hx-headers='{"X-CSRF-Token":"…"}'` on `<body>`; forms carry a hidden `csrf_token` field.
  - Test: the CSP header and a grep of templates for `on\w+=`, `style=` and `<script>` without `src`.
- **URLs:** only root-relative paths in templates and redirects (`/static/…`, `Location: /checks`); `redirect_slashes=False`.

### 12.2 Layout

- Navigation: Overview · Incidents · Checks · Jobs · Hosts · Silences · Logs · Settings. An item appears at the stage where
  its page appears. `POST /logout` in the header.
- Settings submenu: Parameters · Tokens · Export/import · Snapshots (file list, read-only) · Audit.
- Banner area:
  - SAFE_MODE: "monitoring is stopped, statuses frozen as of {internal.sweep_at}";
  - health problems (`problems`);
  - Telegram: "not configured" | "{n} undelivered since {age}" | "error {tg_send_error | tg_poll_error}";
  - dead-man: "not configured" | "error {deadman_last_error}".

### 12.3 Macros (`_macros.html`; guidance)

- `status_badge(status, labels)` → `OK / WARN / CRIT / UNKNOWN` + labels (computed by `queries`);
- `ts(epoch)` → `<time datetime="ISO-UTC" title="ISO-UTC">DD.MM.YYYY HH:MM</time>` in `general.timezone`;
  `ts_short(epoch)` — like `ts()` in Telegram (`HH:MM` today, otherwise `DD.MM HH:MM`); with seconds in logs and runs;
- `age(epoch)`, `duration(a, b)` — the §10.3 format;
- `confirm_post(url, label, question)`, `field(...)`, `pager(next_before_id)`, `empty(text, action_url)`.
- **Colors** (CSS variables): `--c-ok:#1b7f3b; --c-warning:#b26a00; --c-critical:#c62828; --c-unknown:#6b6b6b; --c-silenced:#5c6bc0`.

### 12.4 State display (normative)

| State | Display |
|---|---|
| an expected signal has no `signals` row yet | "awaiting data", grey, no badge |
| unknown | UNKNOWN "since {ts_short}"; with a no_data incident — label "no data" |
| `parent_ok=false` | label "parent unavailable" |
| active silence | "silenced until {ts_short}", color `--c-silenced` |
| acknowledged incident | "ack {by}" |
| summary-only kind | label "summary only" |
| disabled object | "disabled" instead of the status |

An incident row shows both the incident severity and the current signal status (otherwise critical → unknown is invisible).

### 12.5 Pages by stage

| Stage | Pages and actions |
|---|---|
| S0 | `/login` (until a password is set — instructions `docker exec … smon admin set-password`; POST → 409); `POST /logout`; `/` (placeholder); `/settings` — generic, rendered from the §2.3 registry by section: one form per section, `POST /settings/s/{section}` |
| S1 | Overview; `/incidents?state=&key=`, `/incidents/{id}`, `POST /incidents/{id}/ack` and `…/mute` (htmx → `_incident_row.html`); `/signal?key=<urlencoded>`; checks CRUD + `POST /checks/{id}/run`; `/silences`, `/silences/new?target=`, `POST /silences/{id}/cancel` (htmx → `_silence_row.html`); `POST /settings/telegram/test`; `/settings/audit` |
| S3 | `/logs`, `GET /logs/download`; `/settings/tokens` (read tokens); `/settings/snapshots` |
| S4 | jobs CRUD: page with status and reasons, `last_reject`, the next 5 slots in the job's tz, a hint about `start_grace_s`, runs with pagination, tokens and an example `smon-job` call |
| S5 | hosts CRUD; page: heartbeat, expectations (effective thresholds, override, deletion with confirmation), `/hosts/{id}/report` (last report + discovery), tokens, `last_reject`; no active token → a warning. S5.3: `/settings/export` (GET — file), `/settings/import` (§13.9) |

**Detail pages:**
- `/signal?key=…` (the key is in the query, not the path: it contains `/`, `:` and spaces): status, since, message, value,
  `last_update_at`, deadline (from `expected_signals`), parent, active silences, the signal's incidents, `status_history`
  paginated by 50. If the signal does not exist and is not expected — "removed from monitoring" and the history.
- `/incidents/{id}` (to find the `#id` from Telegram): incident fields, the signal history within opened..resolved,
  audit with target `incident:{id}`.
- Overview — the HTML version of the daily summary from the same `notify.summary.collect()`: a "Monitor" block (health and
  problems, uptime, dead-man, undelivered, age of the last sweep, version), counters, open incidents, problematic signals without an
  incident (unknown, summary-only, `self:*`), "awaiting data", silences, disabled objects.

### 12.6 List columns (guidance)

| List | Columns |
|---|---|
| Checks | status · name · type · target · message · latency · since · interval · enabled |
| Jobs | status · name · schedule · last run (ts, status) · expected by · message · enabled |
| Hosts | heartbeat · worst sub-signal and number of bad ones · name · os · report (age) · rejection (if newer than the report) · enabled |
| Incidents | # · severity · current status · title → signal · message · opened (ts + age) · ack · silence · Telegram (sent / pending / suppressed) · [closed · duration · reason] |
| Runs | run_id · start · finish · duration · status · exit · data_time · message |
| Tokens | kind · object · label · prefix · created · last used · revoked |
| Audit | at · actor · action · target · details |

**Sorting and pagination:** open incidents — no pagination, rank(severity) desc, then `opened_at` asc; closed/all incidents,
runs, audit, history — id desc, 50 per page; signals — rank desc, then since asc.

### 12.7 Forms and actions

- **CRUD:** `GET|POST /x/new`, `GET|POST /x/{id}/edit` (the same URL; an error → the same form with status 422 and messages
  at the fields), success → 303 to `/x/{id}`; `POST /x/{id}/delete`, `POST /x/{id}/toggle`. The slug field is editable only
  on creation; a check's `type` is chosen on `/checks/new?type=http` and never changes.
- **Nested routes:** host expectations — `POST /hosts/{id}/expectations` (manual addition: kind + subject + `expected_state`
  for vm), `GET|POST /hosts/{id}/expectations/{eid}/edit`, `POST …/{eid}/delete`; tokens — `POST /hosts/{id}/tokens`,
  `POST /jobs/{id}/tokens`, `POST /settings/tokens` (read), `POST /tokens/{tid}/revoke`. Discovery checkboxes on
  `/hosts/{id}/report` post several items at once to `POST /hosts/{id}/expectations`.
- **Import upload:** `POST /settings/import` body ≤ 1 MiB, otherwise 413.
- **Form parsing (`web/forms.py`):** a missing bool = false; an empty string for a nullable field = null; a non-nullable
  field is prefilled with its default, empty → 422. Schema defaults apply only in the API and import.
- **Durations** in forms — `fmt.parse_duration` ("30m", "4h", "2d", "90s"; a bare number = seconds); displayed with `fmt.duration`.
- **Confirmation** is mandatory for delete, disable, revoke, import, cancelling a silence and a silence with target `*`.
- **PRG with three exceptions** — a 200 response without a redirect and with `Cache-Control: no-store`: a token was created
  (shown once with a warning), the Telegram test result, the import result.
- **Settings:** widgets by registry type — IANA → a select from `zoneinfo.available_timezones()`; `list[int]` — a comma-separated
  string; `int|null` — empty = null; secret → a password field + "set / not set" + a "clear" checkbox. Pair validation
  `1 ≤ crit < warn ≤ 99` (for overrides — on the effective values). The route calls `settings.set_many`, writes the
  `settings.update` audit record and, after commit, `ctx.on_settings_changed(changes)`.
- **"Check now"** — `POST /checks/{id}/run` (enabled checks only): `runtime.probe_now.add(id)`, 303 to the check page; the result
  arrives through the normal path with debounce. Disabled in SAFE_MODE.
- **Telegram test** — `POST /settings/telegram/test`: uses the saved settings, `await` outside a tx; the answer is "sent" or
  `{status} {description}`; audit `telegram.test`. Disabled in SAFE_MODE.
- **Silences:** `/silences/new?target=` and buttons on object, signal and incident pages; target — `*` (with an extra
  confirmation) or a key parsed by `parse_key`; reason is mandatory; presets 1 h, 4 h, 1 day and "until…" (`datetime-local`
  in `general.timezone`; a time in the past → 422). The list shows active ones plus those ended within 7 days.
- **Discovery** (`/hosts/{id}/report`): one table per section (schema columns + "expected"). Checkboxes only **add**
  expectations; an expectation can be removed only from its own row. Defaults — §9.4. Empty state: "No reports yet" + the token.
- **Logs** (`/logs`): the same parameters as `GET /api/v1/logs`; filtering through htmx with the `_log_rows` fragment;
  N from 500 to 5000; "Download" (`/logs/download`) — the filtered result as `.jsonl` with `Content-Disposition: attachment`.
- **Password change** — CLI only (§16.5).

### 12.8 HTMX

HTMX is used for three things only:
- auto-refresh of the Overview and Incidents fragments (`hx-trigger="every 15s"`), keeping query parameters;
- inline Ack, Mute 1h and silence cancellation — the response is the row fragment;
- the log filter.

An htmx request without a session → 401 + header `HX-Redirect: /login` (otherwise the login page would be inserted into a fragment).
A non-htmx UI request without a valid session → 303 `/login`; a successful login → 303 `/` (no `next` parameter: no open
redirect).

---

## 13. Reliability

### 13.1 Supervisor and loops

- `Supervisor` runs every `LoopSpec` as a separate task and sets the contextvar `loop = name` in it:
  `await tick()` → sleep `period_s` → repeat. `Supervisor.run()` returns **only after `stop()`**, even with no loops (S0).
- **Expected external errors** (`TelegramError`, dead-man HTTP errors, `OSError` and sqlite errors of the snapshot and retention)
  are handled **inside** the tick, and the tick counts as successful. An exception escaping a tick means a bug or a failure
  of the main database: log `loop.error` (ERROR, with `exc`), sleep `period_s`, repeat. `CancelledError` propagates.
- After a **successful** tick the supervisor writes `runtime.loop_last_ok[name] = clock.monotonic()`; at startup — the start
  time, and `runtime.loop_stale_after` for loops with `stale_after_s`.
- A loop is stale if `now_mono − loop_last_ok[name] > loop_stale_after[name]`. Age uses the monotonic clock (D20).
- `supervisor.stop()` cancels the loop tasks and calls their `on_stop` (cancelling probe tasks).

| Loop | period_s | stale_after_s | What it does | Tick duration bound |
|---|---|---|---|---|
| probe | 5 | 60 | schedules probes (§7.5); probes are separate tasks, semaphore 10 | instant |
| sweep | 15 | 60 | §13.2 | < 1 s |
| sender | 5 | 120 | notifications and daily summary (§10) | send ≤ 10 s per part; after the first error the tick stops sending |
| poller | 1 (inside a 25-s long poll) | 120 | Telegram updates (§10.5); without Telegram — an empty tick | ≤ 35 s + a 30-s pause on errors |
| maintenance | 60 | 7200 | snapshot (§13.5), retention (§13.8) — through `ctx.run_blocking`, awaited | a snapshot of a DB up to 1 GB |
| deadman | 60 | — | ping or fail (§13.3); the first ping only after 120 s of uptime | ≤ 10 s |

### 13.2 Sweep (`loops/sweep.py`)

```text
sweep_tick:
  now    = clock.now()
  disk   = ctx.disk_usage(data_dir)                        # before tx(): synchronous statvfs, fixed in tests
  newest = ops.snapshots.newest_at(backups_dir)            # before tx(): by file name
  with tx() as s:                                          # no await inside
    st       = gather SelfState (runtime; settings: instance_created_at, ping_url and settings.updated_at of it;
                                 disk; newest)
    expected = catalog.expected_signals(s, now)                            # §6.6
    engine.reconcile(s, expected.keys(), now)              # first: disabled objects get no no-data incident
    ups  = evaluate_deadlines(expected, engine.signal_states(s), now, ctx.process_started_at, ctx.down_from)
    ups += [evaluate_job(j, catalog.jobs.runs_view(s, j.id), now, "sweep") for j in catalog.jobs.enabled(s)]
    ups += evaluate_self(st, now)
    engine.apply(s, [u for u in ups if u.key in expected], now)
    engine.open_no_data(s, now)
    settings.set(s, "internal.sweep_at", now)             # mandatory write: a successful sweep ⇒ the DB is writable
```

**Self-signals (`evaluate_self`)** — only those in `IMPLEMENTED_SELF` (S1: telegram; from S3: all). Messages contain no current
time or age, so a repeated sweep writes nothing:

| Signal | Condition | Status | Message |
|---|---|---|---|
| `self:telegram` (summary) | `now − undelivered_since > 600` or `tg_poll_error` is set | critical | `undelivered for more than 10 min: {tg_send_error}` \| `poller: {tg_poll_error}`; ok — `messages are delivered` |
| `self:snapshot` | `now − max(newest_snapshot_at, instance_created_at) > 26 h` | critical | `no snapshot for more than 26 h[: {snapshot_last_error}]`; ok — `latest snapshot {YYYY-MM-DD HH:MM} UTC` |
| `self:disk` | `/data` free < 5 % or < 1 GiB → critical; < 10 % or < 2 GiB → warning; `ctx.disk_usage` failed → unknown | warning / critical | `{pct:.1f}% free ({free} of {total})` |
| `self:deadman` | only with `deadman_configured`: `now − max(deadman_last_ok_at ?? 0, process_started_at, deadman_url_set_at) > 900` | warning | `no successful ping for more than 15 min[: {deadman_last_error}]`; ok — `pings succeed` |

### 13.3 Health and dead-man

- **Healthy** = not SAFE_MODE, no stale loops (§13.1) and `now − undelivered_since ≤ 600` (or `undelivered_since` unset).
  Computed by `domain.health.evaluate(runtime, safe_mode, now, now_mono)` from memory, without database writes.
  Problems are short strings: `loop sweep stale 75s`, `telegram undelivered 12 min`, `safe_mode`.
- `/healthz` on :8081: 200 `{"healthy":true}` or 503 `{"healthy":false,"problems":[…]}`. On :8080 — only `{"healthy":bool}`.
- **Dead-man** every 60 s, **starting only after 120 s of process uptime** (a process that crashes and restarts in a loop never
  pings, so the external service alerts):
  - healthy → `GET {ping_url}`; otherwise → `POST {ping_url}/fail`, body — problems joined with `\n`;
  - timeout 10 s; 2xx updates `runtime.deadman_last_ok_at` and clears `deadman_last_error`; otherwise
    `deadman_last_error` = the HTTP code or the exception class name (**never the URL**: it contains a secret);
  - **only** the deadman loop in the same event loop pings, using health computed at the moment of the ping. Pinging from
    another thread, process or cron is forbidden: with a hung event loop it would be a silent false OK. A fully blocked event
    loop, a deadlock or a hung process is detected only by the absence of pings (and the Docker healthcheck);
  - `deadman.ping_url` not set → the loop does nothing; the UI and the daily summary say "dead-man not configured";
  - in SAFE_MODE there are deliberately no pings: the external service reports the outage after its grace;
  - snapshot and disk problems go to Telegram only, not to `/fail`.
- The §2.2 target "≈ 7 min" holds with the external check configured as period 1 min, grace 5 min (`docs/operations.md`).

### 13.4 Process and shutdown

- Startup order — §16.4. `main` waits for two tasks — `supervisor.run()` and `server.serve()` — with `FIRST_COMPLETED`
  (in SAFE_MODE there is no supervisor and `main` waits for the server only). If either finishes without a stop request:
  log `process.failed` (ERROR), stop the other one, exit 1 (Docker restarts the container).
- **Signals:** uvicorn ≥ 0.29 installs its own handlers around `Server.serve()` (`capture_signals`) and re-raises the signal
  after serving. smon uses a `uvicorn.Server` subclass whose `capture_signals()` returns `contextlib.nullcontext()`. As the
  **first action** `main` installs `signal.signal(SIGTERM|SIGINT, set_flag)` (works while synchronous migrations block the loop);
  the flag is checked between startup steps (exit 0 when set). Right before `serve()` the handlers are switched to
  `loop.add_signal_handler`, which is idempotent: `server.should_exit = True` and `supervisor.stop()`. The exact uvicorn version
  is pinned (D<n>) and covered by a test.
- Shutdown: `timeout_graceful_shutdown=5`; probes and loops are cancelled without waiting. A snapshot or retention run in
  progress is not interrupted; Docker's SIGKILL after `stop_grace_period` ends it, which is safe (below).
- **SIGKILL at any point is safe:** WAL + `synchronous=FULL`, atomic rename of snapshots, transactional migrations,
  at-least-once delivery (a duplicate message is acceptable). A graceful shutdown is an optimization, not a correctness condition.
- uvicorn settings: `lifespan="off"`, `access_log=False`, `log_config=None`, `proxy_headers=False`, no `[standard]`
  (asyncio + h11). One server, `serve(sockets=[s_ingest, s_admin])`, the ASGI dispatcher `web/dispatch.py` by
  `scope["server"][1]`. Background tasks are never started from sub-app lifespans.

### 13.5 Snapshots (`ops/snapshots.py`)

Everything runs in one `ctx.run_blocking` call on a **separate autocommit sqlite3 connection**, outside `tx()`:
1. Delete all `.tmp-*` in `/data/backups` (the flock rules out a parallel snapshot).
2. Check space: free ≥ 2 × size(DB) + 1 GiB, otherwise `snapshot.failed` "not enough space" without an attempt.
3. `VACUUM INTO /data/backups/.tmp-<stamp>.sqlite` (its own read transaction; does not block WAL writers).
4. `PRAGMA integrity_check` on the copy == `ok`.
5. gzip into `.tmp-<stamp>.sqlite.gz` **as a stream** (`shutil.copyfileobj`, 1-MiB blocks — never read the file whole), fsync;
   delete the uncompressed copy right away.
6. rename to `smon-<YYYYMMDDTHHMMSSZ>-{daily|premigrate}.sqlite.gz` (UTC stamp). The file's existence means it is valid.
7. `finally` — delete the tmp files of this attempt.

- **Daily** — when `local_now ≥ snapshot.time` and there is no daily snapshot for `local_today` yet (the stamp is converted
  to `general.timezone`).
- **Premigrate** — before applying a new migration (from S3, §5.3).
- `ops.snapshots.take()` raises on failure. The maintenance tick owns the runtime fields: on success
  `snapshot_last_error = None`; on `OSError` / `sqlite3.Error` — `snapshot_last_error = <error text>`,
  `snapshot_retry_at = now + 3600` (no retry for an hour) and `snapshot.failed` (ERROR). A premigrate failure → §5.3.
- **Rotation:** the `snapshot.keep_daily` newest daily snapshots; premigrate — the 3 newest.
- `newest_snapshot_at` is taken from the name stamp (any kind), not from mtime.
- Off-host copies of snapshots are made by the host (not the application) and stored encrypted only.

### 13.6 SAFE_MODE (`SMON_SAFE_MODE=1`)

- The supervisor does not start: no probes, sweep, sending, poller, dead-man, snapshots or retention. No outgoing connections.
- Ingest answers 503 `safe_mode`.
- The UI works fully, including configuration edits. Buttons with outgoing connections or loop effects are disabled:
  Telegram test, "check now".
- `/healthz` → 503 `["safe_mode"]`; a UI banner.
- Purpose: restore drill, snapshot inspection, emergency configuration edits.

### 13.7 Restore (normative procedure, `docs/restore.md`)

1. Stop the container.
2. Delete `smon.db`, `smon.db-wal`, `smon.db-shm` (a foreign WAL would corrupt the restored database).
3. `gunzip` the chosen snapshot into `smon.db`. The image must not be older than the snapshot's schema.
4. Start with `SMON_SAFE_MODE=1`, check the configuration, incidents and tokens.
5. Start normally.

- **A drill copy** runs only in SAFE_MODE. To run a copy normally, first clear `deadman.ping_url` and `telegram.bot_token`
  in SAFE_MODE: otherwise the second instance pings the same dead-man and masks the death of the primary.
- No runtime state reset is needed: a rolled-back `tg_offset` is harmless; announce flags cause acceptable duplicates;
  a second daily summary on the same day is possible; old sessions are harmless.
- Known loss: job events after the snapshot. A job that failed after the snapshot looks ok until its next slot; the runbook
  says to compare open incidents with the Telegram history.

### 13.8 Retention (`ops/retention.py`)

- Once a day at 04:30 in `general.timezone`, through `ctx.run_blocking`, in batches of 5000 rows with a commit per batch.
  `ops.retention.run` writes `internal.retention_date = <local today>` in its first transaction, before deleting anything; a
  failed run is logged and retried the next day.
- Deleted:
  - older than `retention.history_days`: `status_history` (by `at`), closed `incidents` (by `resolved_at`), `audit_log`,
    `job_runs` (by `created_at`, **except the last 50 runs of each job**), silences (ended or cancelled);
  - `sessions` with `expires_at < now`;
  - `tg_messages` older than 7 days.
- `signals` and open incidents are never touched. Result — `retention.done` with the number of deleted rows per table;
  an error — `retention.failed`.

### 13.9 Configuration export and import (`catalog/export.py`, `schemas/export_v1.py`)

- **Export** (UI, a JSON file), serialized only through `schemas/export_v1` (`model_dump(mode="json")`):

  ```json
  {"format":"smon-export","version":1,"settings":{…},"checks":[…],
   "hosts":[{…,"expectations":[…]}],"jobs":[…],"tokens":[…]}
  ```

  - settings — all exported registry keys with their effective values (through registry types, so that 15 and 15.0 do not differ);
  - only fields owned by `catalog` (§3.4); no timestamps (`created_at`, `enabled_at`, `last_report*`, `last_reject*`,
    `os_family`) and no secrets;
  - **tokens (D47):** active tokens — `kind`, object (`host_id`/`job_id`), `label`, `prefix`, `sha256`; revoked tokens are
    not exported. The sha256 of a random 256-bit token grants no access, so the file is not a secret;
  - sorting: objects by id, expectations by (kind, subject), tokens by (kind, object, sha256);
    `json.dumps(sort_keys=True, ensure_ascii=False, indent=2)` + `\n`;
  - export and import are implemented once, for all sections, after all configuration objects exist (checkpoint S5.3).
- **Import:**
  - only if the checks, hosts, jobs and tokens tables are empty, otherwise 409 `conflict`;
  - the file is validated as a whole (`extra="forbid"`), then applied in one transaction;
  - `created_at`, `updated_at`, `enabled_at`, `schedule_changed_at` = now; `last_report*`, `last_reject*`, `os_family` = NULL;
  - tokens are restored with `created_at = now` — collectors and wrappers keep working without new tokens;
  - settings go through `settings.set_many`; afterwards — the `config.import` audit record and `ctx.on_settings_changed`.
- Criterion: export → import into an empty instance → a second export is byte-identical to the first.

---

## 14. Logging

### 14.1 Setup

- **File:** `/data/logs/smon.log`, JSON lines, `encoding="utf-8"`. Duplicated to stdout when `SMON_LOG_STDOUT=1` (default).
  Fatal startup errors always go to stderr.
- **Order (§16.4):** before the flock is acquired and in the CLI, logs go to stderr only (two processes with a
  `RotatingFileHandler` on one file break rotation). The file handler is added after the flock with the registry defaults;
  after migrations `apply_log_settings` reads `logs.*` from the database.
- **Loggers:** handlers are attached to root, the root level is WARNING. `logs.level` applies only to the `smon` logger.
  `httpx` and `httpcore` are pinned to WARNING regardless of `logs.level` (at INFO they log URLs, and a Bot API URL contains
  the token). `uvicorn.*` loggers go through the same JSON formatter (`log_config=None`).
- **Alembic:** `env.py` does not call `fileConfig` (otherwise the `smon.*` loggers are disabled, §5.3).
- **Rotation:** `RotatingFileHandler`, `maxBytes = logs.max_total_mb · 1024² // 5` (MiB), `backupCount = 4`. `apply_log_settings`
  changes `maxBytes` and the level of the live handler (effective from the next write, under the handler lock — threads from
  `ctx.run_blocking` are safe). After a limit decrease the total converges to the limit within at most five rotations. It is called at
  startup, when `logs.*` are saved and after an import (the `on_settings_changed` hook, §2.3).

### 14.2 Line format

- `json.dumps(ensure_ascii=False, separators=(",", ":"))`.
- `ts` — `YYYY-MM-DDTHH:MM:SS.mmmZ` from the OS clock (not from Clock).
- `msg` — English. Human-readable signal text goes into the `detail` field.
- **Closed field list** (`log.FIELDS`); an unknown key in `extra` is dropped, and `"dropped":[names]` is written into the line
  (an autouse test fixture fails on `dropped` and on an `event` that does not match `^[a-z_]+\.[a-z_]+$`):
  - `ts`, `level`, `logger`, `event`, `msg`, `exc` (formatException);
  - `request_id`, `loop` — taken by the formatter from contextvars;
  - `signal_key`, `incident_id`, `check_id`, `host_id`, `job_id`, `run_id`, `silence_id`, `token_prefix`, `message_id`,
    `update_id`, `tg_user_id`, `client_ip`;
  - `method`, `path`, `status_code`, `code`, `duration_ms`;
  - `old_status`, `new_status`, `severity`, `origin`, `kind`, `result`, `reason`, `detail`, `error`, `attempt`, `delay_s`, `count`;
  - `actor`, `action`, `target`, `version`, `config`, `revision`, `file`, `size_bytes`, `deleted`, `problems`, `pid`, `safe_mode`.
- `LogRecord` attribute names (`message`, `name`, `args`, `filename`, `module`, `process`…) are forbidden in `extra`
  (they raise `KeyError` in the caller).
- Lines from third-party libraries have no `event` field.
- **contextvars:** `request_id` is set by `web/dispatch.py`; `loop` is set by the supervisor in a loop task; `create_task` and
  `asyncio.to_thread` (behind `ctx.run_blocking`) copy the context, so probes and snapshots inherit `loop`.

### 14.3 Levels

- **DEBUG** — debugging details (probe results, debounce and suppression decisions, htmx GET polling).
- **INFO** — normal events.
- **WARNING** — a degradation the code handled (Telegram error, failed login, a foreign Telegram user).
- **ERROR** — an exception or a lost function (a failed tick, a 500, a snapshot failure).

### 14.4 Event catalogue

The catalogue is informative (D79): an implementer may add an event by adding a row to this table in the same PR (no new decision needed);
the name follows `<area>.<action>` and uses only fields from `log.FIELDS`.

| event | Level | Fields |
|---|---|---|
| `process.started` | I | version, pid, safe_mode, config (RuntimeConfig without secrets), revision |
| `process.stopping`, `process.stopped` | I | — |
| `process.failed` | E | exc (migration failure, bind failure, unexpected task exit) |
| `migration.applied` | I | revision |
| `settings.ignored` | W | target (unknown key read from the database) |
| `http.request` | D for GET 2xx/3xx on :8081, otherwise I | method, path, status_code, duration_ms, client_ip |
| `http.error` | E | exc (500) |
| `ingest.accepted` | I | host_id \| job_id, run_id?, result (accepted \| duplicate), status_code, duration_ms, token_prefix |
| `ingest.rejected` | I; 404/405 on :8080 — D | status_code, code, error, client_ip; the object id — with a valid token |
| `signal.status_changed` | I | signal_key, old_status, new_status, origin, detail |
| `signal.debounce` | D | signal_key, kind, count |
| `signal.removed` | I | signal_key (reconcile) |
| `incident.opened`, `.escalated`, `.resolved`, `.no_data_opened` | I | incident_id, signal_key, severity; resolved — also reason |
| `notify.sent` | I | one line per Item after `mark_sent`: incident_id, signal_key, kind, severity, message_id; test — `kind=test`; summary — `kind=summary` |
| `notify.failed` | W | kind (incl. `callback`, `status`), status_code?, error, attempt, delay_s |
| `notify.suppressed` | D | signal_key, reason (silence \| parent) |
| `tg.callback` | I | update_id, tg_user_id, action, count |
| `tg.command` | I | update_id, tg_user_id (`/status`) |
| `tg.rejected_user` | W | update_id, tg_user_id |
| `tg.poll_error` | W | status_code?, error |
| `probe.result` | D | check_id, result, duration_ms, error |
| `loop.started`, `loop.stopped` | I | loop |
| `loop.error` | E | loop, exc |
| `deadman.ping` | D | status_code |
| `deadman.fail` | W | problems |
| `deadman.error` | W | error (no URL) |
| `snapshot.done` | I | kind (daily \| premigrate), file, size_bytes, duration_ms |
| `snapshot.failed` | E | error |
| `retention.done` | I | deleted {table: n}, duration_ms |
| `retention.failed` | E | exc |
| `admin.login_failed` | W | client_ip |
| `audit` | I | actor, action, target (written by `audit.write()`; details are not logged) |

Admin actions (login, logout, configuration changes, ack, mute, silence, tokens, import) are logged as a single `audit`
event — there are no separate events for them.

### 14.5 Never logged

Tokens, passwords, secrets (`telegram.bot_token`, `deadman.ping_url`), session ids and cookies, CSRF tokens, the
`Authorization` header, Bot API URLs, bodies of target responses, texts of incoming Telegram messages. From ingest bodies —
only validation errors (field + reason, at most 10).

Protection:
- `make_engine(..., hide_parameters=True)` — SQL parameters never reach exception texts;
- `TelegramError.description` and dead-man errors contain no URL (§4.5, §13.3);
- a safety redaction in the formatter: `bot\d+:[\w-]{30,}` and `smon_[\w-]{20,}` in `msg`, `error`, `detail` and `exc` are replaced with `***`;
- test: after a send through `respx` and a dead-man error, the log file contains neither the token nor the URL.

### 14.6 Access

- The UI "Logs" page and `GET /api/v1/logs` use one function, `ops.logs_reader` (contract — §11.3).
- The reader first opens all files (`smon.log`, `.1`…`.4`, newest to oldest), then reads: on POSIX open descriptors survive
  the rename done by rotation. A missing file (`FileNotFoundError`) is skipped.
- The files are accessible through `docker exec`.

---

## 15. Security

The level is "sufficient, not paranoid". Not done: a pepper for tokens, encryption of secrets in the database, binding a
session to an IP or user agent, CSP nonces, HSTS, CORS, password complexity rules, account lockout.

### 15.1 Ingest and read tokens

- Format: `smon_` + `secrets.token_urlsafe(32)`; `prefix` — the first 12 characters; the database stores the sha256
  (lowercase hex of the full token including `smon_`). An unsalted sha256 is correct for 256-bit random tokens.
- Lookup only through the `sha256` index (`catalog.tokens.authenticate`); a linear scan with `compare_digest` is forbidden.
- Shown once; revocation (`revoked_at`) is effective immediately, no cache. Several active tokens per object are allowed
  (rotation: create → roll out → revoke the old one).
- Scope: host, job or read. Host and job tokens are created on the object's page, read tokens in Settings.
- The export contains the hashes of active tokens (§13.9).

### 15.2 Admin UI (:8081)

- **Access is expected over a VPN only** (e.g. Tailscale) — a deployment duty. The second line of defence is the allowlist
  `SMON_ADMIN_ALLOW` (comma-separated CIDRs; default `127.0.0.0/8, ::1/128, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16,
  100.64.0.0/10, fc00::/7`; `*` disables the check). A client IP (§15.4) outside the list → 403 `admin_not_allowed` on any
  :8081 path except `/healthz`. Limitation: with the Docker userland proxy the source is the Docker gateway address (in the
  list); the main protection is publishing the port on `127.0.0.1` only (§16.6) and checking "connection refused from outside"
  during acceptance.
- **Password:** 12..1024 characters; argon2id `PasswordHasher.from_parameters(RFC_9106_LOW_MEMORY)` (t=3, m=64 MiB, p=4);
  after a successful login `check_needs_rehash` and rehashing. Verification runs on the event loop (~50 ms).
  `smon admin set-password` reads the password from a TTY (twice, compared) or with `--stdin` (until EOF, one trailing `\n`
  or `\r\n` is stripped), never from argv. There is no password change in the UI.
- **Sessions:**
  - cookie `smon_session` = `secrets.token_urlsafe(32)`; the database stores its **sha256** (`sessions.id_hash`),
    so a leaked snapshot does not yield a live session;
  - every login creates a new session (fixation protection); expired sessions are deleted on login and by retention;
  - a fixed lifetime: `expires_at = created_at + 14 days`, no extension (log in again after two weeks);
  - cookie: HttpOnly, SameSite=Strict, Secure when `SMON_ADMIN_HTTPS=1`, Path=/;
  - `POST /logout` (with CSRF): delete the row, clear the cookie, audit;
  - `smon admin set-password` deletes all sessions in the same transaction ("log out everywhere").
- **Login limiter** (in the admin app's `app.state`): after ≥ 5 failures in 15 min from one IP the password is not checked —
  429 + `Retry-After` and an HTML "try again in N min". A successful login resets the IP counter; entries older than 15 min are
  purged. Without `SMON_TRUSTED_PROXIES` all clients behind a proxy share one IP — the limit becomes global (acceptable for a
  single admin).
- **CSRF:** for every non-GET on :8081 — `X-CSRF-Token` (htmx, via `hx-headers` on `<body>`) or the `csrf_token` field (forms);
  the token is `sessions.csrf_token`; comparison with `hmac.compare_digest`; the check is a dependency of the admin router (not an
  ASGI middleware: otherwise the form body is read twice); failure → 403 `csrf_failed` (HTML). `POST /login` is exempt: with a single account, login CSRF cannot log a victim into
  another account (D89, supersedes D22); the SameSite=Strict session cookie still protects everything after login.
- **Response headers on :8081:**
  - `Content-Security-Policy: default-src 'self'; img-src 'self' data:; object-src 'none'; base-uri 'none'; form-action 'self'; frame-ancestors 'none'`;
  - `X-Content-Type-Options: nosniff`; `Referrer-Policy: same-origin`;
  - `Cache-Control: no-store` everywhere except `/static`.
- **Templates:** autoescape is mandatory, `|safe` on data is forbidden — service names and errors come from the public ingest.

### 15.3 Secrets

- `telegram.bot_token` and `deadman.ping_url` are stored in the database in plain text, write-only in the UI ("set / not set"
  + "clear"), never shown in the UI, the API, logs (§14.5) or exports.
- Deployment requirement: off-host copies of `/data` and snapshots are encrypted.
- Files in `/data` are created with umask 077 (§16.2).

### 15.4 Client IP

`domain.netaddr.client_ip(peer, xff, trusted)` is the only way (uvicorn with `proxy_headers=False`): a peer outside
`SMON_TRUSTED_PROXIES` → the peer; otherwise `X-Forwarded-For` from right to left, the first address not in trusted; all
trusted → the leftmost; garbage in the header → the peer. Used by the login limiter, the admin allowlist and logs.

### 15.5 Audit

Written by the module that owns the change (§3.4); `audit.write()` also writes the `audit` log event. A failed login is not
audited (only the `admin.login_failed` log event). `target` has the form `<kind>:<id>` (`check:web`, `incident:42`,
`token:7`, `settings:telegram`); actions on several incidents write one record per incident. The `action` vocabulary:

| action | actor | details |
|---|---|---|
| `check.create\|update\|delete\|toggle`, `host.*`, `job.*`, `expectation.create\|update\|delete` | admin | names of the changed fields |
| `token.create`, `token.revoke` | admin | kind, object, prefix |
| `settings.update` | admin | names of the changed keys (secrets as `"changed"` / `"cleared"`) |
| `incident.ack`, `incident.mute` | admin \| `tg:<user_id>` | duration (mute) |
| `silence.create`, `silence.cancel` | admin | target, ends_at, reason |
| `config.import`, `config.export` | admin | object counts |
| `telegram.test` | admin | result |
| `admin.login`, `admin.logout` | admin | client_ip |
| `admin.password_set` | cli | — |

---

## 16. Runtime contract

### 16.1 Image

- `ghcr.io/<owner>/smon:<X.Y.Z>` — a **public** package. Tags: `X.Y.Z` (release) and `sha-<7>` (every push to `main`);
  no `latest` (migrations are irreversible; upgrades must be deliberate). Architectures: amd64 and arm64 for releases;
  `sha-<7>` images are amd64 only.
- Multi-stage on `python:3.12-slim-trixie@sha256:…` (both stages on the same digest):
  - builder: `uv` copied from `ghcr.io/astral-sh/uv:<version>@sha256:…`, `uv sync --frozen --no-dev --no-editable`, `UV_COMPILE_BYTECODE=1`
    (otherwise, with `PYTHONDONTWRITEBYTECODE`, every start and healthcheck recompiles the modules);
  - runtime: `ENV PATH=/app/.venv/bin:$PATH PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1`; `USER 10001:10001` (numeric);
    `RUN mkdir /data && chown 10001:10001 /data && chmod 700 /data`; no `VOLUME` instruction (an anonymous volume would hide a
    forgotten mount); `uv run` is forbidden at runtime (it writes a cache to the read-only file system).
- `CMD ["smon", "serve"]`, no ENTRYPOINT: so that `docker run … smon db upgrade` and `docker exec … smon admin set-password` work.
- `EXPOSE 8080` only — ingest (platforms that publish EXPOSEd ports will not open the admin UI).
- **HEALTHCHECK — liveness:** `--interval=30s --timeout=5s --start-period=120s CMD ["python","-c", "<urllib: GET http://127.0.0.1:$SMON_ADMIN_PORT/healthz, 200 or 503 within 5 s → exit 0>"]`.
  Full health (Telegram, stale loops, SAFE_MODE) is watched by the dead-man and the UI: otherwise platforms would take the
  container out of routing or restart it because of Telegram. `start-period` covers the premigrate snapshot and migrations.
- Compatible with a read-only root file system; writes only to `/data` and `/tmp` (a size-limited tmpfs).
- `TZ` is not needed: time is UTC epoch, display goes through `general.timezone`.

### 16.2 Volume

- `/data` — the database `smon.db`, `backups/`, `logs/`, `.lock`. Local file systems only: SQLite WAL and flock are unreliable
  on NFS/SMB. For development on Docker Desktop use a named volume, not a bind mount.
- The process always runs as uid 10001 (the code is not tied to the uid; it only needs write access). At startup
  `open(.lock, O_CREAT)`; `PermissionError` → exit 2 `cannot write {dir} as uid {uid}: chown -R 10001:10001 <host dir>`.
- umask 077: the database holds `telegram.bot_token` in plain text.
- The flock is held for the whole process lifetime; `.lock` is never deleted.

### 16.3 Env

| Variable | Default | Purpose and validation |
|---|---|---|
| `SMON_DATA_DIR` | `/data` | absolute path |
| `SMON_INGEST_PORT` | `8080` | 1..65535 |
| `SMON_ADMIN_PORT` | `8081` | 1..65535, different from ingest |
| `SMON_TRUSTED_PROXIES` | `""` | comma-separated IPs or CIDRs (`ipaddress.ip_network(strict=False)`) |
| `SMON_ADMIN_ALLOW` | loopback, RFC 1918, 100.64.0.0/10, fc00::/7 | comma-separated CIDRs or `*` (§15.2) |
| `SMON_SAFE_MODE` | `0` | `0` \| `1` (§13.6) |
| `SMON_ADMIN_HTTPS` | `1` | `0` \| `1`; the Secure cookie flag (0 — local development only) |
| `SMON_LOG_STDOUT` | `1` | `0` \| `1` |
| `SMON_TELEGRAM_API_BASE` | `https://api.telegram.org` | http(s) URL without a trailing `/`; for the fake Telegram in tests and development |

- `config.py` validates everything at the start of **any** command. An unknown `SMON_*` variable is an error (exceptions:
  `SMON_SERVICE_*`, `SMON_PORT*` — Kubernetes service links): a typo such as `SMON_SAFEMODE=1` during a restore drill would
  otherwise start the copy with the real dead-man and Telegram.
- Error: one line `config error: SMON_X: <reason>` on stderr, exit 2.

### 16.4 Startup order (`main.serve`)

1. `signal.signal(SIGTERM|SIGINT, set_flag)`: until the server starts the handler only sets a flag (§13.4).
2. env → `RuntimeConfig`.
3. Logs to stderr only, level INFO.
4. Write check of `SMON_DATA_DIR` (§16.2), umask 077.
5. flock `{SMON_DATA_DIR}/.lock` non-blocking; taken → stderr `another smon instance holds {path}`, exit 1.
6. mkdir `logs/`, `backups/`.
7. The log file handler with the registry defaults (§14.1).
8. `main.upgrade()`: premigrate snapshot (§5.3, from S3) → migrations (FK OFF engine, one transaction) → `settings.init_internal`.
9. `apply_log_settings` from the database; `process_started_at = clock.now()` is taken here, after migrations (the downtime
   shift of §6.6 covers the migration time; uptime for the dead-man delay, the summary and `/api/v1/status` counts from this
   moment); `ctx.down_from = internal.sweep_at ?? process_started_at` (§6.6).
10. `AppContext`, `Supervisor` (not in SAFE_MODE), the uvicorn server subclass (§13.4); the signal handlers switch to
    `loop.add_signal_handler`; `serve(sockets=[…])`; waiting per §13.4.

### 16.5 CLI

| Command | Behaviour |
|---|---|
| `smon serve` | §16.4 |
| `smon db upgrade` | takes the flock non-blocking (taken → exit 1: a running serve migrates by itself); creates the database if missing; runs `main.upgrade()` |
| `smon admin set-password [--stdin]` | does not take the flock; no database or revision ≠ head → exit 1 `database not initialized or not at head: start smon serve first`; the password per §15.2; deletes all `sessions`; audit `admin.password_set` (actor `cli`) |

- Exit codes: 0 — success or a normal stop; 1 — lock taken, database not ready or newer than the code, a failure; 2 — a
  configuration or argument error, no write access to `/data`, or a password shorter than 12 characters.
- The CLI writes to stderr only (not to `smon.log`).

### 16.6 Ports and publishing

- Both sockets listen on `0.0.0.0` inside the container. The admin UI is isolated by port publishing: `127.0.0.1:8081:8081`
  (and `127.0.0.1:8080:8080` if the proxy runs on the host), or no `ports` at all if the proxy is in the same Docker network.
  Docker bypasses UFW: `"8081:8081"` would expose the admin UI to the internet. The second line is `SMON_ADMIN_ALLOW` (§15.2).
- Ingest (:8080) is published to the internet through a reverse proxy with TLS; admin (:8081) only into the VPN (e.g.
  `tailscale serve`).
- `SMON_TRUSTED_PROXIES` = the proxy address or the compose network subnet (the subnet is fixed via `ipam`); otherwise all
  clients behind the proxy share one IP.

**`deploy/docker-compose.example.yml` (normative contents):**
- `image: ghcr.io/<owner>/smon:X.Y.Z`, `restart: unless-stopped`;
- named volume `smon-data:/data` (a bind mount only in a comment, with `chown 10001:10001`);
- ports on `127.0.0.1`;
- `read_only: true`, `tmpfs: ["/tmp:size=64m"]`, `cap_drop: [ALL]`, `security_opt: ["no-new-privileges:true"]`;
- `logging: {driver: json-file, options: {max-size: 10m, max-file: "3"}}`;
- `stop_grace_period: 30s`;
- `SMON_TRUSTED_PROXIES` and a fixed network subnet.
The healthcheck comes from the image (do not duplicate it in compose). `init` and `TZ` are not needed.

### 16.7 CI and release (GitHub Actions)

- **`ci.yml`**, minimal `permissions`. Triggers: `push` to `main`; `pull_request` with
  `types: [opened, synchronize, reopened, ready_for_review]`; `workflow_call` (from `release.yml`). Jobs:
  1. `ruff check`, `ruff format --check`, `mypy`, `lint-imports`, `pytest` (including the schema test; `tests/collectors` and
     `tests/wrappers/windows` are excluded by `addopts = "--ignore=tests/collectors --ignore=tests/wrappers/windows"` in
     `[tool.pytest.ini_options]` and run in their own jobs);
  2. collector tests under Python 3.8 (from S5.2, §17.1);
  3. `docker build` and an image smoke test on amd64:
     start with `--read-only --tmpfs /tmp` on a named volume and wait for healthy; `smon admin set-password --stdin`;
     `:8081/login` → 200, `:8080/` → 404 with JSON; a second container on the same volume → exit 1; `docker stop` < 10 s, exit 0;
  4. **`windows`** (from S4.2) on `windows-latest`:
     `uv run --no-project --with pytest python -m pytest -c tests/wrappers/windows/pytest.ini --confcutdir=tests/wrappers/windows tests/wrappers/windows`
     (scenarios run `powershell.exe -NoProfile -File wrappers/smon-job.ps1` against a stdlib HTTP server; the real Windows
     PowerShell 5.1, not pwsh); PSScriptAnalyzer for `wrappers/*.ps1` and, from S6.2, `collectors/windows/*.ps1`;
  5. on push to `main` only: publish `ghcr.io/<owner>/smon:sha-<7>` (amd64; `packages: write` only in this job).

  Required jobs are never skipped with an `if:` condition on the event (a skipped required check counts as passed).
- **`guard.yml`** — a separate workflow on `pull_request` with `types: [opened, synchronize, reopened, edited]` (`edited`
  re-runs only this check after a PR body edit). It runs `devtools/guard.py` **from the base branch**
  (`git show origin/${BASE_REF}:devtools/guard.py`), so a PR never changes the check that judges it; if the base has no
  `guard.py` yet (S0.1), the job passes. It looks at **modified or deleted** files only (added files never trigger it):
  `docs/SPEC.md`, `.importlinter`, the parsed `tool.ruff`, `tool.mypy` and `tool.pytest` tables and the dependency lists
  (`project.dependencies`, `dependency-groups`) of `pyproject.toml` (base vs head), every `ruff.toml` and `pytest.ini`,
  `tests/golden/`, `devtools/guard.py`, a migration file contained in any `v*` tag, `.github/`. If any is touched, the PR body
  must contain a line `Spec: D<n> …` (regex `^Spec: D\d+\b`, the D record approves the change) or `Planned: S<x>[.<n>[.<m>]]`
  (regex `^Planned: S\d+(\.\d+){0,2}\b`, the change is listed in that ROADMAP stage or checkpoint). This is not a ban but a
  loud signal for the process; the reviewer checks that the reference is true.
- **`.github/pull_request_template.md`** (S0.1): `Checkpoint: Sx.n`; `Spec: D<n> …` / `Planned: Sx.n` lines; what was tested;
  remaining deviations; `Review round <n>: …` lines added by the orchestrator.
- **`release.yml`** (tag `v*`): the same checks (`workflow_call`); the tag must equal `v{project.version}` (otherwise `version`
  in `/api/v1/status` lies) and the `VERSION` constant of every collector and wrapper script; `docker/metadata-action` for OCI
  labels with `type=semver,pattern={{version}}` only (no `sha-` tags from releases, no `latest`); buildx amd64 + arm64 (QEMU);
  `provenance: false`, `sbom: false`. Permissions per job, nothing at workflow level: `packages: write` in the image job;
  `contents: write` only in the release-assets job (from S5.2). Release assets: `smon-collectors-X.Y.Z.tar.gz`
  (`collectors/linux/*`, `collectors/windows/*` from S6.2, `wrappers/*`) and `SHA256SUMS` covering it. The scripts carry
  `VERSION = "X.Y.Z"` (sent as `collector_version`), bumped together with `project.version` in the closing PR of a stage.
- **Supply chain:** every action is pinned to a full commit SHA with the version in a comment
  (`uses: docker/build-push-action@<sha> # v6.x`); `actions/checkout` uses `persist-credentials: false`; workflows never
  interpolate `${{ github.event.* }}` text into `run:` — `devtools/guard.py` reads the PR body from `$GITHUB_EVENT_PATH`.
- No `pull_request_target`. Dependabot is enabled after v1.0.

---

## 17. Collectors and wrappers

### 17.1 File layout

```text
collectors/linux/smon-collector.py        python3 ≥ 3.8, stdlib only
collectors/linux/smon-collector.service   Type=oneshot, User=root, TimeoutStartSec=120; no ProtectSystem/ProtectHome/PrivateTmp
collectors/linux/smon-collector.timer     OnBootSec=1min, OnUnitActiveSec=5min, RandomizedDelaySec=30, AccuracySec=1s
collectors/linux/install.sh               idempotent; /usr/local/lib/smon/ (root:root 0755); never overwrites the config
collectors/linux/collector.conf.example   INI [collector]: url, host_id, token, ca_file?, proxy?, timeout_s=20
collectors/linux/ruff.toml                target-version = "py38"
collectors/windows/smon-collector.ps1     collection every 5 min
collectors/windows/install.ps1            Task Scheduler task under SYSTEM, ACL (SYSTEM, Administrators), Event Log source
collectors/windows/probe.ps1              read-only discovery of the host's capabilities (Windows stage)
collectors/windows/collector.json.example
wrappers/smon-job                         bash + curl
wrappers/smon-job.ps1                     PowerShell 5.1
```

**General collector rules:**
- **The collector keeps to the §9.2 schema:** it clips text to the limits; skips items with invalid numbers (not reported in
  `errors[]`); on a duplicate key, too many items or a partial failure it omits the section and reports it in `errors[]`.
  Exception: a 404 on inspecting a container that has just disappeared — skip the item.
- **Applicability:** no source (no `docker.sock`, no `systemctl`; no `vmms` service) → the section is omitted; the source
  exists but refuses (EACCES, a cmdlet error) → `errors[]`.
- **Sequential collection:** sections are collected one after another; every external command and API call has its own timeout.
  A collector that hangs sends no report, and the heartbeat goes critical — fail loud, no partial-report logic.
- **Transport:** `url` is a base URL, https only (http only for `localhost`, otherwise exit 2); TLS verification is mandatory,
  an optional `ca_file` (Windows — the `LocalMachine\Root` store only); redirects are forbidden; no proxy by default,
  environment proxy variables are ignored (`ProxyHandler({})`), an optional `proxy` parameter; compact JSON only.
- `sent_at` is set immediately before the POST.
- POST: timeout 20 s, no retries, no spool (a missed report is caught by the deadline).
- **Exit codes:** 0 — a 2xx response; 1 — any other response (including 3xx) or a send error; 2 — a configuration error.
  A summary line goes to stderr (journal); on Windows — to the Application Event Log (the source is registered by `install.ps1`).
- Scripts and configs live only in directories writable by root or Administrators (otherwise a task running as SYSTEM/root
  becomes a privilege escalation).
- Collector tests — `tests/collectors/` (parsers on fixtures). CI runs them under Python 3.8 without the project environment:
  `uv run --no-project --python 3.8 --with 'pytest>=7,<8.4' python -m pytest -c tests/collectors/pytest.ini --confcutdir=tests/collectors tests/collectors`
  (the hyphenated script is loaded with `importlib.util.spec_from_file_location`), plus `py_compile` under 3.8 and ruff with
  `collectors/linux/ruff.toml` and `tests/collectors/ruff.toml` (both `target-version = "py38"`, created in S5.2); a schema test for `sent_at` and item dates with 6 fractional digits (§9.1).

### 17.2 Linux collector

- `configparser(interpolation=None)`; config `/etc/smon/collector.conf` (0600 root).
- External commands — `subprocess.run(timeout=10)`; Docker API calls — 5 s each. A `statvfs` on a hung mount can block the
  process; the unit's `TimeoutStartSec` then kills it and the heartbeat goes critical (fail loud).
- **Sources:**
  - cpu — two `/proc/stat` samples 1 s apart;
  - memory — `MemTotal`, `MemAvailable` (commit is not sent);
  - disks — `/proc/self/mountinfo` (the file system type after ` - `, paths decoded from `\040`) + `os.statvfs`, filter §9.3;
  - services — `systemctl list-units --type=service --all --plain --no-legend --no-pager` (the first token ending with
    `.service`, the `●` symbol is dropped), then **one** call `systemctl show -p Id,LoadState,ActiveState,UnitFileState -- <names>`;
    mapping §9.3;
  - containers — the Docker Engine API over `/var/run/docker.sock` without a version prefix: `GET /containers/json?all=1`
    + an inspect of each; only state, health, restart_count, oom_killed and image leave the host;
  - storage — not reported (mdadm is a backlog item).

### 17.3 Windows collector (PowerShell 5.1)

- CIM calls use `-OperationTimeoutSec`; the Task Scheduler task has `ExecutionTimeLimit` ≈ 4 min and
  `MultipleInstances=IgnoreNew`, runs as SYSTEM every 5 min. A hung cmdlet is killed by the time limit, no report is sent,
  the heartbeat goes critical (fail loud).
- **Sources:**
  - memory and commit — `Win32_OperatingSystem`: `limit = TotalVirtualMemorySize`, `used = TotalVirtualMemorySize − FreeVirtualMemory`,
    `available = FreePhysicalMemory` (KiB → bytes);
  - cpu — `Win32_Processor.LoadPercentage` (`Get-Counter` uses localized counter names);
  - disks — `Win32_LogicalDisk` `DriveType=3`;
  - services — `Get-Service` without per-user services (`ServiceType & 0x40`); `TriggerInfo` → `start_mode=other`;
  - vms — `Get-VM` (if the `vmms` service exists);
  - storage — `Get-StoragePool` (without primordial), `Get-VirtualDisk`, `Get-PhysicalDisk`; names per §9.3.
- **Mandatory:** `[Net.ServicePointManager]::SecurityProtocol = Tls12`; `Invoke-WebRequest -UseBasicParsing -MaximumRedirection 0`;
  `$ProgressPreference = 'SilentlyContinue'`; `ConvertTo-Json -Depth 10 -Compress`; `@()` around pipeline results;
  enums converted to strings explicitly (ConvertTo-Json 5.1 writes them as numbers); dates
  `.ToUniversalTime().ToString("yyyy-MM-dd'T'HH:mm:ss.ffffff'Z'")` (`'o'` gives 7 fractional digits; §9.1 allows at
  most 6); the body as UTF-8 bytes. PSScriptAnalyzer runs in CI (§16.7).
- **Testing boundary (D105):** the Windows collector has no automated tests of its logic. CI runs PSScriptAnalyzer (§16.7);
  the mapping (services, VMs, storage, enums, dates) is verified by the owner on real hosts in the S6 acceptance. The server
  side is covered by the report tests of S5.
- `probe.ps1` (read-only, run first on every target host): PowerShell version (older Windows Server releases may ship
  PowerShell 4.0; WMF 5.1 is required), TLS 1.2 in .NET, availability of the Hyper-V and Storage modules. The owner reports the
  results without host names; a finding that changes the collector becomes a D record.

### 17.4 The `smon-job` wrapper

The wrappers are the reference way to report a job; any tool can instead call the API directly (§8.2), e.g. from its own
post-job hook.

- Usage: `smon-job [--data-time T] [--warning-codes 2,3] <job_id> -- <cmd…>`; no `--` or no command → exit 64.
- **One config file per job:** `/etc/smon/jobs.d/<job_id>.conf`, owned by the task's account, 0600, lines `url=`, `token=`,
  `ca_file=`; parsed with `IFS='=' read` (never `source`). The `SMON_URL` and `SMON_TOKEN` variables override the file.
  Windows: `C:\ProgramData\smon\jobs\<id>.json`, ACL SYSTEM + Administrators + the task's account.
- **The token never appears in argv:** `printf 'header = "Authorization: Bearer %s"\n' "$t" | curl -K - -sS --connect-timeout 5 --max-time 10 -w '%{http_code}' …`
  (printf is a builtin); no `-L` or `-k`; `--cacert` if `ca_file` is set; https URLs only (except localhost).
- Order:
  1. `run_id` is generated locally (`/proc/sys/kernel/random/uuid` without dashes; in ps1 — `NewGuid().ToString('N')`);
  2. start — one attempt, an error is ignored;
  3. the command runs;
  4. finish (`success`; `warning` for codes from `--warning-codes`; otherwise `failed`; `exit_code`; `message = "exit {code}"`;
     `--data-time` — RFC 3339 with an offset, passed as is).
- **Delivery:** success — 200 or 202. Retries only on a network error, a timeout, 5xx and 429 (a pause of at least
  `Retry-After`), with pauses of 10, 20, 40, 80, 160 and 300 s (≈ 10 min; for tests — env `SMON_JOB_RETRY_PAUSES`).
  On a 4xx — stderr with `error.code` right away. Retries run in the foreground: the caller waits up to ~10 min only when the
  monitor is unavailable — documented behaviour.
- No curl, config or token → the command still runs and the wrapper exits with its code; the error goes to stderr.
- The command's output is never sent.
- **Pipelines:** a shell pipeline returns the exit code of its last command (`sh -c 'pg_dump … | gzip > f'` reports gzip's
  success). Use `bash -o pipefail -c '…'`; the usage text and the README of the wrappers say so.
- **The exit code is always the command's.** Delivery errors never affect it.
- Signals are not trapped: a SIGTERM to the process group kills the wrapper and the command → the run stays open → overrun.
- **`smon-job.ps1`:** parameters `-JobId`, `-WarningCodes int[]`, `-DataTime`; the remaining arguments are the command. Only an
  external process is started (`.cmd` via `cmd /c`, `.ps1` via `powershell -File`): a cmdlet does not set `$LASTEXITCODE` and
  would produce a false success. The native command's stderr is not redirected under `$ErrorActionPreference='Stop'`. The wrapper
  itself is invoked with `-File` (with `-Command` the code collapses to 0/1) and ends with `exit $code`; 32-bit codes, including
  negative ones, are passed as is; a command that could not start → failed, exit 1.
