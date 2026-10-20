---
title: "Semantic Security Defense Module — Backend Advanced"
module: "SSDM-BE"
tier: "Advanced"
version: "1.0"
pairs_with: "01-semantic-norms/advanced.md"
layer: "Backend: physical enforcement, execution environment, and infrastructure control layer"
scope: "Fully isolated sandbox (Docker/VM), physical Egress block, tamper-resistant audit logs"
inherits: "Includes all Backend Basic and Standard items; only additions are documented"
target_agents: "Claude Code / Cursor (IDE・CLI) / Codex CLI"
last_updated: "2026-09-19"
timezone: "JST"
status: "Draft for review"
---

## 0. Diff from Standard

| Aspect | Standard | Advanced |
|---|---|---|
| Primary defense | Tool sandbox + org settings | **The isolation environment (container/VM) itself**. Tool settings are a second defense |
| Secrets | Tool refuses reads | **Do not exist in the isolation environment** |
| Egress | Allowlist via proxy | **Default is physical block**. Allow only via TLS-terminating, content-inspecting proxy |
| Audit logs | Ship + append-only | **hash chain + WORM retention + external anchor** |
| Assumed breach mode | Settings relaxation / bypass | **Assume tool vulnerabilities** (e.g. Cursor CVE-2026-50548/50549 sandbox escape); even after escape, cannot leave the isolation environment |

---

## 1. Applicability conditions

Same as Frontend Advanced (D3, P4, separation of duties + audit).

---

## 2. Isolation architecture

```text
[Developer endpoint]  ── SSH / Dev Container attach only ──▶ [Isolation environment]
   IDE display                                          ├─ Agent (non-root, all caps dropped, no-new-privileges)
   No secrets                                           ├─ Repository (work volume. Rules/hooks are read-only mount)
                                                     ├─ Secrets: none (no production credentials or D3 data)
                                                     ├─ Network: none (--network none)
                                                     │      Exception: one-way path to inspection proxy only (§4)
                                                     └─ Audit log → ship with hash chain outside read-only volume (§5)
[Inspection proxy]  TLS terminate · content inspect · full record ── allowed domains only ──▶ [Model API / registry]
[WORM log]     Object Lock / append-only. head hash to external anchor
```

Choice of isolation means:

| Means | Isolation strength | Applicability |
|---|---|---|
| Docker (standard runc) | Shared kernel. Namespaces + seccomp + caps | Minimum. Allowed only if all §3 hardening settings are applied |
| gVisor / Kata Containers | Userspace kernel / lightweight VM | Recommended. Raise isolation strength with Docker compatibility |
| Dedicated VM (Firecracker, etc.) | Hypervisor boundary | For government/finance sensitive domains, default to VM [Hypothesis] |

---

## 3. Fully isolated sandbox (Docker implementation example)

### 3-1. `.devcontainer/devcontainer.json`

```json
{
  "name": "ssdm-advanced",
  "build": { "dockerfile": "Dockerfile" },
  "runArgs": [
    "--network", "none",
    "--cap-drop", "ALL",
    "--security-opt", "no-new-privileges:true",
    "--security-opt", "seccomp=./seccomp-ssdm.json",
    "--read-only",
    "--tmpfs", "/tmp:rw,noexec,nosuid,size=512m",
    "--pids-limit", "512",
    "--memory", "4g",
    "--cpus", "2"
  ],
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",
  "workspaceFolder": "/workspace",
  "mounts": [
    "source=${localWorkspaceFolder}/.claude,target=/workspace/.claude,type=bind,readonly",
    "source=${localWorkspaceFolder}/CLAUDE.md,target=/workspace/CLAUDE.md,type=bind,readonly",
    "source=${localWorkspaceFolder}/.cursor,target=/workspace/.cursor,type=bind,readonly",
    "source=${localWorkspaceFolder}/.cursorignore,target=/workspace/.cursorignore,type=bind,readonly",
    "source=ssdm-audit-${devcontainerId},target=/var/ssdm,type=volume"
  ],
  "remoteUser": "agent",
  "containerEnv": {
    "SSDM_AUDIT_LOG": "/var/ssdm/audit.chain.jsonl",
    "HOME": "/home/agent"
  }
}
```

Key points:

| Setting | Purpose |
|---|---|
| `--network none` | Physical Egress block. If model API is needed, replace with the dedicated network in §4 |
| `--cap-drop ALL`, `no-new-privileges` | Block privilege escalation |
| `--read-only` + `--tmpfs /tmp noexec` | Immutable root FS; no execution in temp |
| Mount rules/hooks/settings `readonly` | Block agent self-modification **at the mount layer** |
| `remoteUser: agent` | Non-root |
| Do not include secrets in `mounts`; do not put tokens in `containerEnv` | Non-existence of secrets |

### 3-2. `Dockerfile`

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends \
      git jq ca-certificates bubblewrap socat ripgrep \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -u 10001 -s /bin/bash agent \
    && mkdir -p /opt/ssdm/hooks /var/ssdm \
    && chown agent:agent /var/ssdm
COPY --chown=root:root hooks/ /opt/ssdm/hooks/
RUN chmod 755 /opt/ssdm/hooks/*.sh && chmod 644 /opt/ssdm/hooks/SHA256SUMS
# Install agent CLIs at build time with pinned versions (no runtime download)
USER agent
WORKDIR /workspace
```

Follow each tool’s official procedure for installing agent CLIs (Claude Code, etc.), and install at **build time with pinned versions**. Build on CI with network access; deploy with image digests pinned.

### 3-3. `seccomp-ssdm.json`

Based on Docker’s default seccomp profile, additionally deny the following [Hypothesis: minimal set].

| syscall | Reason |
|---|---|
| `mount`, `umount2`, `pivot_root` | Mount operations |
| `ptrace`, `process_vm_readv`, `process_vm_writev` | Peeking at other processes |
| `unshare`, `setns` | Namespace operations (if using bubblewrap inside, allow `unshare`, which becomes equivalent to `enableWeakerNestedSandbox`. Need for double sandbox is §3-5) |
| `keyctl`, `add_key`, `request_key` | Kernel keyring |
| `bpf`, `perf_event_open` | Observation / abuse |

Create the profile by downloading Docker’s official default profile and changing the above to `SCMP_ACT_ERRNO`. Full JSON is environment-dependent and not included in this document.

### 3-4. Host-side requirements

| # | Requirement |
|---|---|
| H-1 | Docker daemon in rootless mode, or a dedicated host. Do not pass `/var/run/docker.sock` to the agent (official warning: reachable to the host via the socket) |
| H-2 | Do not place production credentials on the host either. Run the isolation environment on a “host with no secrets” |
| H-3 | Distribute images with digest pinning after signature verification (cosign, etc.) |
| H-4 | Host `sysctl kernel.unprivileged_userns_clone`, etc., follow org baseline. Needed only when using bubblewrap inside |

### 3-5. Layering with the tool-side sandbox

| Choice | Content | Judgment |
|---|---|---|
| Double (recommended) | Also enable Claude Code `sandbox.enabled: true` inside the container | Functions as the first stage before container escape. Official docs note that in unprivileged containers bwrap cannot mount `/proc` and `enableWeakerNestedSandbox` is needed. That weakens isolation, so allow **only when container-side isolation is sufficient** |
| Single | Depend on the container only; disable the tool sandbox | Acceptable when isolating with gVisor / VM |

---

## 4. Physical Egress block

### 4-1. Principles

| State | Setting |
|---|---|
| Default | `--network none`. Block all, including DNS |
| Model API needed | Place only the proxy container on a dedicated Docker network (internal); the isolation environment cannot reach anything else |
| Package fetch needed | Allow internal mirrors (Nexus / Artifactory, etc.) via the proxy. No direct connection to public registries |

### 4-2. Dedicated network layout

```bash
# On the host. Internal network that only allows isolation environment ↔ proxy
docker network create --internal ssdm-egress
# Proxy connects to both ssdm-egress and an external network (this alone goes out)
docker run -d --name ssdm-proxy --network ssdm-egress -v ./proxy.conf:/etc/proxy.conf:ro <inspection-proxy-image>
docker network connect <externally-reachable-network> ssdm-proxy
# Isolation environment is ssdm-egress only
# In devcontainer.json runArgs, remove "--network","none" and use "--network","ssdm-egress"
```

An `--internal` network is not routed outside the host. The only path from the isolation environment is the proxy container.

### 4-3. Host nftables (tighten the proxy container’s egress)

```text
table inet ssdm {
  set allowed_hosts { type ipv4_addr; flags interval; elements = { <model-API IP ranges>, <internal-mirror IPs> } }
  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
    iifname "br-ssdm-egress" ip daddr @allowed_hosts tcp dport 443 accept
    iifname "br-ssdm-egress" log prefix "ssdm-egress-drop " drop
  }
}
```

Fix IP ranges without depending on DNS resolution (prevent exfiltration via DNS). Confirm the bridge name with `docker network inspect`. **This syntax is unverified because `nft` was unavailable in the author’s environment** [Needs verification].

### 4-4. Inspection-proxy requirements

| # | Requirement |
|---|---|
| E-1 | TLS terminate. CA distributed into the image at build time (runtime fetch forbidden) |
| E-2 | Allow down to hostname + path prefix (e.g. model API `/v1/messages` only) |
| E-3 | Inspect request bodies for D3 patterns (personal identifiers, account-number formats, etc.). On detection, block + record. Patterns defined by the organization |
| E-4 | Ship all requests to the §5 log platform (bodies: inspection results only; do not store bodies themselves) |
| E-5 | Caps on response size and rate (deter DoS and bulk exfiltration) |

Claude Code’s built-in proxy is officially documented as “judging by hostname without inspecting TLS” and “leaving room for domain fronting,” so in Advanced **do not use the built-in proxy as the final Egress defense**.

---

## 5. Tamper-resistant audit logs

### 5-1. Architecture

```text
[Isolation environment] audit-chain.sh append ─▶ /var/ssdm/audit.chain.jsonl (volume, append-only)
      │ On each append, or every N seconds
      ▼
[Ship] fluent-bit (TLS, mutual auth)
      ▼
[WORM] S3 Object Lock (Compliance mode) / append-only storage. Retention period fixed
      │ Daily
      ▼
[External anchor] Record that day’s trailing head hash somewhere independent of the log platform
               (signed timestamp, another org’s ledger, or paper held by the audit department)
```

### 5-2. `audit-chain.sh`

A hash chain where each line includes the previous line’s hash. `append` appends; `verify` verifies all entries.

```bash
#!/bin/bash
# audit-chain.sh — Advanced: tamper-detecting audit log. Each line links the previous hash (hash chain).
# Usage: append  … append one JSON object from stdin   |   verify … verify the whole chain (exit 1 = tamper/missing)
# The endpoint is only a primary buffer. Assume shipping to WORM retention (S3 Object Lock, etc.) on each append.
set -u
LOG="${SSDM_AUDIT_LOG:-$HOME/.ssdm/audit.chain.jsonl}"
mkdir -p "$(dirname "$LOG")"; umask 077
case "${1:-}" in
  append)
    prev=$( [ -s "$LOG" ] && tail -n1 "$LOG" | jq -r '.hash' || echo "GENESIS")
    seq=$( [ -s "$LOG" ] && tail -n1 "$LOG" | jq -r '.seq + 1' || echo 1)
    body=$(jq -c --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --argjson seq "$seq" --arg prev "$prev" \
      '{seq:$seq, ts:$ts, prev:$prev, event:.}') || { echo "audit-chain: invalid input JSON" >&2; exit 2; }
    hash=$(printf '%s' "$body" | sha256sum | cut -d' ' -f1)
    printf '%s\n' "$(printf '%s' "$body" | jq -c --arg h "$hash" '. + {hash:$h}')" >> "$LOG" ;;
  verify)
    prev="GENESIS"; n=0
    while IFS= read -r line; do
      n=$((n+1))
      seq=$(jq -r '.seq' <<<"$line"); p=$(jq -r '.prev' <<<"$line"); h=$(jq -r '.hash' <<<"$line")
      [ "$seq" = "$n" ] || { echo "NG line $n: seq=$seq (missing or inserted)"; exit 1; }
      [ "$p" = "$prev" ] || { echo "NG line $n: prev mismatch"; exit 1; }
      calc=$(jq -c 'del(.hash)' <<<"$line" | tr -d '\n' | sha256sum | cut -d' ' -f1)
      [ "$calc" = "$h" ] || { echo "NG line $n: hash mismatch (tampered)"; exit 1; }
      prev="$h"
    done < "$LOG"
    echo "OK $n entries, chain intact, head=$prev" ;;
  *) echo "usage: $0 append|verify" >&2; exit 2 ;;
esac
```

Verified (2026-09-19):

| Operation | `verify` result |
|---|---|
| Normal 3 entries | `OK 3 entries, chain intact` |
| Rewrite contents of line 2 | `NG line 2: hash mismatch (tampered)` |
| Delete line 2 | `NG line 2: seq=3 (missing or inserted)` |
| **Delete the last 1 line** | `OK 2 entries` (**undetectable by the chain alone**) |

Trailing truncation cannot be detected by the chain alone. This is why §5-1’s **external anchor (independent storage of the head hash)** is mandatory. On verification, also confirm that “the stored head hash matches `verify`’s head.”

Calls from Post-hook pipe Standard `audit-log.sh` output into `audit-chain.sh append` (store `audit-log.sh`’s JSON shaping as `event` as-is).

### 5-3. Retention requirements

| Item | Prescription |
|---|---|
| Retention method | WORM. S3 Object Lock in Compliance mode (undeletable within retention even by root) |
| Retention period | Follow org policy and applicable norms (not prescribed in this document) |
| Access | Write by shipper only; read by audit department. Admins also cannot delete |
| External anchor | Daily independent storage of head hash. Cross-check on verification |
| Verification frequency | Weekly `verify` + anchor cross-check. Record results too |

---

## 6. Separation of duties (backend side)

| Asset | Who can change | Approval | Who verifies |
|---|---|---|---|
| Container image · Dockerfile | Platform role | 2 people (including information-security officer) | Audit department (digest check) |
| nftables · proxy allowlist | Network role | 2 people | Audit department |
| `/opt/ssdm/hooks/` · `SHA256SUMS` | Platform role | 2 people | `verify-hooks.sh` (automatic) + audit department |
| WORM retention settings | Immutable (fixed at set time) | — | Audit department |
| External anchor | Audit department only | — | External audit |

---

## 7. Verification (additions to Standard BE-8–14)

| ID | Procedure | Expected result |
|---|---|---|
| BE-15 | Inside isolation: `getent hosts example.com`, `curl -m 5 https://example.com` | Name resolution fails / unreachable |
| BE-16 | Inside isolation: `env \| grep -iE 'token\|secret\|key'`, `ls ~/.ssh ~/.aws` | No matches |
| BE-17 | `echo x >> /workspace/CLAUDE.md`, `touch /workspace/.claude/x` | `Read-only file system` |
| BE-18 | `cat /proc/1/status \| grep Cap`, equivalent of `capsh --print` | No effective capabilities |
| BE-19 | Place a script under `/tmp` and execute | `Permission denied` (noexec) |
| BE-20 | POST via proxy to a non-allowed path | Blocked; recorded in proxy logs |
| BE-21 | POST a body containing D3 patterns to an allowed host | Blocked by E-3 |
| BE-22 | `audit-chain.sh verify` + external-anchor cross-check | Match |
| BE-23 | Delete a WORM log with admin privileges | Denied |
| BE-24 | Simulate known sandbox-escape techniques (e.g. `working_directory` abuse, symlink) | Even if the tool sandbox is escaped, does not leave the container or the network |
| BE-25 | Confirm `--network none` / `cap-drop ALL` / `read-only` via `docker inspect` | All configured |

---

## 8. Unverified items

- nftables rule syntax (`nft` unused in the author’s environment)
- Validity of the additional seccomp deny set (minimal set is [Hypothesis])
- Selection criteria for gVisor / Kata / Firecracker (decide by org isolation requirements)
- Whether Cursor IDE’s in-Dev-Container sandbox (bwrap / Landlock) works in unprivileged containers. Official forum staff reply says “same behavior” but does not address unprivileged-container constraints
- Inspection-proxy product selection and D3 pattern definitions
- Concrete external-anchor means (signed timestamp, separate ledger) are chosen by the organization
