---
title: "Semantic Security Defense Module — Backend Standard"
module: "SSDM-BE"
tier: "Standard"
version: "1.0"
pairs_with: "01-semantic-security-module-standard.md"
layer: "バックエンド: 物理強制・実行環境・インフラ制御層"
scope: "組織利用でのプロキシ設定、実行フック（Pre-hook）、ログ保管"
inherits: "Backend Basic の全項目を包含し、追加分のみ記載"
target_agents: "Claude Code / Cursor (IDE・CLI) / Codex CLI"
last_updated: "2026-09-19"
timezone: "JST"
status: "Draft for review"
---

## 0. Basic からの差分

| 観点 | Basic | Standard |
|---|---|---|
| 設定の所在 | リポジトリ・個人 | **組織配布（managed）**。個人・プロジェクトで解除不能 |
| ネットワーク | ツールのドメイン許可 | **組織プロキシ**を経由させ、許可リストとログを組織側で持つ |
| フック | リポジトリ内の補助スクリプト | **組織配布の Pre-hook**。ハッシュ固定、個人フック無効化 |
| ログ | なし | **ツール呼出単位の監査ログを組織ストレージへ転送**、保存期間を規定 |
| 自己設定保護 | ツールの保護パス | + **OS の不変属性**、または別ユーザー実行 |

---

## 1. 適用条件

フロントエンド Standard と同じ（D2、P1–P3、責任者が別に存在）。

---

## 2. 統制の全体像

```text
[開発者端末]
  ├─ managed settings（MDM / managed-settings.json / サーバ管理）← 組織が配布、端末側で変更不可
  ├─ /opt/ssdm/hooks/*.sh（root 所有、SHA256SUMS 固定）
  ├─ エージェント（Claude Code / Cursor / Codex）
  │     ├─ Pre-hook → 遮断 / 通過
  │     ├─ サンドボックス（FS・ネットワーク）
  │     └─ Post-hook → 監査ログ（端末一次バッファ）
  └─ ログ転送エージェント（fluent-bit / rsyslog 等）
            │ TLS
            ▼
[組織プロキシ]  許可ドメインのみ通過、全リクエスト記録
            ▼
[ログ基盤]  保存期間・閲覧権限を規程で管理
```

---

## 3. 自己設定保護の強化（OS 層）

Basic のパーミッションでは同一ユーザーのエージェントを止められない。Standard では次のいずれかを必須とする。

| 方式 | 実装 | 長所 | 短所 |
|---|---|---|---|
| A. 不変属性 | Linux: `sudo chattr +i CLAUDE.md .claude/settings.json .mcp.json`。macOS: `chflags uchg`（root なら `schg`） | 単純。root 以外は解除不能 | 正規の更新にも root 操作が要る |
| B. 別ユーザー実行 | エージェントを専用ユーザー `agent` で起動。ルール・設定は `admin` 所有 `644`、リポジトリは `agent` 書込可 | 権限分離が OS 標準機構 | IDE 統合（Cursor）では実装が難しい |
| C. managed 側で完結 | Claude Code: `allowManagedPermissionRulesOnly` + `allowManagedHooksOnly` + `ConfigChange` フックで個人設定変更を遮断 | 配布だけで済む | Claude Code 限定 |

【仮説】Claude Code 中心の組織は C、Cursor 併用の組織は A を併用するのが現実的。

---

## 4. 組織プロキシ

### 4-1. 要件

| # | 要件 | 根拠 |
|---|---|---|
| PX-1 | エージェント由来の全 HTTP(S) を組織プロキシ経由にする。直接接続は遮断 | データ持出し経路の一本化 |
| PX-2 | 既定拒否。許可はドメイン単位の明示リスト（モデル API、パッケージレジストリ、社内 Git） | 最小化 |
| PX-3 | リクエスト単位で「日時・ユーザー・端末・宛先・メソッド・サイズ」を記録。本文は記録しない | 追跡可能性と D2 保護の両立 |
| PX-4 | TLS の内容検査は Standard では**任意**（Advanced で必須） | Claude Code 組込プロキシは既定で TLS を検査しない |

### 4-2. Claude Code での接続

| 設定 | 値 | 備考 |
|---|---|---|
| `env.HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` | 組織プロキシ | 公式ドキュメント: 設定ファイルの `env` ブロックに置くとバックグラウンドエージェントにも効く。サンドボックスは許可リストを適用した後、許可された接続を上流プロキシへトンネルする |
| `sandbox.network.allowedDomains` | 組織の許可リスト | |
| `sandbox.network.allowManagedDomainsOnly` | `true` | managed 以外の追加を無効化 |
| `sandbox.network.httpProxyPort` / `socksProxyPort` | 独自プロキシを使う場合 | 「復号・検査・記録・既存基盤との統合」用途と公式に記載 |

### 4-3. Cursor / Codex

| ツール | 設定 |
|---|---|
| Cursor IDE | 管理ダッシュボードで Sandbox network policy を配布。`sandbox.json` の許可ドメインを組織値で固定。プロキシは環境変数 |
| Cursor CLI | プロキシは環境変数と設定ファイル（公式ドキュメントに記載）。詳細キーは【要確認】 |
| Codex CLI | `network_access = false` を既定。必要時のみ承認。プロキシ設定キーは【要確認】 |

### 4-4. 許可ドメインの初期リスト（例。組織で確定する）

| 用途 | ドメイン |
|---|---|
| モデル API | 利用するプロバイダの API ホスト |
| パッケージ | `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org` |
| ソース管理 | 社内 Git ホスト。`github.com` を許可する場合、公式警告どおり持出し経路になりうることを認識する |

---

## 5. 実行フック（Pre-hook）

### 5-1. 配置と改ざん防止

| 項目 | 規定 |
|---|---|
| 配置先 | `/opt/ssdm/hooks/`（root 所有、`755`、書込不可） |
| 完全性 | `SHA256SUMS` を同梱し、`SessionStart` フックで照合（フロントエンド Advanced §3-3 の `verify-hooks.sh` を Standard でも採用） |
| 個人フックの無効化 | Claude Code: `allowManagedHooksOnly: true` |
| 設定変更の検知 | `ConfigChange` フック（`user_settings\|project_settings\|local_settings`）で exit 2 遮断 + ログ |

### 5-2. Pre-hook の構成

| フック | イベント | 役割 | 実体 |
|---|---|---|---|
| `guard-bash.sh` | `PreToolUse` / `Bash` | 取得物のシェル直結、復号実行、秘密読取、設定書換の遮断 | フロントエンド Basic §3-2（動作確認済み） |
| `guard-mcp.sh` | `PreToolUse` / `mcp__.*` | 許可リスト外 MCP サーバの遮断 | §5-3 |
| `verify-hooks.sh` | `SessionStart` | フックのハッシュ検証 | フロントエンド Advanced §3-3 |

### 5-3. `guard-mcp.sh`

```bash
#!/bin/bash
# guard-mcp.sh — PreToolUse(mcp__*) 用。許可リストにないサーバのツール呼出を遮断。
set -u
ALLOW="${SSDM_MCP_ALLOW:-/opt/ssdm/mcp-allow.txt}"   # 1 行 1 サーバ名。root 所有。
tool=$(jq -r '.tool_name // empty' 2>/dev/null) || { echo "guard-mcp: 入力不正" >&2; exit 2; }
server=$(printf '%s' "$tool" | sed -E 's/^mcp__([^_]+(_[^_]+)*)__.*/\1/')
[ -f "$ALLOW" ] || { echo "guard-mcp: 許可リストなし。全 MCP を遮断" >&2; exit 2; }
grep -qxF "$server" "$ALLOW" && exit 0
echo "guard-mcp: MCP サーバ '$server' は許可リストにない" >&2; exit 2
```

注: ツール名は `mcp__<server>__<tool>` 形式。サーバ名にアンダースコアが含まれる場合の分解は境界が曖昧なため、**許可リストは完全一致で運用し、導入時に実ツール名で検証する**。

### 5-4. Claude Code managed settings（フック部分）

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

公式ドキュメントの注意点を反映:
- `PreToolUse` のコマンドフックはタイムアウトすると**遮断しない**（通常の権限フローへ進む）。フックは軽量に保ち、タイムアウトに依存した遮断をしない。
- `if` フィルタはベストエフォート。**ハードな allow/deny は permissions で行い、フックは補助**とする。
- exit 1 は遮断にならない。遮断は exit 2 のみ。

### 5-5. Cursor / Codex

| ツール | Pre-hook 相当 |
|---|---|
| Cursor IDE | フック機構は公式ドキュメントに記載なし【明示なし】。`permissions.json` の `block_instructions` と管理ダッシュボードの global `permissions.json` で代替 |
| Cursor CLI | 【明示なし】 |
| Codex CLI | `[projects."<path>"] trust_level = "untrusted"` で実行ポリシー規則外のコマンドを承認必須化。`requirements.toml` による組織制約は【要確認】 |

---

## 6. ログ保管

### 6-1. 記録対象

| 種別 | 生成元 | 記録項目 | 記録しないもの |
|---|---|---|---|
| ツール呼出ログ | `audit-log.sh`（Post-hook） | 時刻、ユーザー、端末、session_id、prompt_id、ツール名、コマンド/パス/URL、失敗時のエラー | **ツール出力本文、ファイル内容** |
| 遮断ログ | `guard-*.sh` の stderr → `audit-log.sh` の `PostToolUseFailure` | 遮断理由 | — |
| 設定変更ログ | `ConfigChange` フック | 変更元（user/project/local） | — |
| ネットワークログ | 組織プロキシ | 宛先、メソッド、サイズ | 本文 |
| 完全性検証ログ | `verify-hooks.sh` | 結果 | — |

### 6-2. 保管

| 項目 | 規定 |
|---|---|
| 端末側 | `~/.ssdm/audit.jsonl`、`600`。一次バッファであり、転送後に一定期間で削除可 |
| 転送 | fluent-bit / rsyslog 等で TLS 転送。転送間隔は組織規程で定める |
| 組織側 | 追記専用ストレージ。閲覧は監査・セキュリティ担当に限定 |
| 保存期間 | 組織のログ管理規程に従う。AI 事業者ガイドライン C-6 ①は「事故等の原因究明、再発防止策の検討、損害賠償責任要件の立証上の重要性等を踏まえて、記録方法、頻度、保存期間等について検討する」と規定（具体的期間は定めていない） |
| 相関 | `session_id` / `prompt_id` は Claude Code の OpenTelemetry イベント属性と一致。OTel 経路を使う場合は `OTEL_*` 変数を managed の `env` で配布 |

### 6-3. fluent-bit 設定例（端末側）

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

ホスト名・ポート・パーサ名は環境に合わせる。転送失敗時の再送は `DB` によるオフセット管理で担保。

---

## 7. 検証（Basic BE-1〜7 に追加）

| ID | 手順 | 期待結果 |
|---|---|---|
| BE-8 | 個人設定に `allow: ["Bash(rm -rf *)"]` を書く | managed deny が優先。`ConfigChange` で遮断・記録 |
| BE-9 | `/opt/ssdm/hooks/guard-bash.sh` を 1 バイト改変 | `verify-hooks.sh` がセッション停止 |
| BE-10 | 許可外ドメインへ `curl` | サンドボックス遮断。プロキシに到達しない |
| BE-11 | 許可ドメインへ `curl` | 成功し、プロキシログに記録 |
| BE-12 | 許可外 MCP サーバのツール呼出 | `guard-mcp.sh` で遮断 |
| BE-13 | セッション後、組織ログ基盤で当該 session_id を検索 | 全ツール呼出が存在。出力本文なし |
| BE-14 | `chattr +i` した `CLAUDE.md` へエージェントが追記 | 失敗（方式 A の場合） |

---

## 8. 未検証事項

- Cursor CLI のプロキシ設定キー、Codex のプロキシ設定キー
- Codex `requirements.toml` の書式
- `guard-mcp.sh` のサーバ名分解（アンダースコアを含むサーバ名）
- fluent-bit 設定は構文のみ。実環境での動作は未確認
