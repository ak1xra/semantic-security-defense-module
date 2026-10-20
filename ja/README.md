# Semantic Security Defense Module (SSDM)

> 英語版がリポジトリの主文書です。本ディレクトリ（`ja/`）は日本語訳です。英語版: [`../README.md`](../README.md)

AI コーディングエージェント（Claude Code / Cursor / Codex CLI）を安全に運用するための、**セキュリティ規範とツール設定のテンプレート集**です。
「AI に読ませるルール」と「AI が破っても止まる物理的な制限」を組み合わせ、利用規模に応じて 3 段階（Basic / Standard / Advanced）で導入できます。

- [Semantic Security Defense Module (SSDM)](#semantic-security-defense-module-ssdm)
  - [第 1 部: エンジニア向け](#第-1-部-エンジニア向け)
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
  - [第 2 部: 非エンジニア向け](#第-2-部-非エンジニア向け)
    - [1. このリポジトリでできること](#1-このリポジトリでできること)
    - [2. 誰のためのものか](#2-誰のためのものか)
    - [3. 解決する課題](#3-解決する課題)
    - [4. 主な使い方](#4-主な使い方)
    - [5. 導入で変わること](#5-導入で変わること)
    - [6. 注意点](#6-注意点)
    - [7. よくある質問](#7-よくある質問)
    - [8. 相談・運用時の確認事項](#8-相談運用時の確認事項)
  - [関連資料](#関連資料)

---



## 第 1 部: エンジニア向け



### 1. Overview


| 項目       | 内容                                                  |
| -------- | --------------------------------------------------- |
| 種別       | ドキュメント（仕様書・設定テンプレート集）。実行可能なアプリやライブラリではない            |
| 対象エージェント | Claude Code / Cursor（IDE・CLI）/ Codex CLI            |
| 派生元      | CE-MAX v1.7 Semantic Security（§-1 / §5-1 / §6 / §8） |
| ティア決定軸   | 業種ではなく **データ区分 × エージェント権限**                         |
| 文書ステータス  | `Draft for review`（各文書 v1.0、最終更新 2026-09-19）        |




### 2. Features

- **2 層防御モデル**
  - L-N（Semantic Norm Layer）: LLM が読む規範。入力分類、命令と参照の分離、侵入命令の無効化、確認必須操作の宣言
  - L-E（Enforcement Layer）: 権限設定・フック・サンドボックス・隔離環境による物理遮断
- **3 ティア構成**: 上位ティアは下位ティアを包含し、差分のみ記載
- **貼り付け可能なルールブロック**: `CLAUDE.md` / `AGENTS.md` / `.cursor/rules/*.mdc` にそのまま貼れる L-N ルール（N1〜N16）
- **ツール別設定例**: `.claude/settings.json`、managed settings、`.cursor/cli.json`、`~/.codex/config.toml`、Dev Container、seccomp、nftables
- **補助スクリプト例**: Bash ガードフック、パーミッション固定、監査ログ、ハッシュチェーン
- **検証ケース**: 導入時・設定変更時に実施する期待結果付きテスト手順



### 3. Architecture

```text
[01-semantic-norms]  フロントエンド
  L-N: エージェントの「意図」を守る
  - 入力分類 / 参照の非実行 / 確認要求 / 報告
        │
        │  L-N が破られても
        ▼
[02-enforcement]  バックエンド
  L-E: 操作の「結果」を守る（LLM 判定に依存しない）
  - OS 権限 / ツール deny・ask / サンドボックス
  - 組織プロキシ / 監査ログ / 隔離環境
```

原則:

- L-E を入れずに L-N だけで運用してはならない（MUST NOT）
- L-E の設定はエージェント起動前に確定し、エージェント自身は変更できない（MUST）

ティア別の適用条件:


| ティア      | データ区分             | エージェント権限                   | 承認者           | 典型例              |
| -------- | ----------------- | -------------------------- | ------------- | ---------------- |
| Basic    | D0 公開 / D1 自社内部   | P1 作業ディレクトリ読み書き / P2 限定シェル | 利用者本人         | 個人開発、小規模事業者      |
| Standard | D2 他者の個人情報・守秘対象   | P1〜P3（認証付き外部サービス書込）        | 本人とは別の責任者     | 士業、上場企業、情報システム部門 |
| Advanced | D3 要配慮個人情報・決済・要機密 | P4 本番・基幹に影響する操作            | 職務分掌された複数人＋監査 | 医療、金融、官公庁        |


統制のティア別追加範囲:


| ティア      | L-N ルール   | L-E 統制   | バックエンドの主眼                                |
| -------- | --------- | -------- | ---------------------------------------- |
| Basic    | N1〜N7     | E1〜E3    | ファイルパーミッション、CLI ツールの既定 deny/ask          |
| Standard | + N8〜N12  | + E4〜E8  | managed settings 配布、組織プロキシ、Pre-hook、ログ保管 |
| Advanced | + N13〜N16 | + E9〜E15 | コンテナ/VM 隔離、Egress 物理遮断、改ざん防止監査ログ         |




### 4. Repository Structure

```text
Semantic-Security-Defense-Module/
├── README.md                              # 英語（主）
├── 01-semantic-norms/                     # L-N 規範層（フロントエンド）
│   ├── README.md                          # ティア選択の目安
│   ├── basic.md
│   ├── standard.md
│   └── advanced.md
├── 02-enforcement/                        # L-E 物理強制層（バックエンド）
│   ├── README.md                          # ティア選択と収録物の一覧
│   ├── basic.md
│   ├── standard.md
│   └── advanced.md
└── ja/                                    # 日本語（副）
    ├── README.md
    ├── 01-semantic-norms/
    └── 02-enforcement/
```

`01-semantic-norms/` と `02-enforcement/` は同じティア名のファイル同士で対になります（例: 双方の `basic.md`）。
L-E の実装（E1〜E15）は `02-enforcement/` が正です。`01-semantic-norms/basic.md` §3 にも L-E の記載が残っていますが、`02-enforcement/` 側で移管済みと定義されています。

### 5. Requirements


| 対象                            | 要件                                                               |
| ----------------------------- | ---------------------------------------------------------------- |
| 文書の閲覧                         | Markdown ビューア（GitHub 等）                                          |
| Claude Code のガードフック           | `bash`、`jq`                                                      |
| Claude Code / Cursor のサンドボックス | macOS / Linux / WSL2。Windows ネイティブは Claude Code 非対応、Cursor は明示なし |
| Advanced の隔離環境                | Docker（gVisor / Kata Containers 推奨）または専用 VM                      |
| Standard の組織配布                | MDM またはシステムディレクトリへの `managed-settings.json` 配置手段                 |
| ビルド・依存パッケージ                   | なし（本リポジトリ自体にビルド工程はない）                                            |




### 6. Setup

本リポジトリに自動インストーラはありません。各ティア文書の手順に従い、対象プロジェクトへ手動で反映します。

1. 下表で自プロジェクトのティアを決める。データ区分と権限の **どちらか一方でも超えたら上位ティア** を選ぶ（MUST）
2. `01-semantic-norms/<tier>.md` §2 の L-N ルールブロックを、エージェントの指示ファイルに貼る
3. `02-enforcement/<tier>.md` の設定ファイル・スクリプトを対象プロジェクトに配置する
4. 両文書の検証ケースを 1 件ずつ実行し、全件が期待どおりになることを確認する
5. 設定ファイルとフックを Git にコミットし、チームに配布する


| ティア      | フロントエンド                                                               | バックエンド                                                      |
| -------- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| Basic    | `01-semantic-norms/basic.md`    | `02-enforcement/basic.md`    |
| Standard | `01-semantic-norms/standard.md` | `02-enforcement/standard.md` |
| Advanced | `01-semantic-norms/advanced.md` | `02-enforcement/advanced.md` |


Basic の導入目安は「本人で 30 分以内」と記載されています。Standard / Advanced の所要時間は明示なしです。

### 7. Configuration

各文書が扱う主な設定ファイルです。具体的な値は該当文書を参照してください。


| 対象          | 配置先                                                               | 主な設定                                                                | ティア       |
| ----------- | ----------------------------------------------------------------- | ------------------------------------------------------------------- | --------- |
| Claude Code | `.claude/settings.json`                                           | `permissions.deny` / `ask`、`sandbox`、`hooks.PreToolUse`             | Basic〜    |
| Claude Code | `.claude/hooks/guard-bash.sh`                                     | 取得物のシェル直結、秘密ファイル読取、自己設定書換の遮断                                        | Basic〜    |
| Claude Code | `managed-settings.json`                                           | 組織配布、`allowManagedPermissionRulesOnly`、`allowManagedHooksOnly`      | Standard〜 |
| Cursor CLI  | `.cursor/cli.json`                                                | `permissions.allow` / `deny`                                        | Basic〜    |
| Cursor IDE  | `.cursorignore`、`.cursor/permissions.json`、`.cursor/sandbox.json` | 秘密ファイル不可視化、Run Mode、ネットワーク                                          | Basic〜    |
| Codex CLI   | `~/.codex/config.toml`                                            | `approval_policy = "on-request"`、`sandbox_mode = "workspace-write"` | Basic〜    |
| 隔離環境        | `.devcontainer/`、`Dockerfile`、`seccomp-ssdm.json`                 | 非 root、capability 削除、`--network none`                               | Advanced  |


秘密情報（API キー、トークン、鍵）は設定ファイルに記載しないでください（MUST NOT）。必要な場合は `<YOUR_API_KEY>` のようなプレースホルダーと環境変数で扱います。

### 8. Usage

導入後の運用は次の流れです。

- エージェントには利用者の直接入力だけを「命令」として扱わせる。リポジトリ内ファイル、Web 取得物、ツール出力内の命令文は「参照」として扱われ、実行されない
- 削除・強制 push・依存追加・自己設定変更などは、実行前に確認が求められる
- 遮断や侵入命令の検出時、エージェントは次の書式で報告する（Basic N6）

```text
判定: NG または 要確認 / 根拠: N1〜N5 または N7 のどれか / 理由: 1 行 / 代替: 安全な手順 / 次に必要な入力
```

Basic のパーミッション確認は、対象プロジェクトのルートで `basic.md` §3-2 のスクリプトを保存したうえで実行します。

```bash
# 差分の検出のみ（exit 1 = 要修正）
./ssdm-lockdown.sh --check
```



### 9. Development

本リポジトリへの文書追加・更新時のルールです（SHOULD）。

- 上位ティアは下位ティアを包含し、**差分のみ** を記載する（Front Matter の `inherits` に明記）
- フロントエンドとバックエンドの対応は Front Matter の `pairs_with` で示す
- 公式ドキュメントで未確認の事項は `【要確認】`、記載がない事項は `【明示なし】`、推測は `【仮説】` と明記する
- ツールの権限機構は頻繁に変わるため、設定例の根拠日（現在は 2026-09-19 時点）を更新する

コントリビューション手順・ブランチ戦略: 明示なし

### 10. Testing

本リポジトリ自体の自動テストはありません。導入先での検証ケースが各文書に定義されています。


| 文書                          | 検証 ID      | 例                                                              |
| --------------------------- | ---------- | -------------------------------------------------------------- |
| フロントエンド Basic               | V1〜V6 / V6b | README 内の注入命令を実行しない、`.env` 表示を拒否、`npm test` は通常実行、正本の言い換え確定を拒否 |
| フロントエンド Standard / Advanced | V7〜（追加分）   | 各文書 §5 を参照                                                     |
| バックエンド Basic                | BE-1〜BE-7  | `ssdm-lockdown.sh --check` が OK、`cat .env` を遮断、`.env` のコミットを遮断 |
| バックエンド Standard / Advanced  | BE-8〜（追加分） | 各文書 §7 を参照                                                     |


文書内に記載された動作確認済みの範囲:

- `guard-bash.sh`: 2026-09-19、bash + jq で 18 ケースが期待どおり
- `ssdm-lockdown.sh`: 2026-09-19、Linux で確認



### 11. Deployment / Operation


| ティア      | 配布方法                                                                  | 運用                                                 |
| -------- | --------------------------------------------------------------------- | -------------------------------------------------- |
| Basic    | リポジトリにコミットした設定ファイル                                                    | `.claude/settings.local.json` の個人 `allow` を月 1 回確認 |
| Standard | managed settings（MDM / システムディレクトリ / 管理コンソール）、root 所有のフック＋`SHA256SUMS` | ツール呼出単位の監査ログを組織ストレージへ転送、管理責任者を 1 名定める              |
| Advanced | 隔離環境（Dev Container / VM）、検査プロキシ、WORM ログ                               | 四半期のレッドチーム評価、E 層設定変更は 2 名承認＋署名付きコミット               |




### 12. Security Notes

- L-N（ルールファイル）だけでは防御できません。LLM の判定は破られる前提で、L-E を必ず併用してください（MUST）
- `Bash(...)` の deny やガードフックは文字列照合であり、変数展開・エイリアス・別名バイナリで迂回され得ます。権限設定とサンドボックスを主、フックを副として扱います
- 自動承認モード（`bypassPermissions`、Cursor の Run Everything、Codex の `danger-full-access` / `approval_policy = "never"`）は使用しないでください（MUST NOT）
- 危険なコマンド例（`rm -rf`、`git push --force`、`curl ... | sh` など）は **遮断対象の例** として文書に記載されています。実行しないでください
- Advanced の外部規範（医療・金融・官公庁）は一部原典未確認です。適用前に法務・コンプライアンス部門が該当条項を特定してください（MUST）
- 脆弱性報告窓口（`SECURITY.md`）: 明示なし



### 13. Troubleshooting


| 症状                                                       | 原因と対処                                                                    |
| -------------------------------------------------------- | ------------------------------------------------------------------------ |
| Windows でサンドボックスが効かない                                    | Claude Code はネイティブ Windows 非対応。WSL2 で実行するか、E1〜E3 を `permissions` だけで担保する |
| `guard-bash.sh` が全コマンドを遮断する                              | `jq` が未インストールで入力 JSON を解釈できていない可能性。`jq` を導入し、`chmod +x` を確認する            |
| `ssdm-lockdown.sh --check` が鍵ファイルの NG を exit code に反映しない | 既知の仕様。`find` 内の判定はサブシェルで動くため、鍵ファイルは表示で確認する                               |
| Cursor CLI で `git push --force` が通る                      | `Shell(git)` を allow すると先頭トークン照合のため通る。L-N の N4 とリモート側のブランチ保護で止める         |
| Codex CLI でファイル単位の deny ができない                            | 同等構文は未確認（要確認）。`sandbox_mode = "workspace-write"` と `.gitignore` 運用で代替する  |




### 14. Roadmap

各文書の「未検証事項」に挙げられている確認予定の項目です。

- Codex CLI のファイル単位 deny 構文、`.codex/` 自身の保護、`requirements.toml` の詳細
- Cursor CLI のフック相当機構、作業ディレクトリ外書込の扱い、組織管理機構
- Windows ネイティブでの各サンドボックスの挙動
- pre-commit 用秘密検出ツールの選定
- 総務省「AI のセキュリティ確保のための技術的対策に係るガイドライン」原典の確認

リリース計画・バージョン方針: 明示なし

### 15. License

[Creative Commons Attribution 4.0 International (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/)。[`LICENSE`](../LICENSE) を参照。

---



## 第 2 部: 非エンジニア向け



### 1. このリポジトリでできること

AI に仕事を任せるときの「安全のための決まりごと」と「鍵のかけ方」をまとめた手引書です。

- AI に守らせるルールの文面（そのまま貼れる形）
- AI がルールを破っても、大事な情報や設定に触れられないようにする設定例
- 正しく設定できたかを確かめるチェック項目



### 2. 誰のためのものか


| 立場                | 使い方                       |
| ----------------- | ------------------------- |
| AI エージェントを使う開発者   | 自分のプロジェクトに設定を入れる          |
| 情報システム部門・セキュリティ担当 | 社内の AI 利用ルールと配布設定のたたき台にする |
| 管理職・経営者           | 自社がどの段階の対策を取るべきか判断する材料にする |
| 法務・コンプライアンス担当     | 医療・金融・行政など規制業種での適用可否を確認する |




### 3. 解決する課題

AI エージェントは、読み込んだ文書やウェブページに紛れ込んだ「悪意ある指示」に従ってしまうことがあります。その結果、次のような事故が起こり得ます。

- パスワードや顧客情報などの秘密が読み取られる、外部に送られる
- ファイルや作業履歴が消される
- AI が自分の制限を勝手に緩める

「AI に注意させる」だけでは防ぎきれないため、本リポジトリでは **ルール（AI への注意）** と **鍵（AI が破っても止まる仕組み）** の二重の備えを用意しています。

### 4. 主な使い方

扱う情報の重要度と、AI に任せる範囲で 3 段階から選びます。


| 段階       | 目安                                   | 例                        |
| -------- | ------------------------------------ | ------------------------ |
| Basic    | 公開情報や自社内だけの情報。AI の作業は手元のフォルダ内まで      | 個人開発、小さな会社の社内ツール         |
| Standard | 顧客の個人情報や守秘義務のある情報を扱う。AI が外部サービスに書き込む | 税理士・弁護士事務所、上場企業、情報システム部門 |
| Advanced | 医療情報・決済情報・行政の機密を扱う。本番システムに影響し得る      | 病院、金融機関、官公庁              |


迷ったら上の段階を選びます。選んだ段階の文書を担当エンジニアに渡し、設定とチェックを依頼してください。

### 5. 導入で変わること

- AI が危険な操作（削除、外部への送信、設定変更など）の前に、必ず人間に確認するようになる
- 文書などに紛れた不審な指示を AI が実行せず、報告するようになる
- 秘密のファイルは、AI が読もうとしても仕組みの側で止まる
- Standard 以上では、AI の操作記録が残り、後から確認できるようになる



### 6. 注意点

- 本リポジトリはまだ **レビュー待ちの草案** です。重要な業務に使う前に、社内の担当者による確認が必要です
- 対策を入れても危険がゼロになるわけではありません。定期的な見直しが必要です
- AI ツールの仕様はよく変わります。設定例は 2026 年 9 月時点の情報に基づいています
- 医療・金融・行政向けの法令やガイドラインは、一部まだ原文を確認していません。必ず法務部門に確認してください
- 利用条件は [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)（[`LICENSE`](../LICENSE) を参照）



### 7. よくある質問

**Q. プログラミングの知識がなくても使えますか？**
段階の選び方と注意点は読めば分かるように書かれています。実際の設定作業はエンジニアに依頼してください。

**Q. 導入にどれくらい時間がかかりますか？**
Basic は担当者 1 人で 30 分以内が目安です。Standard と Advanced は組織の体制によるため、目安は示されていません。

**Q. AI へのルールを書くだけでは駄目なのですか？**
駄目です。AI は巧妙な指示にだまされることがあるため、AI がルールを破っても止まる「鍵」の設定を必ず一緒に入れます。

**Q. どの AI ツールに対応していますか？**
Claude Code、Cursor、Codex CLI の 3 つです。それ以外のツールは対象外です。

**Q. 費用はかかりますか？**
本リポジトリ自体の費用は明示されていません。Advanced で必要になるサーバーやログ保管の費用は、組織の環境によって変わります。

### 8. 相談・運用時の確認事項

導入前・運用中に、社内で次の点を確認してください。

- [ ] 扱う情報の重要度（D0〜D3）と、AI に任せる範囲（P1〜P4）を決めたか
- [ ] 選んだ段階が、両方の条件を満たしているか
- [ ] 設定変更を承認する責任者を決めたか（Standard 以上）
- [ ] AI の操作記録を誰が、どれくらいの期間保管するか決めたか（Standard 以上）
- [ ] 規制業種の場合、法務・コンプライアンス部門の確認を受けたか（Advanced）
- [ ] AI ツールの更新時に、チェック項目を再実施する担当者を決めたか
- [ ] 問い合わせ・相談窓口: 明示なし

---



## 関連資料


| 資料                                                   | URL                                                                                                                                |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| c — Executive Summary                                | [https://gist.github.com/ak1xra/b48fd5b9439356b6e23dd749ae972cbc](https://gist.github.com/ak1xra/b48fd5b9439356b6e23dd749ae972cbc) |
| 01_Semantic-Security-Agent_Development-Specification | [https://gist.github.com/ak1xra/9c499555dd2ddad74642b88e70c317d8](https://gist.github.com/ak1xra/9c499555dd2ddad74642b88e70c317d8) |
| 02_Semantic-Security-Agent_System-Design             | [https://gist.github.com/ak1xra/79bdf7dc23f331afc7202ca292a47e53](https://gist.github.com/ak1xra/79bdf7dc23f331afc7202ca292a47e53) |


