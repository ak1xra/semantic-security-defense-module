---
title: "Semantic Security Defense Module — Backend Standard"
module: "SSDM-BE"
tier: "Standard"
version: "1.0"
pairs_with: "01-semantic-norms/standard.md"
layer: "Backend: physical enforcement, execution environment, and infrastructure control layer"
scope: "Proxy configuration for organizational use, execution hooks (Pre-hook), log retention"
inherits: "Includes all Backend Basic items; only additions are documented"
target_agents: "Claude Code / Cursor (IDE・CLI) / Codex CLI"
last_updated: "2026-09-19"
timezone: "JST"
status: "Draft for review"
---

## 0. Diff from Basic

| Aspect | Basic | Standard |
|---|---|---|
| Where settings live | Repository / personal | **Organizational distribution (managed)**. Cannot be disabled by personal or project settings |
| Network | Tool domain allowlists | Route through an **organizational proxy**; allowlist and logs are held on the org side |
| Hooks | Auxiliary scripts in the repository | **Org-distributed Pre-hooks**. Hash-pinned; personal hooks disabled |
| Logs | None | **Transfer tool-call–level audit logs to org storage**; define retention period |
| Self-settings protection | Tool protected paths | + **OS immutable attributes**, or separate-user execution |

---

## 1. Applicability conditions

Same as Frontend Standard (D2, P1–P3, a responsible party exists separately).

---

## 2. Overall control picture

```text
[Developer endpoint]
  ├─ managed settings (MDM / managed-settings.json / server-managed) ← org distributes; immutable on the endpoint
  ├─ /opt/ssdm/hooks/*.sh (root-owned, SHA256SUMS pinned)
  ├─ Agent (Claude Code / Cursor / Codex)
  │     ├─ Pre-hook → block / pass
  │     ├─ Sandbox (FS · network)
  │     └─ Post-hook → audit log (endpoint primary buffer)
  └─ Log shipper (fluent-bit / rsyslog, etc.)
            │ TLS
            ▼
[Org proxy]  Only allowed domains pass; all requests recorded
            ▼
[Log platform]  Retention and access governed by policy
```

---

## 3. Strengthened self-settings protection (OS layer)

Basic permissions cannot stop an agent running as the same user. Standard requires one of the following.

| Method | Implementation | Pros | Cons |
|---|---|---|---|
| A. Immutable attributes | Linux: `sudo chattr +i CLAUDE.md .claude/settings.json .mcp.json`. macOS: `chflags uchg` (`schg` if root) | Simple. Cannot be cleared except by root | Legitimate updates also need root |
| B. Separate-user execution | Start the agent as dedicated user `agent`. Rules/settings owned by `admin` at `644`; repository writable by `agent` | Privilege separation via standard OS mechanisms | Hard to implement with IDE integration (Cursor) |
| C. Completed on the managed side | Claude Code: `allowManagedPermissionRulesOnly` + `allowManagedHooksOnly` + `ConfigChange` hook blocks personal settings changes | Distribution alone suffices | Claude Code only |

[Hypothesis] For organizations centered on Claude Code, C is realistic; for organizations that also use Cursor, using A in combination is realistic.

---

## 4. Organizational proxy

### 4-1. Requirements

| # | Requirement | Rationale |
|---|---|---|
| PX-1 | Route all agent-originated HTTP(S) through the org proxy. Block direct connections | Unify data-exfiltration paths |
| PX-2 | Default deny. Allow only an explicit domain-level list (model APIs, package registries, internal Git) | Minimization |
| PX-3 | Per request, record “datetime, user, endpoint, destination, method, size.” Do not record bodies | Balance traceability with D2 protection |
| PX-4 | TLS content inspection is **optional** in Standard (required in Advanced) | Claude Code’s built-in proxy does not inspect TLS by default |

### 4-2. Connection in Claude Code

| Setting | Value | Notes |
|---|---|---|
| `env.HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` | Org proxy | Official docs: placing these in the settings file `env` block also applies to background agents. After applying the allowlist, the sandbox tunnels permitted connections to the upstream proxy |
| `sandbox.network.allowedDomains` | Org allowlist | |
| `sandbox.network.allowManagedDomainsOnly` | `true` | Disable additions outside managed |
| `sandbox.network.httpProxyPort` / `socksProxyPort` | When using a custom proxy | Officially described for “decrypt, inspect, record, and integrate with existing infrastructure” |

### 4-3. Cursor / Codex

| Tool | Setting |
|---|---|
| Cursor IDE | Distribute Sandbox network policy via admin dashboard. Pin `sandbox.json` allow domains to org values. Proxy via environment variables |
| Cursor CLI | Proxy via environment variables and settings file (documented officially). Detail keys are [Needs verification] |
| Codex CLI | Default `network_access = false`. Approve only when needed. Proxy setting keys are [Needs verification] |

### 4-4. Initial allow-domain list (example; finalize per organization)

| Use | Domains |
|---|---|
| Model API | API hosts of the providers in use |
| Packages | `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org` |
| Source control | Internal Git hosts. If allowing `github.com`, recognize it can become an exfiltration path as officially warned |

---

## 5. Execution hooks (Pre-hook)

### 5-1. Placement and tamper resistance

| Item | Prescription |
|---|---|
| Placement | `/opt/ssdm/hooks/` (root-owned, `755`, not writable) |
| Integrity | Ship `SHA256SUMS`; verify via `SessionStart` hook (adopt Frontend Advanced §3-3 `verify-hooks.sh` in Standard as well) |
| Disable personal hooks | Claude Code: `allowManagedHooksOnly: true` |
| Detect settings changes | `ConfigChange` hook (`user_settings\|project_settings\|local_settings`): exit 2 block + log |

### 5-2. Pre-hook composition

| Hook | Event | Role | Artifact |
|---|---|---|---|
| `guard-bash.sh` | `PreToolUse` / `Bash` | Block shell-direct use of fetched artifacts, decrypt-and-execute, secret reads, settings rewrite | Frontend Basic §3-2 (verified) |
| `guard-mcp.sh` | `PreToolUse` / `mcp__.*` | Block MCP servers outside the allowlist | §5-3 |
| `verify-hooks.sh` | `SessionStart` | Hook hash verification | Frontend Advanced §3-3 |

### 5-3. `guard-mcp.sh`

```bash
#!/bin/bash
# guard-mcp.sh — For PreToolUse(mcp__*). Block tool calls to servers not on the allowlist.
set -u
ALLOW="${SSDM_MCP_ALLOW:-/opt/ssdm/mcp-allow.txt}"   # One server name per line. Root-owned.
tool=$(jq -r '.tool_name // empty' 2>/dev/null) || { echo "guard-mcp: invalid input" >&2; exit 2; }
server=$(printf '%s' "$tool" | sed -E 's/^mcp__([^_]+(_[^_]+)*)__.*/\1/')
[ -f "$ALLOW" ] || { echo "guard-mcp: no allowlist. Blocking all MCP" >&2; exit 2; }
grep -qxF "$server" "$ALLOW" && exit 0
echo "guard-mcp: MCP server '$server' is not on the allowlist" >&2; exit 2
```

Note: Tool names follow `mcp__<server>__<tool>`. Parsing when the server name contains underscores has ambiguous boundaries, so **operate the allowlist with exact match and verify with real tool names at introduction**.

### 5-4. Claude Code managed settings (hooks portion)

```json
{
  "allowManagedHooksOnly": true,
  "hooks": {
    "SessionStart": [
      { "hooks": [ { "type": "command", "command": "/opt/ssdm/hooks/verify-hooks.sh", "args": [], "timeout": 10 } ] }
    ],
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [ { "type": "command", "command": "/opt/ssdm/hooks/guard-bash.sh", "args": [], "timeout": 10 } ] },
      { "matcher": "mcp__.*", "hooks": [ { "type": "command", "command": "/opt/ssdm/hooks/guard-mcp.sh", "args": [], "timeout": 10 } ] }
    ],
    "PostToolUse": [
      { "matcher": "*", "hooks": [ { "type": "command", "command": "/opt/ssdm/hooks/audit-log.sh", "args": [], "timeout": 10 } ] }
    ],
    "PostToolUseFailure": [
      { "matcher": "*", "hooks": [ { "type": "command", "command": "/opt/ssdm/hooks/audit-log.sh", "args": [], "timeout": 10 } ] }
    ],
    "ConfigChange": [
      { "matcher": "user_settings|project_settings|local_settings", "hooks": [ { "type": "command", "command": "/opt/ssdm/hooks/deny-all.sh", "args": [], "timeout": 5 } ] }
    ]
  }
}
```

Reflecting official documentation caveats:
- Command hooks for `PreToolUse` **do not block** on timeout (proceed to the normal permission flow). Keep hooks lightweight; do not rely on timeout for blocking.
- `if` filters are best-effort. **Hard allow/deny is done via permissions; hooks are auxiliary.**
- exit 1 does not block. Blocking is exit 2 only.

### 5-5. Cursor / Codex

| Tool | Pre-hook equivalent |
|---|---|
| Cursor IDE | Hook mechanism not in official docs [Not specified]. Substitute with `permissions.json` `block_instructions` and admin-dashboard global `permissions.json` |
| Cursor CLI | [Not specified] |
| Codex CLI | `[projects."<path>"] trust_level = "untrusted"` requires approval for commands outside execution-policy rules. Organizational constraints via `requirements.toml` are [Needs verification] |

---

## 6. Log retention

### 6-1. What to record

| Kind | Source | Recorded fields | Not recorded |
|---|---|---|---|
| Tool-call log | `audit-log.sh` (Post-hook) | Time, user, endpoint, session_id, prompt_id, tool name, command/path/URL, error on failure | **Tool output bodies, file contents** |
| Block log | stderr of `guard-*.sh` → `PostToolUseFailure` of `audit-log.sh` | Block reason | — |
| Settings-change log | `ConfigChange` hook | Change source (user/project/local) | — |
| Network log | Org proxy | Destination, method, size | Body |
| Integrity-verification log | `verify-hooks.sh` | Result | — |

### 6-2. Retention

| Item | Prescription |
|---|---|
| Endpoint side | `~/.ssdm/audit.jsonl`, `600`. Primary buffer; may be deleted after a set period once shipped |
| Shipping | TLS ship via fluent-bit / rsyslog, etc. Interval set by org policy |
| Org side | Append-only storage. Read access limited to audit/security roles |
| Retention period | Follow org log-management policy. AI operator guideline C-6 ① states to “consider recording methods, frequency, retention period, etc., in light of the importance for investigating causes of incidents, studying recurrence prevention, and proving elements of liability for damages” (no specific period is set) |
| Correlation | `session_id` / `prompt_id` align with Claude Code OpenTelemetry event attributes. When using the OTel path, distribute `OTEL_*` variables via managed `env` |

### 6-3. fluent-bit config example (endpoint side)

```ini
[INPUT]
    Name   tail
    Path   /home/*/.ssdm/audit.jsonl
    Parser json
    Tag    ssdm.audit
    DB     /var/lib/fluent-bit/ssdm.db

[OUTPUT]
    Name   forward
    Match  ssdm.*
    Host   logs.example.internal
    Port   24224
    tls    on
    tls.verify on
```

Adjust hostname, port, and parser name to the environment. Retry on ship failure is covered by offset management via `DB`.

---

## 7. Verification (additions to Basic BE-1–7)

| ID | Procedure | Expected result |
|---|---|---|
| BE-8 | Write `allow: ["Bash(rm -rf *)"]` in personal settings | managed deny wins. Blocked and recorded via `ConfigChange` |
| BE-9 | Alter `/opt/ssdm/hooks/guard-bash.sh` by 1 byte | `verify-hooks.sh` stops the session |
| BE-10 | `curl` to a non-allowed domain | Sandbox blocks. Does not reach the proxy |
| BE-11 | `curl` to an allowed domain | Succeeds; recorded in proxy logs |
| BE-12 | Tool call to a non-allowed MCP server | Blocked by `guard-mcp.sh` |
| BE-13 | After the session, search the org log platform for that session_id | All tool calls present. No output bodies |
| BE-14 | Agent appends to `CLAUDE.md` with `chattr +i` | Fails (when using method A) |

---

## 8. Unverified items

- Cursor CLI proxy setting keys; Codex proxy setting keys
- Codex `requirements.toml` format
- `guard-mcp.sh` server-name parsing (server names containing underscores)
- fluent-bit config is syntax only; runtime behavior in a real environment is unconfirmed
