---
status: "Doing"
---

# 02-enforcement (L-E Physical Enforcement Layer)

Provides a **layer that physically stops AI coding agent (Claude Code / Cursor / Codex CLI) operations without relying on LLM judgment (L-E: Enforcement Layer)**, across 3 tiers.
It includes configuration examples and auxiliary scripts for file permissions, tool deny / ask, sandboxes, organizational proxies, hooks, audit logs, and isolation environments.

> L-E is the layer that protects “outcomes.” Even if the agent's intent is wrong, it creates a state where secrets cannot be read, cannot be sent out, and self-configuration cannot be changed. L-E settings are finalized before the agent starts, and the agent itself cannot change them (MUST).

## Responsibility Boundary with the Frontend

| Layer | What it protects | Result when broken | Owner |
| --- | --- | --- | --- |
| [`01-semantic-norms`](../01-semantic-norms/) (L-N) | The agent's “intent.” Input classification, non-execution of references, confirmation requests | Incorrect intent is formed | Rule files |
| `02-enforcement` (L-E, this folder) | The “outcome” of operations | Actual harm occurs | OS · tool settings · infrastructure |

L-E descriptions in each `01-semantic-norms/` document §3 have been migrated to this folder. **Implementations of E1–E15 are authoritative in `02-enforcement/`**.

## How to Choose a Tier

Application conditions are the same as the paired frontend. Choose the **higher tier if either** data classification or agent privileges exceeds the scope (MUST).

| Tier | Application conditions | Defense principal | Focus |
| --- | --- | --- | --- |
| Basic | D0–D1, P1–P2, approver is the user themselves | Local settings (repository · personal) | File permissions, default deny / ask for CLI tools, Git-side protection |
| Standard | D2, P1–P3, a responsible person other than the user | Organizational distribution (managed). Cannot be disabled by individuals · projects | Strengthened self-configuration protection, organizational proxy, Pre-hook, audit log retention |
| Advanced | D3, P4, segregation of duties + audit | The isolation environment (container / VM) itself. Tool settings are a second defense | Fully isolated sandbox, physical Egress blocking, tamper-resistant audit logs |

As the tier rises, the following changes.

| Aspect | Basic | Standard | Advanced |
| --- | --- | --- | --- |
| Location of settings | Repository · personal | Organizational distribution (managed) | Isolation environment image and mounts |
| Secrets | Tool refuses reads | Same as left + organizational distribution of `sandbox.credentials` | Do not exist in the isolation environment |
| Network | Tool domain allowlist | Allowlist via organizational proxy | Default is physical blocking. Allowed only via TLS-terminating · content-inspecting proxy |
| Self-configuration protection | Tool protected paths | + OS immutable attributes, execution as a different user, or completion via managed | Mount rules · hooks read-only |
| Logs | None | Transfer per tool call to organizational storage | Hash chain + WORM retention + external anchor |

## File Structure

| File | Main content | Inheritance | Paired frontend |
| --- | --- | --- | --- |
| [`basic.md`](basic.md) | Protection targets B-1–B-5, permission defaults, `ssdm-lockdown.sh`, per-tool default deny / ask, Git protection, BE-1–BE-7 | — | [`basic.md`](../01-semantic-norms/basic.md) |
| [`standard.md`](standard.md) | 3 methods of self-configuration protection, organizational proxy PX-1–PX-4, `guard-mcp.sh`, managed settings hook configuration, log retention and fluent-bit configuration example, BE-8–BE-14 | Subsumes Basic and documents only deltas | [`standard.md`](../01-semantic-norms/standard.md) |
| [`advanced.md`](advanced.md) | Isolation architecture, `devcontainer.json` · `Dockerfile` · seccomp, host requirements, Egress blocking (internal network, nftables, inspection proxy), `audit-chain.sh`, segregation of duties, BE-15–BE-25 | Subsumes Basic · Standard and documents only deltas | [`advanced.md`](../01-semantic-norms/advanced.md) |

## Included Settings and Scripts

| Type | Name | Purpose | Tier |
| --- | --- | --- | --- |
| Script | `ssdm-lockdown.sh` | Confirm · correct permissions and `.gitignore` (diff detection only with `--check`) | Basic |
| Hook | `guard-bash.sh` | Block piping fetched content directly into a shell, decryption-and-execute, secret reads, configuration rewrite (body is in `01-semantic-norms/basic.md` §3-2) | Basic〜 |
| Hook | `audit-log.sh` | Audit log of tool calls (JSON Lines. Does not record output body) | Standard〜 |
| Hook | `guard-mcp.sh` | Block MCP server calls outside the allowlist | Standard〜 |
| Hook | `verify-hooks.sh` | At startup, compare hook hashes against `SHA256SUMS` | Standard〜 |
| Hook | `deny-all.sh` | Uniformly block setting changes and similar outside managed | Standard〜 |
| Script | `audit-chain.sh` | Append and verify audit logs with hash chain | Advanced |
| Settings | `.claude/settings.json`, `managed-settings.json` | permissions, sandbox, hooks | Basic〜 / Standard〜 |
| Settings | `.cursor/cli.json`, `.cursorignore`, `.cursor/permissions.json`, `.cursor/sandbox.json` | Cursor CLI · IDE blocking | Basic〜 |
| Settings | `~/.codex/config.toml` | `approval_policy`, `sandbox_mode`, `network_access` | Basic〜 |
| Settings | fluent-bit configuration example | TLS transfer of audit logs | Standard〜 |
| Settings | `.devcontainer/devcontainer.json`, `Dockerfile`, `seccomp-ssdm.json`, nftables | Isolation environment and Egress blocking | Advanced |

## How to Use

1. Decide your project's tier using the table above (use the same tier as the frontend)
2. Read documents up through the corresponding tier in order from Basic, and place configuration files · hooks · scripts in the target project or organizational distribution destination
3. Paste the L-N rule block from the paired `01-semantic-norms/<tier>.md` into the agent's instruction file
4. Run one verification case from each document and confirm that all cases match expectations
5. Manage settings and hooks with Git; at Standard and above place an approver on changes (Advanced is 2-person approval + signed commit)

Verification cases accumulate with higher tiers (Basic: BE-1–BE-7, Standard: BE-1–BE-14, Advanced: BE-1–BE-25).

For Basic permission checks, run the following at the root of the target project.

```bash
# Diff detection only (exit 1 = needs correction). With no arguments, apply corrections
./ssdm-lockdown.sh --check
```

## Notes

- All 3 documents are `Draft for review` (v1.0, last updated 2026-09-19). Because tool privilege mechanisms change frequently, confirm setting keys in official documentation before introduction
- Because the agent runs as the same user as the owner, Basic permissions alone cannot stop agent writes. At Basic, cover with tool-side deny and protected paths; at Standard and above add immutable attributes · execution as a different user · managed settings
- `Bash(...)` deny and guard hooks are string matching. Because they may be bypassed via variable expansion · aliases · alternate binaries, treat privilege settings and sandboxes as primary, hooks as secondary
- Claude Code `PreToolUse` hooks, on timeout, do not block and proceed to the normal privilege flow. Blocking is only with exit 2; exit 1 does not block
- Key-file NG is not reflected in the exit code of `ssdm-lockdown.sh --check`. Confirm key files via display
- `audit-chain.sh` alone cannot detect deletion of the last line of the chain. Independently retain the head hash as an external anchor and compare at verification (required for Advanced)
- There are unverified items. nftables syntax, additional deny sets for seccomp, fluent-bit behavior in real environments, Cursor · Codex proxy setting keys, Codex `requirements.toml` format, etc. For items marked `[Needs verification]` / `[Hypothesis]`, confirm on an actual machine before introduction
- Dangerous command examples in the documents (`rm -rf`, `git push --force`, `curl ... | sh`, etc.) are examples of blocking targets. Do not execute them
- For the overall picture · troubleshooting, see [`README.md`](../README.md) at the repository root
