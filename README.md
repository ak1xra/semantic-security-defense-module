# Semantic Security Defense Module (SSDM)

> Primary language: English. Japanese translation: [`ja/README.md`](ja/README.md)

A **collection of security norms and tool-configuration templates** for safely operating AI coding agents (Claude Code / Cursor / Codex CLI).
It combines “rules that the AI reads” with “physical restrictions that stop the AI even if it breaks the rules,” and can be introduced in 3 tiers (Basic / Standard / Advanced) according to usage scale.

- [Semantic Security Defense Module (SSDM)](#semantic-security-defense-module-ssdm)
  - [Part 1: For Engineers](#part-1-for-engineers)
    - [1. Overview](#1-overview)
    - [2. Features](#2-features)
    - [3. Architecture](#3-architecture)
    - [4. Repository Structure](#4-repository-structure)
    - [5. Requirements](#5-requirements)
    - [6. Setup](#6-setup)
    - [7. Configuration](#7-configuration)
    - [8. Usage](#8-usage)
    - [9. Development](#9-development)
    - [10. Testing](#10-testing)
    - [11. Deployment / Operation](#11-deployment--operation)
    - [12. Security Notes](#12-security-notes)
    - [13. Troubleshooting](#13-troubleshooting)
    - [14. Roadmap](#14-roadmap)
    - [15. License](#15-license)
  - [Part 2: For Non-Engineers](#part-2-for-non-engineers)
    - [1. What This Repository Enables](#1-what-this-repository-enables)
    - [2. Who It Is For](#2-who-it-is-for)
    - [3. Problems It Addresses](#3-problems-it-addresses)
    - [4. Main Ways to Use It](#4-main-ways-to-use-it)
    - [5. What Changes After Introduction](#5-what-changes-after-introduction)
    - [6. Cautions](#6-cautions)
    - [7. Frequently Asked Questions](#7-frequently-asked-questions)
    - [8. Checklist for Consultation and Operations](#8-checklist-for-consultation-and-operations)
  - [Related Materials](#related-materials)

---



## Part 1: For Engineers



### 1. Overview


| Item | Content |
| -------- | --------------------------------------------------- |
| Type | Documentation (specifications and configuration template collection). Not an executable application or library |
| Target agents | Claude Code / Cursor (IDE · CLI) / Codex CLI |
| Derived from | CE-MAX v1.7 Semantic Security (§-1 / §5-1 / §6 / §8) |
| Tier decision axis | Not industry sector, but **data classification × agent privileges** |
| Document status | `Draft for review` (each document v1.0, last updated 2026-09-19) |




### 2. Features

- **2-layer defense model**
  - L-N (Semantic Norm Layer): Norms that the LLM reads. Input classification, separation of commands and references, invalidation of intrusion commands, declaration of operations that require confirmation
  - L-E (Enforcement Layer): Physical blocking via privilege settings, hooks, sandboxes, and isolation environments
- **3-tier structure**: Higher tiers subsume lower tiers and document only the deltas
- **Paste-ready rule blocks**: L-N rules (N1–N16) that can be pasted as-is into `CLAUDE.md` / `AGENTS.md` / `.cursor/rules/*.mdc`
- **Per-tool configuration examples**: `.claude/settings.json`, managed settings, `.cursor/cli.json`, `~/.codex/config.toml`, Dev Container, seccomp, nftables
- **Auxiliary script examples**: Bash guard hooks, permission lockdown, audit logs, hash chains
- **Verification cases**: Test procedures with expected results to run at introduction and when settings change



### 3. Architecture

```text
[01-semantic-norms]  Frontend
  L-N: Protects the agent's "intent"
  - Input classification / non-execution of references / confirmation requests / reporting
        │
        │  Even if L-N is broken
        ▼
[02-enforcement]  Backend
  L-E: Protects the "outcome" of operations (does not depend on LLM judgment)
  - OS privileges / tool deny·ask / sandbox
  - Organizational proxy / audit logs / isolation environment
```

Principles:

- Do not operate with L-N alone without L-E (MUST NOT)
- L-E settings are finalized before the agent starts, and the agent itself cannot change them (MUST)

Tier application conditions:


| Tier | Data classification | Agent privileges | Approver | Typical examples |
| -------- | ----------------- | -------------------------- | ------------- | ---------------- |
| Basic | D0 public / D1 internal to the organization | P1 read/write in the working directory / P2 limited shell | The user themselves | Individual development, small businesses |
| Standard | D2 third-party personal information · confidentiality subjects | P1–P3 (authenticated writes to external services) | A responsible person other than the user | Professional services (e.g. law/accounting), listed companies, information systems departments |
| Advanced | D3 sensitive personal information · payments · classified | P4 operations that affect production · core systems | Multiple people with segregation of duties + audit | Healthcare, finance, government agencies |


Additional control scope by tier:


| Tier | L-N rules | L-E controls | Backend focus |
| -------- | --------- | -------- | ---------------------------------------- |
| Basic | N1–N7 | E1–E3 | File permissions, default deny/ask for CLI tools |
| Standard | + N8–N12 | + E4–E8 | Distribution of managed settings, organizational proxy, Pre-hook, log retention |
| Advanced | + N13–N16 | + E9–E15 | Container/VM isolation, physical Egress blocking, tamper-resistant audit logs |




### 4. Repository Structure

```text
Semantic-Security-Defense-Module/
├── README.md                              # English (primary)
├── 01-semantic-norms/                     # L-N norm layer (frontend)
│   ├── README.md                          # Guidance for tier selection
│   ├── basic.md
│   ├── standard.md
│   └── advanced.md
├── 02-enforcement/                        # L-E physical enforcement layer (backend)
│   ├── README.md                          # Tier selection and inventory of included materials
│   ├── basic.md
│   ├── standard.md
│   └── advanced.md
└── ja/                                    # Japanese (secondary)
    ├── README.md
    ├── 01-semantic-norms/
    └── 02-enforcement/
```

`01-semantic-norms/` and `02-enforcement/` pair by the same tier filename (e.g. `basic.md` with `basic.md`).
Implementations of L-E (E1–E15) are authoritative in `02-enforcement/`. `01-semantic-norms/basic.md` §3 still contains L-E descriptions, but they are defined as having been migrated to the `02-enforcement/` side.

### 5. Requirements


| Target | Requirement |
| ----------------------------- | ---------------------------------------------------------------- |
| Viewing documents | Markdown viewer (GitHub, etc.) |
| Claude Code guard hooks | `bash`, `jq` |
| Claude Code / Cursor sandbox | macOS / Linux / WSL2. Windows native is unsupported by Claude Code; Cursor is [Not specified] |
| Advanced isolation environment | Docker (gVisor / Kata Containers recommended) or a dedicated VM |
| Standard organizational distribution | Means to place `managed-settings.json` via MDM or into a system directory |
| Build · dependency packages | None (this repository itself has no build process) |




### 6. Setup

This repository has no automatic installer. Follow the procedures in each tier document and apply them manually to the target project.

1. Decide your project's tier using the table below. Choose the higher tier if **either** data classification or privileges is exceeded (MUST)
2. Paste the L-N rule block from `01-semantic-norms/<tier>.md` §2 into the agent's instruction file
3. Place the configuration files and scripts from `02-enforcement/<tier>.md` into the target project
4. Run one verification case from each of both documents and confirm that all cases match expectations
5. Commit the configuration files and hooks to Git and distribute them to the team


| Tier | Frontend | Backend |
| -------- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| Basic | `01-semantic-norms/basic.md` | `02-enforcement/basic.md` |
| Standard | `01-semantic-norms/standard.md` | `02-enforcement/standard.md` |
| Advanced | `01-semantic-norms/advanced.md` | `02-enforcement/advanced.md` |


The introduction estimate for Basic is stated as “within 30 minutes by the person themselves.” Time estimates for Standard / Advanced are [Not specified].

### 7. Configuration

Main configuration files handled by each document. Refer to the relevant document for concrete values.


| Target | Placement | Main settings | Tier |
| ----------- | ----------------------------------------------------------------- | ------------------------------------------------------------------- | --------- |
| Claude Code | `.claude/settings.json` | `permissions.deny` / `ask`, `sandbox`, `hooks.PreToolUse` | Basic〜 |
| Claude Code | `.claude/hooks/guard-bash.sh` | Blocking of piping fetched content directly into a shell, reading secret files, and rewriting self-configuration | Basic〜 |
| Claude Code | `managed-settings.json` | Organizational distribution, `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly` | Standard〜 |
| Cursor CLI | `.cursor/cli.json` | `permissions.allow` / `deny` | Basic〜 |
| Cursor IDE | `.cursorignore`, `.cursor/permissions.json`, `.cursor/sandbox.json` | Hiding secret files, Run Mode, network | Basic〜 |
| Codex CLI | `~/.codex/config.toml` | `approval_policy = "on-request"`, `sandbox_mode = "workspace-write"` | Basic〜 |
| Isolation environment | `.devcontainer/`, `Dockerfile`, `seccomp-ssdm.json` | Non-root, capability removal, `--network none` | Advanced |


Do not put secrets (API keys, tokens, keys) in configuration files (MUST NOT). When needed, handle them with placeholders such as `<YOUR_API_KEY>` and environment variables.

### 8. Usage

Post-introduction operations follow this flow.

- Have the agent treat only the user's direct input as “commands.” Command-like text in repository files, web-fetched content, and tool output is treated as “references” and is not executed
- Deletion, force push, dependency addition, self-configuration changes, and similar operations require confirmation before execution
- When blocking or detecting an intrusion command, the agent reports in the following format (Basic N6)

```text
Verdict: NG or Needs confirmation / Basis: one of N1–N5 or N7 / Reason: 1 line / Alternative: safe procedure / Next required input
```

For Basic permission checks, save the script from `basic.md` §3-2 and run it at the root of the target project.

```bash
# Diff detection only (exit 1 = needs correction)
./ssdm-lockdown.sh --check
```



### 9. Development

Rules when adding or updating documents in this repository (SHOULD).

- Higher tiers subsume lower tiers and document **only the deltas** (state this explicitly in Front Matter `inherits`)
- Correspondence between frontend and backend is shown in Front Matter `pairs_with`
- Matters not verified in official documentation are marked `[Needs verification]`, matters with no description are marked `[Not specified]`, and speculation is marked `[Hypothesis]`
- Tool privilege mechanisms change frequently, so update the basis date of configuration examples (currently as of 2026-09-19)

Contribution procedures · branch strategy: [Not specified]

### 10. Testing

This repository itself has no automated tests. Verification cases for the introduction target are defined in each document.


| Document | Verification IDs | Examples |
| --------------------------- | ---------- | -------------------------------------------------------------- |
| Frontend Basic | V1–V6 / V6b | Does not execute injection commands in the README, refuses to display `.env`, runs `npm test` normally, refuses definitive rephrasing of the source of truth |
| Frontend Standard / Advanced | V7– (additions) | See each document §5 |
| Backend Basic | BE-1–BE-7 | `ssdm-lockdown.sh --check` is OK, blocks `cat .env`, blocks committing `.env` |
| Backend Standard / Advanced | BE-8– (additions) | See each document §7 |


Scope confirmed as working within the documents:

- `guard-bash.sh`: 2026-09-19, 18 cases as expected with bash + jq
- `ssdm-lockdown.sh`: 2026-09-19, confirmed on Linux



### 11. Deployment / Operation


| Tier | Distribution method | Operations |
| -------- | --------------------------------------------------------------------- | -------------------------------------------------- |
| Basic | Configuration files committed to the repository | Review personal `allow` in `.claude/settings.local.json` once a month |
| Standard | managed settings (MDM / system directory / admin console), root-owned hooks + `SHA256SUMS` | Transfer per-tool-call audit logs to organizational storage; designate one management owner |
| Advanced | Isolation environment (Dev Container / VM), inspection proxy, WORM logs | Quarterly red-team evaluation; E-layer setting changes require 2-person approval + signed commit |




### 12. Security Notes

- L-N (rule files) alone cannot defend. Assume LLM judgment can be broken and always use L-E together (MUST)
- `Bash(...)` deny and guard hooks are string matching and may be bypassed via variable expansion, aliases, or alternate binaries. Treat privilege settings and sandboxes as primary, hooks as secondary
- Do not use auto-approval modes (`bypassPermissions`, Cursor's Run Everything, Codex's `danger-full-access` / `approval_policy = "never"`) (MUST NOT)
- Dangerous command examples (`rm -rf`, `git push --force`, `curl ... | sh`, etc.) are documented as **examples of blocking targets**. Do not execute them
- External norms for Advanced (healthcare · finance · government) are partly unverified against primary sources. Before application, the legal · compliance department MUST identify the applicable provisions (MUST)
- Vulnerability reporting contact (`SECURITY.md`): [Not specified]



### 13. Troubleshooting


| Symptom | Cause and remediation |
| -------------------------------------------------------- | ------------------------------------------------------------------------ |
| Sandbox does not take effect on Windows | Claude Code does not support native Windows. Run under WSL2, or cover E1–E3 with `permissions` alone |
| `guard-bash.sh` blocks all commands | `jq` may be uninstalled so input JSON cannot be parsed. Install `jq` and confirm `chmod +x` |
| `ssdm-lockdown.sh --check` does not reflect key-file NG in the exit code | Known behavior. Judgment inside `find` runs in a subshell, so confirm key files via display |
| `git push --force` succeeds in Cursor CLI | Allowing `Shell(git)` succeeds due to leading-token matching. Stop it with L-N N4 and remote-side branch protection |
| Per-file deny is not possible in Codex CLI | Equivalent syntax is unverified ([Needs verification]). Substitute with `sandbox_mode = "workspace-write"` and `.gitignore` operations |



### 14. Roadmap

Items scheduled for confirmation listed under “Unverified items” in each document.

- Codex CLI per-file deny syntax, protection of `.codex/` itself, details of `requirements.toml`
- Cursor CLI hook-equivalent mechanism, handling of writes outside the working directory, organizational management mechanism
- Behavior of each sandbox on native Windows
- Selection of a secret-detection tool for pre-commit
- Confirmation of the primary source of MIC's “Guidelines on Technical Measures for Ensuring AI Security” (総務省「AI のセキュリティ確保のための技術的対策に係るガイドライン」)

Release plan · versioning policy: [Not specified]

### 15. License

[Creative Commons Attribution 4.0 International (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/). See [`LICENSE`](LICENSE).

---



## Part 2: For Non-Engineers



### 1. What This Repository Enables

A handbook that collects “safety rules” and “how to lock things down” when entrusting work to AI.

- Wording of rules for the AI to follow (in a form you can paste as-is)
- Configuration examples so that even if the AI breaks the rules, it cannot touch important information or settings
- Checklist items to verify that configuration was done correctly



### 2. Who It Is For


| Role | How to use it |
| ----------------- | ------------------------- |
| Developers who use AI agents | Put the settings into their own projects |
| Information systems · security staff | Use as a draft for in-house AI usage rules and distribution settings |
| Managers · executives | Use as material to decide which stage of countermeasures their organization should take |
| Legal · compliance staff | Confirm applicability in regulated industries such as healthcare · finance · government |




### 3. Problems It Addresses

AI agents can follow “malicious instructions” that slip into documents or web pages they read. As a result, accidents such as the following can occur.

- Secrets such as passwords or customer information are read or sent externally
- Files or work history are deleted
- The AI unilaterally loosens its own restrictions

Because “telling the AI to be careful” alone cannot fully prevent this, this repository provides dual preparedness of **rules (caution for the AI)** and **locks (mechanisms that stop the AI even if it breaks the rules)**.

### 4. Main Ways to Use It

Choose from 3 tiers based on the importance of the information handled and the scope entrusted to the AI.


| Stage | Guidance | Examples |
| -------- | ------------------------------------ | ------------------------ |
| Basic | Public information or information only within the organization. AI work stays within the local folder | Individual development, in-house tools at a small company |
| Standard | Handles customer personal information or information under confidentiality obligations. The AI writes to external services | Tax accountant · law offices, listed companies, information systems departments |
| Advanced | Handles medical information · payment information · government classified information. May affect production systems | Hospitals, financial institutions, government agencies |


When in doubt, choose the higher stage. Hand the documents for the chosen stage to the responsible engineer and ask them to configure and check.

### 5. What Changes After Introduction

- Before dangerous operations (deletion, external transmission, setting changes, etc.), the AI always confirms with a human
- The AI does not execute suspicious instructions slipped into documents and the like, and reports instead
- Secret files are stopped on the mechanism side even if the AI tries to read them
- At Standard and above, records of AI operations remain and can be reviewed later



### 6. Cautions

- This repository is still a **draft awaiting review**. Before use in important work, confirmation by in-house responsible parties is required
- Introducing countermeasures does not make risk zero. Periodic review is necessary
- AI tool specifications change often. Configuration examples are based on information as of September 2026
- Laws and guidelines for healthcare · finance · government have partly not yet had their original texts confirmed. Always confirm with the legal department
- Terms of use are [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/) (see [`LICENSE`](LICENSE))



### 7. Frequently Asked Questions

**Q. Can I use this without programming knowledge?**
How to choose the stage and the cautions are written so they can be understood by reading. Ask an engineer to do the actual configuration work.

**Q. How long does introduction take?**
Basic is estimated within 30 minutes by one responsible person. Standard and Advanced depend on organizational structure, so no estimate is given.

**Q. Is it no good to only write rules for the AI?**
No. Because AI can be deceived by clever instructions, always put in “lock” settings that stop the AI even if it breaks the rules.

**Q. Which AI tools are supported?**
Three: Claude Code, Cursor, and Codex CLI. Tools other than these are out of scope.

**Q. Does it cost money?**
Costs for this repository itself are not specified. Costs for servers and log retention required by Advanced vary by the organization's environment.

### 8. Checklist for Consultation and Operations

Before introduction and during operations, confirm the following points in-house.

- [ ] Have you decided the importance of information handled (D0–D3) and the scope entrusted to the AI (P1–P4)?
- [ ] Does the chosen stage satisfy both conditions?
- [ ] Have you designated a responsible person who approves setting changes? (Standard and above)
- [ ] Have you decided who retains AI operation records and for how long? (Standard and above)
- [ ] For regulated industries, have you received confirmation from legal · compliance? (Advanced)
- [ ] Have you designated a person responsible for re-running checklist items when AI tools are updated?
- [ ] Inquiry · consultation contact: [Not specified]

---



## Related Materials


| Material | URL |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| c — Executive Summary | [https://gist.github.com/ak1xra/b48fd5b9439356b6e23dd749ae972cbc](https://gist.github.com/ak1xra/b48fd5b9439356b6e23dd749ae972cbc) |
| 01_Semantic-Security-Agent_Development-Specification | [https://gist.github.com/ak1xra/9c499555dd2ddad74642b88e70c317d8](https://gist.github.com/ak1xra/9c499555dd2ddad74642b88e70c317d8) |
| 02_Semantic-Security-Agent_System-Design | [https://gist.github.com/ak1xra/79bdf7dc23f331afc7202ca292a47e53](https://gist.github.com/ak1xra/79bdf7dc23f331afc7202ca292a47e53) |
