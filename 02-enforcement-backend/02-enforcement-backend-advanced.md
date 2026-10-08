---
title: "Semantic Security Defense Module — Backend Advanced"
module: "SSDM-BE"
tier: "Advanced"
version: "1.0"
pairs_with: "01-semantic-security-module-advanced.md"
layer: "バックエンド: 物理強制・実行環境・インフラ制御層"
scope: "完全隔離サンドボックス（Docker/VM）、Egress 物理遮断、改ざん防止監査ログ"
inherits: "Backend Basic・Standard の全項目を包含し、追加分のみ記載"
target_agents: "Claude Code / Cursor (IDE・CLI) / Codex CLI"
last_updated: "2026-09-19"
timezone: "JST"
status: "Draft for review"
---

## 0. Standard からの差分

| 観点 | Standard | Advanced |
|---|---|---|
| 防御の主体 | ツールのサンドボックス + 組織設定 | **隔離環境（コンテナ/VM）そのもの**。ツール設定は第二防御 |
| 秘密 | ツールが読取を拒否 | **隔離環境に存在しない** |
| Egress | プロキシ経由で許可リスト | **既定は物理遮断**。許可は TLS 終端・内容検査プロキシ経由のみ |
| 監査ログ | 転送・追記専用 | **ハッシュチェーン + WORM 保管 + 外部アンカー** |
| 想定する破られ方 | 設定の緩和・迂回 | **ツールの脆弱性（例: Cursor CVE-2026-50548/50549 のサンドボックス脱出）を前提**に、脱出しても隔離環境の外へ出られない |

---

## 1. 適用条件

フロントエンド Advanced と同じ（D3、P4、職務分掌 + 監査）。

---

## 2. 隔離アーキテクチャ

```text
[開発者端末]  ── SSH / Dev Container attach のみ ──▶ [隔離環境]
   IDE 表示                                          ├─ エージェント（非 root、cap 全落とし、no-new-privileges）
   秘密なし                                          ├─ リポジトリ（作業用 volume。ルール・フックは read-only mount）
                                                     ├─ 秘密: なし（本番資格情報・D3 データは存在しない）
                                                     ├─ ネットワーク: なし（--network none）
                                                     │      例外: 検査プロキシへの一方向経路のみ（§4）
                                                     └─ 監査ログ → 読取専用 volume 外へ hash chain 付きで転送（§5）
[検査プロキシ]  TLS 終端・内容検査・全記録 ── 許可ドメインのみ ──▶ [モデル API / レジストリ]
[WORM ログ]     Object Lock / 追記専用。head hash を外部アンカーへ
```

隔離手段の選択:

| 手段 | 分離強度 | 適用 |
|---|---|---|
| Docker（標準 runc） | カーネル共有。名前空間 + seccomp + cap | 最低限。§3 の強化設定を全て適用する条件で可 |
| gVisor / Kata Containers | ユーザー空間カーネル / 軽量 VM | 推奨。Docker 互換で分離強度を上げる |
| 専用 VM（Firecracker 等） | ハイパーバイザ境界 | 官公庁・金融の要機密領域では VM を既定とする【仮説】 |

---

## 3. 完全隔離サンドボックス（Docker 実装例）

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

要点:

| 設定 | 目的 |
|---|---|
| `--network none` | Egress の物理遮断。モデル API が必要な場合は §4 の専用ネットワークに置き換える |
| `--cap-drop ALL`、`no-new-privileges` | 権限昇格の遮断 |
| `--read-only` + `--tmpfs /tmp noexec` | ルート FS 不変、一時領域で実行不可 |
| ルール・フック・設定を `readonly` mount | エージェントの自己改変を **マウント層で** 遮断 |
| `remoteUser: agent` | 非 root |
| 秘密を `mounts` に含めない、`containerEnv` にトークンを入れない | 秘密の非存在 |

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
# エージェント CLI のインストールはビルド時に固定バージョンで行う（実行時ダウンロード禁止）
USER agent
WORKDIR /workspace
```

エージェント CLI（Claude Code 等）のインストール手順は各ツールの公式手順に従い、**ビルド時に固定バージョン**で入れる。ビルドはネットワークのある CI で行い、イメージのダイジェストを固定してデプロイする。

### 3-3. `seccomp-ssdm.json`

Docker の既定 seccomp プロファイルを基に、次を追加で拒否する【仮説: 最小セット】。

| syscall | 理由 |
|---|---|
| `mount`, `umount2`, `pivot_root` | マウント操作 |
| `ptrace`, `process_vm_readv`, `process_vm_writev` | 他プロセス覗き見 |
| `unshare`, `setns` | 名前空間操作（bubblewrap を内側で使う場合は `unshare` を許可し `enableWeakerNestedSandbox` 相当の扱いになる。二重サンドボックスの要否は §3-5） |
| `keyctl`, `add_key`, `request_key` | カーネル鍵リング |
| `bpf`, `perf_event_open` | 観測・悪用 |

プロファイルの作成は Docker 公式の既定プロファイルをダウンロードし、上記を `SCMP_ACT_ERRNO` に変更する。完全な JSON は環境依存のため本書には含めない。

### 3-4. ホスト側の要件

| # | 要件 |
|---|---|
| H-1 | Docker デーモンは rootless モード、または専用ホスト。エージェントに `/var/run/docker.sock` を渡さない（公式警告: ソケット経由でホストへ到達可能） |
| H-2 | ホストにも本番資格情報を置かない。隔離環境は「秘密がないホスト」の上で動かす |
| H-3 | イメージは署名検証（cosign 等）を通したダイジェスト固定で配布 |
| H-4 | ホストの `sysctl kernel.unprivileged_userns_clone` 等は組織の基準に従う。bubblewrap を内側で使う場合のみ必要 |

### 3-5. ツール側サンドボックスとの重ね方

| 選択 | 内容 | 判断 |
|---|---|---|
| 二重（推奨） | コンテナ内で Claude Code の `sandbox.enabled: true` も有効化 | コンテナ脱出前の第一段として機能。非特権コンテナでは bwrap が `/proc` をマウントできず `enableWeakerNestedSandbox` が必要になる旨が公式に記載。これは分離を弱めるので、**コンテナ側の分離が十分な場合のみ**許容 |
| 単重 | コンテナだけに依存し、ツールサンドボックスは無効 | gVisor / VM で分離している場合は選択可 |

---

## 4. Egress 物理遮断

### 4-1. 原則

| 状態 | 設定 |
|---|---|
| 既定 | `--network none`。DNS も含め全遮断 |
| モデル API が必要 | 専用 Docker network（internal）にプロキシコンテナだけを置き、隔離環境はそのプロキシ以外へ到達不能にする |
| パッケージ取得が必要 | 社内ミラー（Nexus / Artifactory 等）をプロキシ経由で許可。公開レジストリ直結は不可 |

### 4-2. 専用ネットワーク構成

```bash
# ホスト側。隔離環境 ↔ プロキシ のみ通す internal ネットワーク
docker network create --internal ssdm-egress
# プロキシは ssdm-egress と外部ネットワークの両方に接続（ここだけが外へ出る）
docker run -d --name ssdm-proxy --network ssdm-egress -v ./proxy.conf:/etc/proxy.conf:ro <検査プロキシイメージ>
docker network connect <外部到達可能ネットワーク> ssdm-proxy
# 隔離環境は ssdm-egress のみ
# devcontainer.json の runArgs から "--network","none" を外し "--network","ssdm-egress" にする
```

`--internal` ネットワークはホスト外へルーティングされない。隔離環境からの経路はプロキシコンテナのみ。

### 4-3. ホスト nftables（プロキシコンテナの出口を絞る）

```text
table inet ssdm {
  set allowed_hosts { type ipv4_addr; flags interval; elements = { <モデルAPIのIP範囲>, <社内ミラーIP> } }
  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
    iifname "br-ssdm-egress" ip daddr @allowed_hosts tcp dport 443 accept
    iifname "br-ssdm-egress" log prefix "ssdm-egress-drop " drop
  }
}
```

IP 範囲は DNS 解決に依存せず固定する（DNS 経由の持出しを防ぐ）。ブリッジ名は `docker network inspect` で確認する。**この構文は本書執筆環境で `nft` が使えず未検証**【要確認】。

### 4-4. 検査プロキシの要件

| # | 要件 |
|---|---|
| E-1 | TLS 終端。CA はビルド時にイメージへ配布（実行時取得不可） |
| E-2 | 許可はホスト名 + パス前置きまで（例: モデル API の `/v1/messages` のみ） |
| E-3 | リクエスト本文の D3 パターン検査（個人識別子、口座番号形式等）。検出時は遮断 + 記録。パターンは組織で定義 |
| E-4 | 全リクエストを §5 のログ基盤へ転送（本文は検査結果のみ、本文そのものは保存しない） |
| E-5 | レスポンスサイズ・レートの上限（DoS と大量持出しの抑止） |

Claude Code 組込プロキシは「ホスト名で判定し TLS を検査しない」「ドメインフロンティングの余地がある」と公式に明記されているため、Advanced では**組込プロキシを Egress の最終防御にしない**。

---

## 5. 改ざん防止監査ログ

### 5-1. 構成

```text
[隔離環境] audit-chain.sh append ─▶ /var/ssdm/audit.chain.jsonl（volume、追記専用）
      │ 追記ごと、または N 秒ごと
      ▼
[転送] fluent-bit（TLS、相互認証）
      ▼
[WORM] S3 Object Lock（Compliance mode）/ 追記専用ストレージ。保持期間を固定
      │ 日次
      ▼
[外部アンカー] 当日末尾の head hash を、ログ基盤と独立した場所へ記録
               （署名付きタイムスタンプ、別組織の台帳、または監査部門保管の紙）
```

### 5-2. `audit-chain.sh`

各行が前行のハッシュを含むハッシュチェーン。`append` で追記、`verify` で全件検証。

```bash
#!/bin/bash
# audit-chain.sh — Advanced: 改ざん検知付き監査ログ。各行に前行ハッシュを連結（ハッシュチェーン）。
# 用法: append  … stdin の JSON 1 件を追記   |   verify … チェーン全体を検証（exit 1 = 改ざん/欠落）
# 端末側は一次バッファに過ぎない。追記ごとに WORM 保管（S3 Object Lock 等）へ転送する前提。
set -u
LOG="${SSDM_AUDIT_LOG:-$HOME/.ssdm/audit.chain.jsonl}"
mkdir -p "$(dirname "$LOG")"; umask 077
case "${1:-}" in
  append)
    prev=$( [ -s "$LOG" ] && tail -n1 "$LOG" | jq -r '.hash' || echo "GENESIS")
    seq=$( [ -s "$LOG" ] && tail -n1 "$LOG" | jq -r '.seq + 1' || echo 1)
    body=$(jq -c --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --argjson seq "$seq" --arg prev "$prev" \
      '{seq:$seq, ts:$ts, prev:$prev, event:.}') || { echo "audit-chain: 入力JSON不正" >&2; exit 2; }
    hash=$(printf '%s' "$body" | sha256sum | cut -d' ' -f1)
    printf '%s\n' "$(printf '%s' "$body" | jq -c --arg h "$hash" '. + {hash:$h}')" >> "$LOG" ;;
  verify)
    prev="GENESIS"; n=0
    while IFS= read -r line; do
      n=$((n+1))
      seq=$(jq -r '.seq' <<<"$line"); p=$(jq -r '.prev' <<<"$line"); h=$(jq -r '.hash' <<<"$line")
      [ "$seq" = "$n" ] || { echo "NG line $n: seq=$seq (欠落または挿入)"; exit 1; }
      [ "$p" = "$prev" ] || { echo "NG line $n: prev 不一致"; exit 1; }
      calc=$(jq -c 'del(.hash)' <<<"$line" | tr -d '\n' | sha256sum | cut -d' ' -f1)
      [ "$calc" = "$h" ] || { echo "NG line $n: hash 不一致（改ざん）"; exit 1; }
      prev="$h"
    done < "$LOG"
    echo "OK $n entries, chain intact, head=$prev" ;;
  *) echo "usage: $0 append|verify" >&2; exit 2 ;;
esac
```

動作確認済み（2026-09-19）:

| 操作 | `verify` の結果 |
|---|---|
| 正常 3 件 | `OK 3 entries, chain intact` |
| 2 行目の内容を書換 | `NG line 2: hash 不一致（改ざん）` |
| 2 行目を削除 | `NG line 2: seq=3 (欠落または挿入)` |
| **末尾 1 行を削除** | `OK 2 entries`（**チェーン単独では検出不能**） |

末尾切り詰めはチェーンだけでは検出できない。これが §5-1 の **外部アンカー（head hash の独立保管）** を必須とする理由。検証時は「保管された head hash と `verify` の head が一致するか」まで確認する。

Post-hook からの呼出は、Standard の `audit-log.sh` の出力を `audit-chain.sh append` に流す形にする（`audit-log.sh` の JSON 整形をそのまま `event` として格納）。

### 5-3. 保管要件

| 項目 | 規定 |
|---|---|
| 保管方式 | WORM。S3 Object Lock は Compliance mode（root でも保持期間内は削除不可） |
| 保持期間 | 組織規程と適用規範に従う（本書では定めない） |
| アクセス | 書込は転送エージェントのみ、読取は監査部門。管理者も削除不可 |
| 外部アンカー | 日次で head hash を独立保管。検証時に照合 |
| 検証頻度 | 週次で `verify` + アンカー照合。結果も記録 |

---

## 6. 職務分掌（バックエンド側）

| 資産 | 変更できる者 | 承認 | 検証する者 |
|---|---|---|---|
| コンテナイメージ・Dockerfile | プラットフォーム担当 | 2 名（情報セキュリティ責任者を含む） | 監査部門（ダイジェスト照合） |
| nftables・プロキシ許可リスト | ネットワーク担当 | 2 名 | 監査部門 |
| `/opt/ssdm/hooks/`・`SHA256SUMS` | プラットフォーム担当 | 2 名 | `verify-hooks.sh`（自動）+ 監査部門 |
| WORM 保持設定 | 変更不可（設定時に固定） | — | 監査部門 |
| 外部アンカー | 監査部門のみ | — | 外部監査 |

---

## 7. 検証（Standard BE-8〜14 に追加）

| ID | 手順 | 期待結果 |
|---|---|---|
| BE-15 | 隔離環境内で `getent hosts example.com`、`curl -m 5 https://example.com` | 名前解決失敗 / 到達不能 |
| BE-16 | 隔離環境内で `env \| grep -iE 'token\|secret\|key'`、`ls ~/.ssh ~/.aws` | 該当なし |
| BE-17 | `echo x >> /workspace/CLAUDE.md`、`touch /workspace/.claude/x` | `Read-only file system` |
| BE-18 | `cat /proc/1/status \| grep Cap`、`capsh --print` 相当 | 実効 capability なし |
| BE-19 | `/tmp` にスクリプトを置いて実行 | `Permission denied`（noexec） |
| BE-20 | プロキシ経由で許可外パスへ POST | 遮断、プロキシログに記録 |
| BE-21 | D3 パターンを含む本文を許可ホストへ POST | E-3 で遮断 |
| BE-22 | `audit-chain.sh verify` + 外部アンカー照合 | 一致 |
| BE-23 | WORM 上のログを管理者権限で削除 | 拒否 |
| BE-24 | 既知のサンドボックス脱出手口（例: `working_directory` 悪用、symlink）を模擬 | ツールサンドボックスを抜けても、コンテナ外・ネットワーク外へ出ない |
| BE-25 | `docker inspect` で `--network none` / `cap-drop ALL` / `read-only` を確認 | 全て設定済み |

---

## 8. 未検証事項

- nftables ルールの構文（執筆環境で `nft` 未使用）
- seccomp 追加拒否セットの妥当性（最小セットは【仮説】）
- gVisor / Kata / Firecracker の選定基準（組織の分離要件で決める）
- Cursor IDE の Dev Container 内サンドボックス（bwrap / Landlock）が非特権コンテナで動作するか。公式フォーラムのスタッフ回答は「同一動作」だが、非特権コンテナの制約には触れていない
- 検査プロキシの製品選定と D3 パターン定義
- 外部アンカーの具体的手段（署名付きタイムスタンプ、別台帳）は組織で選定
