# smon — Simple Monitor

> Pre-1.0: developed stage by stage, see [docs/STATUS.md](docs/STATUS.md). External pull requests are not accepted before v1.0.

A small self-contained monitoring service for a small infrastructure run by a single operator.

- **External checks:** HTTP(S), TCP, TLS (including certificate expiry), DNS.
- **Jobs and backups:** missed run, failure, overrun, stale data. Any task reports `start`/`finish` through a simple HTTP API —
  through the `smon-job` wrappers or a direct call from a script or a post-job hook.
- **Hosts:** lightweight push collectors for Linux/Docker and Windows/Hyper-V — disks, memory, services, containers, VMs, storage.
- **Notifications:** Telegram (Ack and Mute 1h buttons, the `/status` command), a daily summary, an external dead-man
  (healthchecks.io-compatible).
- **Interfaces:** an administrator web UI (VPN only, password-protected) and a read-only API for external agents and integrations.

The main principle: **everything is a signal with a deadline.** No fresh positive confirmation by the deadline means a problem.
Silence is never "OK".

## Documentation

| Document | Contents |
|---|---|
| [docs/SPEC.md](docs/SPEC.md) | requirements and architecture (normative) |
| [docs/ROADMAP.md](docs/ROADMAP.md) | development stages, definition of done, acceptance |
| [docs/STATUS.md](docs/STATUS.md) | current state of development |
| [docs/DECISIONS.md](docs/DECISIONS.md) | decision log |
| [AGENTS.md](AGENTS.md) | rules and process for AI agents developing the project |
| [SECURITY.md](SECURITY.md) | reporting vulnerabilities |

## Running (after stage S0)

smon ships as the Docker image `ghcr.io/<owner>/smon:<version>` with a `/data` volume and two ports: `8080` — ingest (public,
behind a reverse proxy with TLS) and `8081` — the admin UI (VPN only; published on `127.0.0.1`). An example is
`deploy/docker-compose.example.yml`; the runtime contract is `docs/SPEC.md` §16.

```bash
docker compose -f deploy/docker-compose.example.yml up -d
docker compose -f deploy/docker-compose.example.yml exec smon smon admin set-password
```

## License

[MIT](LICENSE).
