# Dredd MCP — Pre-Flight Security for the MCP Ecosystem

> *"Jeevesus saves. Dredd judges."*

**Dredd MCP** is a pre-invocation security check for the [Model Context Protocol](https://modelcontextprotocol.io) ecosystem. Before your agent calls a tool on any other MCP server, Dredd renders a verdict: `BLOCK`, `ADVISORY`, or `ALLOW`. Every verdict is HMAC-signed and cites the IOC or behavioral signal that drove the decision.

The MCP ecosystem has had no defender. Three PyPI ML packages were compromised in eight days during late April 2026. Twenty-plus MCP-named GitHub repositories were caught serving SmartLoader malware in the wild. The official MCP Registry was clean of those when we measured — but the typosquat surface is wide open.

Dredd is the layer that catches the next compromise *before* the malicious tool gets called.

---

## What Dredd Checks

Every preflight call evaluates four signals:

1. **Compromised dependency.** The target server's package manifest is parsed and joined against our continuously updated IOC corpus (Socket, Aikido, GitGuardian, ReversingLabs, Phylum, StepSecurity, Wiz). If the server pins `lightning==2.6.2` or any other known-compromised version, the call is blocked.
2. **Tool surface drift.** The list of tools the server exposes today versus the snapshot the user originally approved. New tools that appeared since the last review trigger an advisory. Mid-session rugpull is the threat model.
3. **Remote URL drift.** The server's runtime endpoint compared against the URL it published in the registry. A server quietly calling out to a different host than the one you signed up for is a hijack signature.
4. **Permission escalation.** A server requesting write or exec permissions it did not have last week.

The verdict comes back signed in under 200 ms (Cloudflare-edge cached, 5-minute TTL). The hook fails *open* by default — if our endpoint is ever down, Dredd does not brick your tooling.

---

## Install — Claude Desktop

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "dredd": {
      "url": "https://analytics.dugganusa.com/api/v1/dredd/mcp"
    }
  }
}
```

Restart Claude Desktop. You'll see Dredd available with one tool: `check_mcp_server`.

## Install — Cursor

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "dredd": {
      "url": "https://analytics.dugganusa.com/api/v1/dredd/mcp"
    }
  }
}
```

## Test from terminal

```bash
curl -X POST https://analytics.dugganusa.com/api/v1/dredd/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

You should see one tool: `check_mcp_server`.

---

## The Tool

### `check_mcp_server`

| Argument | Type | Required | Description |
|---|---|---|---|
| `server` | string | yes | MCP server name (e.g. `io.github.foo/bar`) or substring |
| `version` | string | no | Optional semver of the server |
| `tool` | string | no | Optional name of the specific tool being invoked |

Returns a JSON verdict:

```json
{
  "success": true,
  "server": "io.github.foo/bar",
  "version": "1.2.3",
  "tool": "list_files",
  "verdict": "ALLOW",
  "severity": "clean",
  "findings_count": 0,
  "findings": [],
  "checked_at": "2026-05-04T20:00:00Z",
  "ttl_seconds": 300,
  "signature": "sha256=..."
}
```

Verdict values:

| Verdict | Severity tier | What to do |
|---|---|---|
| `BLOCK` | `critical` or `high` | Refuse the invocation. Do not call the target tool. |
| `ADVISORY` | `medium` or `advisory` | Surface to the user; let them decide. |
| `ALLOW` | `clean` | Proceed. |

The `signature` field is an HMAC-SHA256 of the canonical verdict body using a server-side secret. Hooks should verify this to defeat MITM-forged "all clear" responses (verification key distributed out of band on request).

---

## The Public Watchtower

Real-time aggregate dashboard of every active finding across the registry — free, no auth, no email gate:

**[`https://analytics.dugganusa.com/api/v1/dredd/watchtower.json`](https://analytics.dugganusa.com/api/v1/dredd/watchtower.json)**

Returns counts by severity, recent findings, current verdict (`CLEAN` / `WATCH` / `HIGH` / `CRITICAL`).

Updated continuously as the daily fetcher + correlator pipeline runs against the registered MCP corpus.

---

## How Dredd Works

```
       ┌─────────────────────────────────────────────────┐
       │  Your Agent (Claude Desktop / Cursor / custom)   │
       │     calls check_mcp_server(server, version, tool)│
       └─────────────────┬───────────────────────────────┘
                         │ JSON-RPC over HTTPS
                         ▼
        ┌────────────────────────────────────────────────┐
        │  Dredd MCP — analytics.dugganusa.com/api/v1/dredd/mcp │
        │  - look up findings for (server, version, tool)│
        │  - aggregate severity, render verdict          │
        │  - HMAC-sign canonical verdict                 │
        └────────────────┬───────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────────────────┐
        │  mcp_findings index — populated by             │
        │  daily fetcher + correlator joining            │
        │  mcp_dependencies × IOC corpus                 │
        │  (Socket, Aikido, GitGuardian, ReversingLabs)  │
        └────────────────────────────────────────────────┘
```

The correlation cadence today is 12 hours (08:30 UTC and 20:30 UTC). When a real compromise lands in the registered-MCP corpus, cadence tightens.

---

## Trust Posture

- **HMAC-signed responses.** Hook implementations should verify the `signature` field on every verdict.
- **Fail-open by default.** If our endpoint is down, Dredd does not brick your tooling — it returns "advisory: backend unavailable" and lets the user decide. Document override (`DREDD_BYPASS=<reason>`) for critical workflows.
- **Read-only.** Dredd never modifies your environment. Verdict only.
- **No tool argument leakage.** Hooks should send `(server, version, tool)` only — never the contents of tool arguments. Those stay on your machine.
- **95% epistemic ceiling.** We cap our claims at 95% per [DugganUSA's epistemic humility rule](https://www.dugganusa.com/post/95-percent-epistemic-humility). Coverage gap: about 60-70% of MCP servers in the registry today don't expose a public source repository, which means Dredd cannot inspect their dependency tree. The advisory tier exists for those.

---

## The Family

Dredd is the 13th member of the [DugganUSA defender family](https://github.com/pduggusa) — and the first MCP-native member:

- [`dugganusa-scanner-core`](https://github.com/pduggusa/dugganusa-scanner-core) — Core IOC scanning engine
- [`dugganusa-vscode`](https://github.com/pduggusa/dugganusa-vscode) — VS Code extension
- [`dugganusa-splunk`](https://github.com/pduggusa/dugganusa-splunk) — Splunk Technology Add-on
- [`dugganusa-slack`](https://github.com/pduggusa/dugganusa-slack) — Slack bot
- [`dugganusa-raycast`](https://github.com/pduggusa/dugganusa-raycast) — Raycast extension
- [`dugganusa-sentinel`](https://github.com/pduggusa/dugganusa-sentinel) — Microsoft Sentinel TAXII connector
- [`dugganusa-obsidian`](https://github.com/pduggusa/dugganusa-obsidian) — Obsidian plugin
- [`dugganusa-nvim`](https://github.com/pduggusa/dugganusa-nvim) — Neovim plugin
- [`dugganusa-elastic`](https://github.com/pduggusa/dugganusa-elastic) — Elastic / OpenSearch integration
- [`dugganusa-edge-shield`](https://github.com/pduggusa/dugganusa-edge-shield) — Cloudflare Worker
- [`dugganusa-cli`](https://github.com/pduggusa/dugganusa-cli) — CLI scanner
- [`dugganusa-chrome`](https://github.com/pduggusa/dugganusa-chrome) — Chrome extension
- [`dugganusa-action`](https://github.com/pduggusa/dugganusa-action) — GitHub Action

Companion MCP server: **[Jeevesus](https://github.com/pduggusa)** — natural-language threat intelligence search across 17.9M documents. *Jeevesus saves. Dredd judges.*

---

## License

MIT — see [LICENSE](LICENSE).

## Support

- Watchtower dashboard: [analytics.dugganusa.com/api/v1/dredd/watchtower.json](https://analytics.dugganusa.com/api/v1/dredd/watchtower.json)
- Issues: [github.com/pduggusa/dredd-mcp/issues](https://github.com/pduggusa/dredd-mcp/issues)
- DugganUSA blog: [www.dugganusa.com](https://www.dugganusa.com)

---

*Built in Minneapolis. Defender-grade. Read-only. Receipts do the work.*

---

<!-- DUGGANUSA-FAMILY-FOOTER-V1 -->
## DugganUSA Defender Family

Same threat corpus, surfaced wherever you live. Open source, MIT licensed, receipts on every repo.

| Plugin | Surface |
|---|---|
| [dugganusa-scanner-core](https://github.com/pduggusa/dugganusa-scanner-core) | Core IOC scanning engine |
| [dugganusa-vscode](https://github.com/pduggusa/dugganusa-vscode) | VS Code extension |
| [dugganusa-splunk](https://github.com/pduggusa/dugganusa-splunk) | Splunk Technology Add-on |
| [dugganusa-slack](https://github.com/pduggusa/dugganusa-slack) | Slack bot |
| [dugganusa-raycast](https://github.com/pduggusa/dugganusa-raycast) | Raycast extension |
| [dugganusa-sentinel](https://github.com/pduggusa/dugganusa-sentinel) | Microsoft Sentinel TAXII connector |
| [dugganusa-obsidian](https://github.com/pduggusa/dugganusa-obsidian) | Obsidian plugin |
| [dugganusa-nvim](https://github.com/pduggusa/dugganusa-nvim) | Neovim plugin |
| [dugganusa-elastic](https://github.com/pduggusa/dugganusa-elastic) | Elastic / OpenSearch integration |
| [dugganusa-edge-shield](https://github.com/pduggusa/dugganusa-edge-shield) | Cloudflare Worker |
| [dugganusa-cli](https://github.com/pduggusa/dugganusa-cli) | CLI scanner |
| [dugganusa-chrome](https://github.com/pduggusa/dugganusa-chrome) | Chrome extension |
| [dugganusa-action](https://github.com/pduggusa/dugganusa-action) | GitHub Action |
| **dredd-mcp** _(this repo)_ | Pre-flight MCP security (this repo) |

Backed by the live DugganUSA threat intel platform: [analytics.dugganusa.com](https://analytics.dugganusa.com).

_Jeevesus saves. Dredd judges._
