# Security policy

## Reporting a vulnerability

Please report vulnerabilities privately through GitHub's **"Report a vulnerability"** button on the repository's
Security tab (private vulnerability reporting). Do not open public issues for security problems.

Include what you found, how to reproduce it and the affected version or commit. There is no bug bounty.

## Supported versions

Before v1.0 only the latest `main` is supported. After v1.0 — the latest release.

## Scope notes

smon is designed for a single trusted administrator: the admin UI is expected to be reachable only over a VPN
(`docs/SPEC.md` §15). Findings that require access to the admin UI from the public internet are out of scope unless they bypass
the `SMON_ADMIN_ALLOW` allowlist or the authentication.
