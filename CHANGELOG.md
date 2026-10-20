# Changelog

All notable changes to Dredd MCP are documented here.

## [1.2.0] - 2026-10-20

### Added
- **Published-CVE check on the server itself.** `check_mcp_server` now looks up known CVEs for the named server (for example io.github.grafana/mcp-grafana v1.0.0, CVE-2026-19516, CVSS 9.1, which used to return ALLOW). A CVE yields ADVISORY, never BLOCK: BLOCK stays reserved for compromise or a malicious dependency.
- **Registry publishing from CI.** `.github/workflows/publish-registry.yml` publishes `server.json` to the MCP Registry with GitHub OIDC when a `v*` tag is pushed, so the registry listing can no longer lag the repo (it sat at 1.0.0 since May while the repo said 1.1.1).

### Changed
- `server.json` description shortened to the registry's 100-character limit, and version bumped to `1.2.0`.

## [1.1.2] - 2026-09-29

### Changed
- Refreshed corpus figures from the live `/api/v1/search/stats` (1.9M+ IOCs, ~68M documents across 70 indexes).
- Removed the retired "275+ consumers in 46 countries" line. It counted blocked, User-Agent-less scrapers as consumers, so we stopped quoting it on 2026-05-30.

## [1.1.1] - 2026-06-30

### Added
- Documented the fourth live validation axis — Liveness (`/api/v1/feed-efficacy`) — in Trust Posture.

### Changed
- Refreshed corpus copy to ~38M documents across 65 indexes (Jeevesus companion line).
- Reworded the Timeliness validation reference to point at the live kev-lead ledger instead of a fixed "~31 days ahead" average.
- `server.json` version bumped to `1.1.1`.

## [1.1.0] - 2026-06-27

### Added
- **Transitive dependency-graph checking (Shai-Hulud class).** Every preflight now resolves the target server's full npm/pypi dependency graph and joins *every* transitive package against the IOC corpus — not just the named server. Catches the supply-chain compromises that hide three levels down (stolen publish tokens on transitive deps), which a server-identity-only check is blind to.
- **`dep_graph` field on the signed verdict.** Reports whether transitive supply-chain risk was evaluated for the target (`evaluated`, `packages_checked`, `malicious_transitive`). If the manifest isn't resolvable, `evaluated: false` and Dredd drops to advisory rather than claiming clean.
- **OSV malicious-package feeds (npm + PyPI)** added to the correlation corpus behind verdicts.
- **Feed-validation surfacing** in Trust Posture: novelty (`/api/v1/feed-uniqueness`), timeliness (`/api/v1/kev-lead`), accuracy (`/api/v1/spamhaus-validation`).
- **CHANGELOG.md** — you're reading it.

### Changed
- `server.json` description rewritten to lead with dependency-graph checking and the `dep_graph` field.
- Architecture diagram now shows the correlator joining the resolved transitive dep graph (incl. OSV npm/PyPI) against the IOC corpus.

## [1.0.0]

### Added
- Initial release: pre-invocation `check_mcp_server` tool over streamable-HTTP MCP.
- HMAC-signed `BLOCK` / `ADVISORY` / `ALLOW` verdicts.
- Compromised-dependency, tool-surface-drift, remote-URL-drift, and permission-escalation signals.
- Public Watchtower dashboard (`/api/v1/dredd/watchtower.json`).
