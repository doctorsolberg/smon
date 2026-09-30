# Development status

- **Stage:** S0 — not started.
- **Open branches and PRs:** none.

## Done

- Specification v2.2 (`docs/SPEC.md`) after four rounds of independent review and a final review; decisions —
  `docs/DECISIONS.md` (D1–D106).
- Stages (`docs/ROADMAP.md`); process and rules for agents (`AGENTS.md`).

## Unfinished

None.

## Next step

The orchestrator starts S0 (`docs/ROADMAP.md`, S0): delegates checkpoint S0.1 to an implementer on branch `stage/S0.1`.

## External prerequisites (done by the owner)

- [ ] `gh` is installed and authenticated under the repository owner's account (`gh auth status`); `gh auth setup-git` is not
      run; the git credential used for pushes may update workflow files (`workflow` scope); Docker is available where the
      orchestrator runs ([stand] scenarios) — before S0.
- [ ] The agent tool's own commit attribution (co-author trailers, "Generated with" lines) is turned off — before S0.
- [ ] Repository settings — before S0: Actions enabled; "Allow squash merging"; secret scanning and push protection; private
      vulnerability reporting; approval required for workflow runs from forks.
- [ ] A branch protection rule for `main` — before S0: "Require a pull request before merging" on, **"Require approvals" off**
      (the orchestrator opens PRs under the owner's account and merges them itself once CI is green); "Require status checks
      to pass before merging" on; **"Do not allow bypassing the above settings" on** (nobody, including the agent with the
      owner's admin rights, can merge a red PR or push to `main` directly); force pushes and deletions off. Required status
      checks can be selected only after they have run once: add them (including `guard`) after the first PR of S0.1, and add
      the new CI jobs after S0.3 (image), S4.2 (`windows`) and S5.2 (Python 3.8 collector tests).
- [ ] Make the GHCR package public after the first release — S0.
- [ ] A Telegram bot (BotFather), a `chat_id` and your own `user_id` — S2 (real check).
- [ ] A healthchecks.io (or compatible) check with period 1 min and grace 5 min, with email and Telegram notifications on their
      side — S3.
- [ ] VPN access to the admin UI (e.g. `tailscale serve`) — S3.
- [ ] Encrypted off-host copies of `/data/backups` — S3.
- [ ] DNS and TLS for the public ingest domain — S4.
- [ ] Test VMs for the collectors ([owner] scenarios; agents get no access to them) — S5 and S6.

The deployment plan for a concrete infrastructure is kept outside this repository.

## Deviations from the SPEC

None.

## Questions

None.

## Notes for later stages

None.

## Backlog after v1.0

See `docs/ROADMAP.md`, "Backlog after v1.0". New items go here.
