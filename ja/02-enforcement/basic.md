---
title: "Semantic Security Defense Module — Backend Basic"
module: "SSDM-BE"
tier: "Basic"
version: "1.0"
pairs_with: "01-semantic-norms/basic.md（フロントエンド: 意味論・プロンプト規範層）"
layer: "バックエンド: 物理強制・実行環境・インフラ制御層"
scope: "ローカル環境のファイルパーミッション、CLI ツールの既定 許可/拒否"
target_agents: "Claude Code / Cursor (IDE・CLI) / Codex CLI"
last_updated: "2026-09-19"
timezone: "JST"
status: "Draft for review"
---

## 0. 位置づけと責任分界

| 層 | 何を守るか | 破られたときの結果 | 担当 |
|---|---|---|---|
| フロントエンド（01-*） | エージェントの「意図」。入力分類、参照非実行、確認要求 | 誤った意図が形成される | ルールファイル |
| **バックエンド（02-*、本書）** | 「結果」。意図が誤っていても、秘密を読めず、外へ出せず、自分の設定を変えられない | 実害が発生する | OS・ツール設定・インフラ |

フロントエンド 01-* の §3「L-E Enforcement Layer」は本書へ移管する。01-* 側は L-N のみを規定し、E1〜E15 の実装は本書（02-*）が正とする。

原則: **バックエンドは LLM の判定に一切依存しない。** 設定・権限・境界はエージェント起動前に確定し、エージェントは変更できない。

---

## 1. 適用条件

フロントエンド Basic と同じ（データ区分 D0–D1、権限 P1–P2、承認者 = 本人）。

---

## 2. 保護対象と統制の対応表

| ID | 保護対象 | 脅威 | 統制（本書の節） |
|---|---|---|---|
| B-1 | 秘密情報ファイル（`.env`、鍵、認証情報） | 読取・コミット・外部送信 | §3 パーミッション、§4 ツール deny、§5 Git 保護 |
| B-2 | エージェント自身の設定・ルール・フック | 書換による自己権限拡大 | §3 パーミッション、§4 ツール deny・保護パス |
| B-3 | 作業ディレクトリ外のファイル | 読取・書込 | §4 サンドボックス |
| B-4 | ネットワーク | 取得物の実行、データ持出し | §4 サンドボックスのドメイン制御 |
| B-5 | 履歴・本番 | 破壊的操作 | §4 ask/deny、§5 ブランチ保護 |

---

## 3. ファイルパーミッション（OS 層）

### 3-1. 規定値

| 対象 | パーミッション | 理由 |
|---|---|---|
| `.env`、`.env.*`、`*.pem`、`*.key`、`*.p12`、`id_*` | `600` | 所有者以外は読めない |
| `~/.ssh`、`~/.aws` | `700` | ディレクトリ列挙も不可 |
| `CLAUDE.md`、`AGENTS.md`、`.mcp.json`、`.cursorignore`、各設定 JSON | `644` | 読取は許可、書込は所有者のみ |
| `.claude/`、`.cursor/`、`.codex/` | `755` | 同上 |
| `.claude/hooks/*.sh` | `755` | 実行可、書込は所有者のみ |

注: エージェントは所有者と同じユーザーで動くため、`600`/`644` だけでは **エージェントの書込を止められない**。B-2 の実効的な遮断は §4 のツール側 deny と保護パス、Standard 以降では別ユーザー実行・不変属性で担保する。Basic のパーミッションは「意図しない露出（他ユーザー、共有 PC、誤コミット）」の防止が目的。

### 3-2. 固定スクリプト `ssdm-lockdown.sh`

リポジトリ直下で実行。`--check` で差分のみ検出（exit 1 = 要修正）、引数なしで修正。

```bash
#!/bin/bash
# ssdm-lockdown.sh — Basic: ローカルのファイルパーミッションを固定する。リポジトリ直下で実行。
# 冪等。変更点を表示し、--check で差分検出のみ（exit 1 = 要修正）。
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
# 1) 秘密情報: 所有者のみ
for f in .env .env.* ; do fix "$f" 600 "secret"; done
find . -path ./node_modules -prune -o -path ./.git -prune -o \( -name '*.pem' -o -name '*.key' -o -name '*.p12' -o -name 'id_rsa*' -o -name 'id_ed25519*' \) -type f -print 2>/dev/null | while read -r f; do fix "$f" 600 "key"; done
[ -d "$HOME/.ssh" ] && fix "$HOME/.ssh" 700 "ssh dir"
[ -d "$HOME/.aws" ] && fix "$HOME/.aws" 700 "aws dir"
# 2) エージェント設定・ルール: 書込は所有者のみ、グループ・他者は読取のみ
for p in CLAUDE.md AGENTS.md .mcp.json .cursorignore .claude/settings.json .cursor/cli.json .cursor/permissions.json .cursor/sandbox.json; do fix "$p" 644 "agent config"; done
for d in .claude .cursor .codex .claude/hooks .cursor/rules; do fix "$d" 755 "agent dir"; done
for h in .claude/hooks/*.sh; do fix "$h" 755 "hook"; done
# 3) .gitignore に秘密パターンがあるか
if [ -f .gitignore ]; then
  for pat in '.env' '*.pem' '*.key'; do
    grep -qxF "$pat" .gitignore || { echo "NG  .gitignore に '$pat' がない"; rc=1; }
  done
else echo "NG  .gitignore がない"; rc=1; fi
[ "$mode" = "--check" ] && { [ $rc -eq 0 ] && echo "OK  all checks passed"; exit $rc; }
exit 0
```

動作確認済み（2026-09-19、Linux）: `666` の `.env` → `600`、`666` のフック → `755`、`.gitignore` 欠落パターンを NG 表示。`find` 内の `fix` はサブシェルで動くため、鍵ファイルの NG は `rc` に反映されない（表示のみ）。**`--check` の exit code は `.env`・設定・`.gitignore` の判定を反映し、鍵ファイルは表示で確認する。**
macOS は `stat -f '%Lp'` にフォールバックする。`chattr` / `chflags` は Basic では使わない（Standard §3 で扱う）。

---

## 4. CLI ツールの既定 許可/拒否

方針: **deny を先に、allow は最小に、ask を破壊的操作へ。** 自動承認モードは使わない。

### 4-1. Claude Code `.claude/settings.json`（リポジトリにコミット）

フロントエンド Basic §3-1 と同一内容。要点のみ再掲。

| 区分 | 設定 | 効果の範囲 |
|---|---|---|
| deny | `Read(./.env)`, `Read(./.env.*)`, `Read(./**/*.pem)`, `Read(./**/*.key)`, `Read(~/.ssh/**)`, `Read(~/.aws/**)` | Read ツール。**サンドボックス内シェルには効かない**ため `sandbox.credentials` を併用 |
| deny | `Edit(./.claude/**)`, `Edit(./CLAUDE.md)`, `Edit(./.mcp.json)` | Edit/Write ツール。サンドボックスの保護パスがシェル側を担当 |
| deny | `Bash(rm -rf *)`, `Bash(git push --force *)`, `Bash(git push -f *)`, `Bash(git reset --hard *)` | 文字列照合。迂回可能なのでフック `guard-bash.sh` を併用 |
| ask | `Bash(git push *)`, `Bash(npm install *)`, `Bash(pip install *)`, `Bash(curl *)`, `Bash(wget *)` | 承認プロンプト |
| sandbox | `"enabled": true` | macOS / Linux / WSL2。Windows ネイティブ非対応 |

追加（Basic でも入れる）:

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

理由: 公式ドキュメントに「既定の読取ポリシーは `~/.aws/credentials` や `~/.ssh/` の読取を許す」「組込の認証情報 deny リストは存在しない」と明記されている。

### 4-2. Cursor

| 面 | 設定 | 備考 |
|---|---|---|
| IDE: Run Mode | Auto-review または Allowlist。Run Everything 禁止 | Settings → Agent |
| IDE: Sandbox | 有効。Network mode は `sandbox.json` only | macOS Seatbelt / Linux Landlock・bwrap。Windows は記載なし【明示なし】 |
| IDE: `.cursorignore` | `.env*`, `**/*.pem`, `**/*.key`, `.cursor/rules/`, `.cursor/*.json` | ツールは常に参照。サンドボックス有効時はシェルからも不可視（Git 管理下のみ） |
| IDE: `.cursor/permissions.json` | `block_instructions` に破壊的操作・取得物実行・設定書換 | 自然言語→分類器。文字列 deny ではない |
| CLI: `.cursor/cli.json` | `deny`: `Shell(rm)`, `Shell(curl)`, `Shell(wget)`, `Read(.env*)`, `Write(.cursor/**)` | deny が allow に優先 |

### 4-3. Codex CLI `~/.codex/config.toml`

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = false
```

`danger-full-access`、`--dangerously-bypass-approvals-and-sandbox`、`approval_policy = "never"` は禁止。ファイル単位 deny の有無は【要確認】。

### 4-4. 既定値の比較

| 統制 | Claude Code | Cursor IDE | Cursor CLI | Codex CLI |
|---|---|---|---|---|
| 秘密読取遮断 | deny + `sandbox.credentials` | `.cursorignore` | `Read(.env*)` deny | 【要確認】 |
| 自己設定書換遮断 | deny + 保護パス | `.cursorignore` + 保護パス | `Write(.cursor/**)` deny | 【要確認】 |
| 作業ディレクトリ外書込 | サンドボックス | サンドボックス | 【明示なし】 | `workspace-write` |
| ネットワーク | `allowedDomains` | `sandbox.json` | `WebFetch(...)` | `network_access = false` |
| 自動承認の禁止 | `bypassPermissions` を使わない | Run Everything を使わない | — | `never` を使わない |

---

## 5. Git 側の保護（ローカルで完結する範囲）

| 対策 | 設定 | 目的 |
|---|---|---|
| `.gitignore` | `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `.claude/settings.local.json` | 誤コミット防止 |
| pre-commit フック | 秘密検出ツール（gitleaks 等）を `pre-commit` で実行 | コミット時点で遮断。ツールの選定は【明示なし】 |
| `git config` | `push.default = current`、`receive.denyNonFastForwards` は Basic ではリモート側のため対象外 | 意図しない push 先の防止 |

エージェントが `git` を実行する場合、Git フックも同じユーザー権限で動く。フックの改変は §4 の `.git/hooks` 保護（Claude Code サンドボックスは `.git` 内の `hooks/` と `config` への書込を拒否）で防ぐ。

---

## 6. 検証

| ID | 手順 | 期待結果 |
|---|---|---|
| BE-1 | `ssdm-lockdown.sh --check` | `OK all checks passed` |
| BE-2 | エージェントに `cat .env` を実行させる | 遮断（Claude Code: deny + フック。Cursor: `.cursorignore`） |
| BE-3 | エージェントに `cat ~/.ssh/id_ed25519` を実行させる | 遮断（`sandbox.credentials` / サンドボックス） |
| BE-4 | エージェントに `echo x >> .claude/settings.json` を実行させる | 遮断（保護パス） |
| BE-5 | エージェントに `curl https://example.com` を実行させる | ask プロンプト、またはサンドボックスのドメイン確認 |
| BE-6 | `.env` を `git add` してコミット | pre-commit で遮断、または `.gitignore` で無視 |
| BE-7 | `npm test` | 通常実行 |

---

## 7. 未検証事項

- Codex CLI のファイル単位 deny、自己設定保護
- Cursor CLI の作業ディレクトリ外書込の扱い
- Windows ネイティブでの各サンドボックスの挙動（Claude Code は非対応と明記、Cursor は記載なし）
- pre-commit 秘密検出ツールの選定
