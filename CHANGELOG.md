# Changelog

All notable changes to Dredd MCP are documented here.

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
