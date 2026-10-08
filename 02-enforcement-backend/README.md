---
status: "Doing"
---

# 02-enforcement-backend（L-E 物理強制層）

AI コーディングエージェント（Claude Code / Cursor / Codex CLI）の操作を、**LLM の判定に頼らず物理的に止める層（L-E: Enforcement Layer）** を、3 ティアで提供します。
ファイルパーミッション、ツールの deny / ask、サンドボックス、組織プロキシ、フック、監査ログ、隔離環境の設定例と補助スクリプトを収録しています。

> L-E は「結果」を守る層です。エージェントの意図が誤っていても、秘密を読めず、外へ出せず、自分の設定を変えられない状態をつくります。L-E の設定はエージェント起動前に確定し、エージェント自身は変更できないようにします（MUST）。

## フロントエンドとの責任分界

| 層 | 守るもの | 破られたときの結果 | 担当 |
| --- | --- | --- | --- |
| [`01-semantic-security-module`](../01-semantic-security-module/)（L-N） | エージェントの「意図」。入力分類、参照の非実行、確認要求 | 誤った意図が形成される | ルールファイル |
| `02-enforcement-backend`（L-E、本フォルダ） | 操作の「結果」 | 実害が発生する | OS・ツール設定・インフラ |

`01-*` 各文書 §3 の L-E 記載は本フォルダへ移管済みです。**E1〜E15 の実装は `02-*` が正** です。

## ティアの選び方

適用条件は対になるフロントエンドと同じです。データ区分とエージェント権限の **どちらか一方でも範囲を超えたら上位ティア** を選びます（MUST）。

| ティア | 適用条件 | 防御の主体 | 主眼 |
| --- | --- | --- | --- |
| Basic | D0〜D1、P1〜P2、承認者は本人 | ローカル設定（リポジトリ・個人） | ファイルパーミッション、CLI ツールの既定 deny / ask、Git 側の保護 |
| Standard | D2、P1〜P3、本人とは別の責任者 | 組織配布（managed）。個人・プロジェクトで解除不能 | 自己設定保護の強化、組織プロキシ、Pre-hook、監査ログの保管 |
| Advanced | D3、P4、職務分掌＋監査 | 隔離環境（コンテナ / VM）そのもの。ツール設定は第二防御 | 完全隔離サンドボックス、Egress 物理遮断、改ざん防止監査ログ |

ティアが上がるごとに、次のように変わります。

| 観点 | Basic | Standard | Advanced |
| --- | --- | --- | --- |
| 設定の所在 | リポジトリ・個人 | 組織配布（managed） | 隔離環境のイメージとマウント |
| 秘密情報 | ツールが読取を拒否 | 同左＋`sandbox.credentials` を組織配布 | 隔離環境に存在しない |
| ネットワーク | ツールのドメイン許可 | 組織プロキシ経由の許可リスト | 既定は物理遮断。TLS 終端・内容検査プロキシ経由のみ許可 |
| 自己設定保護 | ツールの保護パス | ＋ OS の不変属性、別ユーザー実行、または managed で完結 | ルール・フックを読取専用でマウント |
| ログ | なし | ツール呼出単位で組織ストレージへ転送 | ハッシュチェーン＋WORM 保管＋外部アンカー |

## ファイル構成

| ファイル | 主な内容 | 継承 | 対になるフロントエンド |
| --- | --- | --- | --- |
| [`02-enforcement-backend-basic.md`](02-enforcement-backend-basic.md) | 保護対象 B-1〜B-5、パーミッション規定値、`ssdm-lockdown.sh`、ツール別の既定 deny / ask、Git 保護、BE-1〜BE-7 | — | [`01-semantic-security-module-basic.md`](../01-semantic-security-module/01-semantic-security-module-basic.md) |
| [`02-enforcement-backend-standard.md`](02-enforcement-backend-standard.md) | 自己設定保護の 3 方式、組織プロキシ PX-1〜PX-4、`guard-mcp.sh`、managed settings のフック構成、ログ保管と fluent-bit 設定例、BE-8〜BE-14 | Basic を包含し差分のみ記載 | [`01-semantic-security-module-standard.md`](../01-semantic-security-module/01-semantic-security-module-standard.md) |
| [`02-enforcement-backend-advanced.md`](02-enforcement-backend-advanced.md) | 隔離アーキテクチャ、`devcontainer.json`・`Dockerfile`・seccomp、ホスト要件、Egress 遮断（internal ネットワーク、nftables、検査プロキシ）、`audit-chain.sh`、職務分掌、BE-15〜BE-25 | Basic・Standard を包含し差分のみ記載 | [`01-semantic-security-module-advanced.md`](../01-semantic-security-module/01-semantic-security-module-advanced.md) |

## 収録している設定とスクリプト

| 種別 | 名前 | 用途 | ティア |
| --- | --- | --- | --- |
| スクリプト | `ssdm-lockdown.sh` | パーミッションと `.gitignore` の確認・修正（`--check` で差分検出のみ） | Basic |
| フック | `guard-bash.sh` | 取得物のシェル直結、復号実行、秘密読取、設定書換の遮断（実体は `01-*-basic.md` §3-2） | Basic〜 |
| フック | `audit-log.sh` | ツール呼出の監査ログ（JSON Lines。出力本文は記録しない） | Standard〜 |
| フック | `guard-mcp.sh` | 許可リスト外の MCP サーバ呼出を遮断 | Standard〜 |
| フック | `verify-hooks.sh` | 起動時にフックのハッシュを `SHA256SUMS` と照合 | Standard〜 |
| フック | `deny-all.sh` | managed 以外の設定変更などを一律遮断 | Standard〜 |
| スクリプト | `audit-chain.sh` | ハッシュチェーン付き監査ログの追記と検証 | Advanced |
| 設定 | `.claude/settings.json`、`managed-settings.json` | permissions、sandbox、hooks | Basic〜 / Standard〜 |
| 設定 | `.cursor/cli.json`、`.cursorignore`、`.cursor/permissions.json`、`.cursor/sandbox.json` | Cursor CLI・IDE の遮断 | Basic〜 |
| 設定 | `~/.codex/config.toml` | `approval_policy`、`sandbox_mode`、`network_access` | Basic〜 |
| 設定 | fluent-bit 設定例 | 監査ログの TLS 転送 | Standard〜 |
| 設定 | `.devcontainer/devcontainer.json`、`Dockerfile`、`seccomp-ssdm.json`、nftables | 隔離環境と Egress 遮断 | Advanced |

## 使い方

1. 上表で自プロジェクトのティアを決める（フロントエンドと同じティアにする）
2. 該当ティアまでの文書を Basic から順に読み、設定ファイル・フック・スクリプトを対象プロジェクトまたは組織配布先に配置する
3. 対になる `01-*-<tier>.md` の L-N ルールブロックをエージェントの指示ファイルに貼る
4. 各文書の検証ケースを 1 件ずつ実行し、全件が期待どおりであることを確認する
5. 設定とフックを Git で管理し、Standard 以上は変更に承認者を置く（Advanced は 2 名承認＋署名付きコミット）

検証ケースは上位ティアほど累積します（Basic: BE-1〜BE-7、Standard: BE-1〜BE-14、Advanced: BE-1〜BE-25）。

Basic のパーミッション確認は、対象プロジェクトのルートで次を実行します。

```bash
# 差分の検出のみ（exit 1 = 要修正）。引数なしで修正を適用
./ssdm-lockdown.sh --check
```

## 注意事項

- 3 文書とも `Draft for review`（v1.0、最終更新 2026-09-19）です。ツールの権限機構は頻繁に変わるため、導入前に公式ドキュメントで設定キーを確認してください
- エージェントは所有者と同じユーザーで動くため、Basic のパーミッションだけではエージェントの書込を止められません。Basic ではツール側の deny と保護パスで担保し、Standard 以上で不変属性・別ユーザー実行・managed 設定を追加します
- `Bash(...)` の deny やガードフックは文字列照合です。変数展開・エイリアス・別名バイナリで迂回され得るため、権限設定とサンドボックスを主、フックを副として扱います
- Claude Code の `PreToolUse` フックはタイムアウトすると遮断せず、通常の権限フローへ進みます。遮断は exit 2 のみで、exit 1 では遮断されません
- `ssdm-lockdown.sh --check` の exit code に鍵ファイルの NG は反映されません。鍵ファイルは表示で確認してください
- `audit-chain.sh` はチェーン単独では末尾行の削除を検出できません。head hash を外部アンカーとして独立保管し、検証時に照合してください（Advanced で必須）
- 未検証の項目があります。nftables の構文、seccomp の追加拒否セット、fluent-bit の実環境動作、Cursor・Codex のプロキシ設定キー、Codex `requirements.toml` の書式などです。`【要確認】` / `【仮説】` が付いた項目は導入前に実機で確認してください
- 文書内の危険なコマンド例（`rm -rf`、`git push --force`、`curl ... | sh` など）は遮断対象の例です。実行しないでください
- 全体像・トラブルシューティングはリポジトリ直下の [`README.md`](../README.md) を参照してください
