---
title: "Semantic Security Defense Module — Backend Basic"
module: "SSDM-BE"
tier: "Basic"
version: "1.0"
pairs_with: "01-semantic-norms/basic.md (frontend: semantics and prompt-norm layer)"
layer: "Backend: physical enforcement, execution environment, and infrastructure control layer"
scope: "Local file permissions; default allow/deny for CLI tools"
target_agents: "Claude Code / Cursor (IDE・CLI) / Codex CLI"
last_updated: "2026-09-19"
timezone: "JST"
status: "Draft for review"
---

## 0. Positioning and division of responsibility

| Layer | What it protects | Consequence if broken | Owner |
|---|---|---|---|
| Frontend (01-*) | Agent “intent.” Input classification, reference-without-execution, confirmation requests | Incorrect intent is formed | Rule files |
| **Backend (02-*, this document)** | “Outcomes.” Even if intent is wrong, secrets cannot be read, cannot leave, and own settings cannot be changed | Real harm occurs | OS, tool configuration, infrastructure |

Frontend 01-* §3 “L-E Enforcement Layer” is transferred to this document. The 01-* side specifies L-N only; implementations of E1–E15 are authoritative in this document (02-*).

Principle: **The backend does not depend on LLM judgment in any way.** Settings, permissions, and boundaries are fixed before the agent starts, and the agent cannot change them.

---

## 1. Applicability conditions

Same as Frontend Basic (data classes D0–D1, privileges P1–P2, approver = self).

---

## 2. Protected assets and control mapping

| ID | Protected asset | Threat | Control (section in this document) |
|---|---|---|---|
| B-1 | Secret files (`.env`, keys, credentials) | Read, commit, external transmission | §3 permissions, §4 tool deny, §5 Git protection |
| B-2 | The agent’s own settings, rules, and hooks | Rewrite that expands self-privilege | §3 permissions, §4 tool deny and protected paths |
| B-3 | Files outside the working directory | Read, write | §4 sandbox |
| B-4 | Network | Execution of fetched artifacts, data exfiltration | §4 sandbox domain control |
| B-5 | History / production | Destructive operations | §4 ask/deny, §5 branch protection |

---

## 3. File permissions (OS layer)

### 3-1. Prescribed values

| Target | Permission | Reason |
|---|---|---|
| `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `id_*` | `600` | Unreadable except by owner |
| `~/.ssh`, `~/.aws` | `700` | Directory listing also denied |
| `CLAUDE.md`, `AGENTS.md`, `.mcp.json`, `.cursorignore`, each settings JSON | `644` | Read allowed; write owner-only |
| `.claude/`, `.cursor/`, `.codex/` | `755` | Same as above |
| `.claude/hooks/*.sh` | `755` | Executable; write owner-only |

Note: Because the agent runs as the same user as the owner, `600`/`644` alone **cannot stop agent writes**. Effective blocking of B-2 is provided by tool-side deny and protected paths in §4, and from Standard onward by separate-user execution and immutable attributes. Basic permissions aim to prevent “unintended exposure” (other users, shared PCs, accidental commits).

### 3-2. Lockdown script `ssdm-lockdown.sh`

Run at the repository root. `--check` detects diffs only (exit 1 = needs fix); with no arguments, applies fixes.

```bash
#!/bin/bash
# ssdm-lockdown.sh — Basic: fix local file permissions. Run at repository root.
# Idempotent. Shows changes; with --check, detect diffs only (exit 1 = needs fix).
set -u
mode="${1:-apply}"; rc=0
fix() { # fix <path> <perm> <desc>
  local p="$1" want="$2" d="$3" cur
  [ -e "$p" ] || return 0
  cur=$(stat -c '%a' "$p" 2>/dev/null || stat -f '%Lp' "$p")
  if [ "$cur" != "$want" ]; then
    if [ "$mode" = "--check" ]; then echo "NG  $p  $cur -> $want  ($d)"; rc=1
    else chmod "$want" "$p" && echo "FIX $p  $cur -> $want  ($d)"; fi
  fi
}
# 1) Secrets: owner only
for f in .env .env.* ; do fix "$f" 600 "secret"; done
find . -path ./node_modules -prune -o -path ./.git -prune -o \( -name '*.pem' -o -name '*.key' -o -name '*.p12' -o -name 'id_rsa*' -o -name 'id_ed25519*' \) -type f -print 2>/dev/null | while read -r f; do fix "$f" 600 "key"; done
[ -d "$HOME/.ssh" ] && fix "$HOME/.ssh" 700 "ssh dir"
[ -d "$HOME/.aws" ] && fix "$HOME/.aws" 700 "aws dir"
# 2) Agent settings/rules: write owner-only; group and others read-only
for p in CLAUDE.md AGENTS.md .mcp.json .cursorignore .claude/settings.json .cursor/cli.json .cursor/permissions.json .cursor/sandbox.json; do fix "$p" 644 "agent config"; done
for d in .claude .cursor .codex .claude/hooks .cursor/rules; do fix "$d" 755 "agent dir"; done
for h in .claude/hooks/*.sh; do fix "$h" 755 "hook"; done
# 3) Whether .gitignore has secret patterns
if [ -f .gitignore ]; then
  for pat in '.env' '*.pem' '*.key'; do
    grep -qxF "$pat" .gitignore || { echo "NG  '$pat' missing from .gitignore"; rc=1; }
  done
else echo "NG  .gitignore missing"; rc=1; fi
[ "$mode" = "--check" ] && { [ $rc -eq 0 ] && echo "OK  all checks passed"; exit $rc; }
exit 0
```

Verified (2026-09-19, Linux): `.env` at `666` → `600`, hooks at `666` → `755`, missing `.gitignore` patterns shown as NG. Because `fix` inside `find` runs in a subshell, key-file NGs are not reflected in `rc` (display only). **The `--check` exit code reflects judgments for `.env`, settings, and `.gitignore`; confirm key files via display.**
macOS falls back to `stat -f '%Lp'`. `chattr` / `chflags` are not used in Basic (covered in Standard §3).

---

## 4. CLI tool default allow/deny

Policy: **deny first, minimize allow, ask for destructive operations.** Do not use auto-approval modes.

### 4-1. Claude Code `.claude/settings.json` (committed to the repository)

Same content as Frontend Basic §3-1. Key points restated only.

| Category | Setting | Scope of effect |
|---|---|---|
| deny | `Read(./.env)`, `Read(./.env.*)`, `Read(./**/*.pem)`, `Read(./**/*.key)`, `Read(~/.ssh/**)`, `Read(~/.aws/**)` | Read tool. **Does not affect the in-sandbox shell**, so combine with `sandbox.credentials` |
| deny | `Edit(./.claude/**)`, `Edit(./CLAUDE.md)`, `Edit(./.mcp.json)` | Edit/Write tools. Sandbox protected paths cover the shell side |
| deny | `Bash(rm -rf *)`, `Bash(git push --force *)`, `Bash(git push -f *)`, `Bash(git reset --hard *)` | String matching. Bypassable, so combine with hook `guard-bash.sh` |
| ask | `Bash(git push *)`, `Bash(npm install *)`, `Bash(pip install *)`, `Bash(curl *)`, `Bash(wget *)` | Approval prompt |
| sandbox | `"enabled": true` | macOS / Linux / WSL2. Windows native not supported |

Also include (even in Basic):

```json
{
  "sandbox": {
    "enabled": true,
    "credentials": {
      "files": [
        { "path": "~/.ssh", "mode": "deny" },
        { "path": "~/.aws", "mode": "deny" }
      ]
    }
  }
}
```

Reason: Official docs state that “the default read policy allows reading `~/.aws/credentials` and `~/.ssh/`” and that “there is no built-in credentials deny list.”

### 4-2. Cursor

| Surface | Setting | Notes |
|---|---|---|
| IDE: Run Mode | Auto-review or Allowlist. Run Everything prohibited | Settings → Agent |
| IDE: Sandbox | Enabled. Network mode is `sandbox.json` only | macOS Seatbelt / Linux Landlock・bwrap. Windows not documented [Not specified] |
| IDE: `.cursorignore` | `.env*`, `**/*.pem`, `**/*.key`, `.cursor/rules/`, `.cursor/*.json` | Always applied to tools. When sandbox is enabled, also invisible from the shell (Git-tracked only) |
| IDE: `.cursor/permissions.json` | Destructive ops, execution of fetched artifacts, and settings rewrite in `block_instructions` | Natural language → classifier. Not string deny |
| CLI: `.cursor/cli.json` | `deny`: `Shell(rm)`, `Shell(curl)`, `Shell(wget)`, `Read(.env*)`, `Write(.cursor/**)` | deny takes precedence over allow |

### 4-3. Codex CLI `~/.codex/config.toml`

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = false
```

`danger-full-access`, `--dangerously-bypass-approvals-and-sandbox`, and `approval_policy = "never"` are prohibited. Whether file-level deny exists is [Needs verification].

### 4-4. Default-value comparison

| Control | Claude Code | Cursor IDE | Cursor CLI | Codex CLI |
|---|---|---|---|---|
| Block secret reads | deny + `sandbox.credentials` | `.cursorignore` | `Read(.env*)` deny | [Needs verification] |
| Block self-settings rewrite | deny + protected paths | `.cursorignore` + protected paths | `Write(.cursor/**)` deny | [Needs verification] |
| Write outside working directory | Sandbox | Sandbox | [Not specified] | `workspace-write` |
| Network | `allowedDomains` | `sandbox.json` | `WebFetch(...)` | `network_access = false` |
| Prohibit auto-approval | Do not use `bypassPermissions` | Do not use Run Everything | — | Do not use `never` |

---

## 5. Git-side protection (scope completable locally)

| Measure | Setting | Purpose |
|---|---|---|
| `.gitignore` | `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `.claude/settings.local.json` | Prevent accidental commits |
| pre-commit hook | Run a secret-detection tool (gitleaks, etc.) via `pre-commit` | Block at commit time. Tool selection is [Not specified] |
| `git config` | `push.default = current`; `receive.denyNonFastForwards` is remote-side and out of scope for Basic | Prevent unintended push targets |

When the agent runs `git`, Git hooks also run with the same user privileges. Hook tampering is prevented by `.git/hooks` protection in §4 (Claude Code sandbox denies writes to `hooks/` and `config` under `.git`).

---

## 6. Verification

| ID | Procedure | Expected result |
|---|---|---|
| BE-1 | `ssdm-lockdown.sh --check` | `OK all checks passed` |
| BE-2 | Have the agent run `cat .env` | Blocked (Claude Code: deny + hook. Cursor: `.cursorignore`) |
| BE-3 | Have the agent run `cat ~/.ssh/id_ed25519` | Blocked (`sandbox.credentials` / sandbox) |
| BE-4 | Have the agent run `echo x >> .claude/settings.json` | Blocked (protected path) |
| BE-5 | Have the agent run `curl https://example.com` | ask prompt, or sandbox domain check |
| BE-6 | `git add` `.env` and commit | Blocked by pre-commit, or ignored by `.gitignore` |
| BE-7 | `npm test` | Runs normally |

---

## 7. Unverified items

- Codex CLI file-level deny and self-settings protection
- How Cursor CLI treats writes outside the working directory
- Behavior of each sandbox on Windows native (Claude Code documented as unsupported; Cursor not documented)
- Selection of pre-commit secret-detection tool
