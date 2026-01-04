# Agent Skills（agent_skills）ベストプラクティス

Agent Skills（Agent Skills / `SKILL.md` 形式）を使って、エージェントに「再利用可能な手順・知識・補助スクリプト」を安全かつ効率よく提供するためのベストプラクティスをまとめます。

## 目次

1. [Agent Skills とは](#agent-skills-とは)
2. [スキルのディレクトリ設計](#スキルのディレクトリ設計)
3. [SKILL.md（frontmatter）のベストプラクティス](#skillmdfrontmatterのベストプラクティス)
4. [SKILL.md（本文）のベストプラクティス](#skillmd本文のベストプラクティス)
5. [references/ のベストプラクティス](#references-のベストプラクティス)
6. [scripts/ のベストプラクティス](#scripts-のベストプラクティス)
7. [セキュリティのベストプラクティス](#セキュリティのベストプラクティス)
8. [エージェントへの統合（クライアント実装）](#エージェントへの統合クライアント実装)
9. [検証と運用（skills-ref）](#検証と運用skills-ref)
10. [よくある落とし穴と回避](#よくある落とし穴と回避)
11. [チェックリスト](#チェックリスト)

---

## Agent Skills とは

Agent Skills は、スキル（= フォルダ）として知識や手順をファイルにまとめ、エージェントが必要なときに読み込んで利用できるようにするオープン形式です。

重要な考え方は **progressive disclosure（段階的開示）** です。

- **Discovery（起動時）**: すべてのスキルの `name` / `description` だけを読み、どのスキルが使えそうかを判断できる状態にする
- **Activation（必要時）**: 該当スキルの `SKILL.md` 全体を読み、具体手順を実行する
- **Execution（必要時）**: `references/` を追加で読む、`scripts/` を実行する、など「必要な分だけ」使う

この前提に合わせて、スキル作者は **検索されるためのメタデータ** と **実行されるための手順・資材** を設計します。

---

## スキルのディレクトリ設計

### 置き場所（クライアント別の一般形）

Agent Skills 自体はオープン標準ですが、どこに置くと自動検出されるかはクライアントごとに差があります。

- GitHub Copilot / VS Code（Copilot）: `.github/skills/<skill-name>/SKILL.md` が推奨
- 互換のために `.claude/skills/` を読む実装もある（レガシー扱いのことが多い）
- Claude Code: 共有用のプロジェクトスキルは `.claude/skills/`、個人用は `~/.claude/skills/` を使う

まずは **配布したい対象（Copilot / Claude Code / Cursor など）** を決め、そのクライアントの検出パスに合わせて配置します。

### 最小構成

スキルは最低でも `SKILL.md` を含むディレクトリです。

```
my-skill/
└── SKILL.md
```

### 推奨構成（任意ディレクトリ）

仕様では、追加のディレクトリを任意で置けます。

- `scripts/`: 実行可能コード（Python/Bash/JSなど。対応言語は実装依存）
- `references/`: 追加ドキュメント（必要時に読む）
- `assets/`: テンプレート、図、データなど（必要時に読む）

```
my-skill/
├── SKILL.md
├── scripts/
├── references/
└── assets/
```

### ディレクトリ名と `name` の一致

`SKILL.md` の `name` は、親ディレクトリ名と一致させます（互換クライアントでの発見・検証のため）。

---

## SKILL.md（frontmatter）のベストプラクティス

`SKILL.md` は YAML frontmatter + Markdown 本文で構成されます。

```markdown
---
name: pdf-processing
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDFs or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing
...
```

### `name`

- **小文字 + 数字 + ハイフン**のみにする
- 先頭/末尾ハイフン、連続ハイフンは避ける
- 1〜64文字に収める
- 親ディレクトリ名と一致させる
- スキル群で命名パターンを揃える（探しやすさ・会話での参照性が上がる）

命名例（わかりやすい活動名）:

- `pdf-processing`
- `testing-code`
- `writing-documentation`

避けたい例:

- `utils`, `helper`, `tools`（曖昧）
- 大文字混在や記号混在（検証に落ちやすい）

※ 一部実装（例: Anthropic系）では予約語やXMLタグ等を避ける追加ルールがあるため、配布先の実装要件も確認してください。

### `description`

`description` は discovery の精度を大きく左右します。

- **「何をするか」+「いつ使うか」** を1フィールドで書く
- **具体キーワード**（ファイル拡張子、対象システム名、成果物名、操作名）を含める
- 視点は一貫させる（第三者記述が無難）
- 曖昧な短文は避ける（例: “Helps with documents.”）

良い例（何を + いつ）:

- 「PDFからテキストとテーブルを抽出し、フォーム入力や結合を行う。PDF、フォーム、ドキュメント抽出が話題のときに使う」

### `compatibility`（任意）

- **本当に環境要件がある場合のみ**書く
- 依存コマンド/ネットワーク要件/想定クライアントなどを短く列挙する

例:

- `compatibility: Requires git, docker, jq, and internet access`

### `metadata`（任意）

- author / version / ownership などを入れる
- キー名は衝突しにくい形にする（例: `org.example/version` のような命名も検討）

### `allowed-tools`（任意・実装依存）

ツール実行を伴うスキルは、事前許可の範囲を明示できる場合があります。

- **最小権限（allowlist）** を基本にする
- 具体的なツール名/サブコマンドスコープを絞る
- 実装によりサポート状況が異なるため、運用環境で実際に効くかを確認する

---

## SKILL.md（本文）のベストプラクティス

### 1. 簡潔さを最優先する

`SKILL.md` が読み込まれた後は、本文のトークンが会話履歴などと競合します。

- モデルが既知の前提（一般知識、基礎説明）は書き過ぎない
- 目的は「説明」ではなく「実行の成功率を上げること」

### 2. 自由度を設計する（壊れやすさに合わせる）

- **高自由度**: レビュー、調査、要約など（手順の骨子だけ）
- **中自由度**: テンプレート/擬似コード/引数で調整できるスクリプト
- **低自由度**: 事故が起きやすい操作（移行、削除、権限変更など）は「このコマンドをそのまま実行」のようにガードレールを強める

### 3. 推奨セクションを揃える

本文に形式制約はありませんが、次の情報があると成功率が上がります。

- **When to use**: どんな入力/状況で起動するべきか
- **Inputs/Outputs**: 期待する入出力（ファイル名、フォーマット例）
- **Step-by-step**: 手順（番号付き）
- **Examples**: 入力→出力の具体例
- **Edge cases**: 例外・失敗時の分岐
- **Verification**: 検証（lint/validate/差分確認/ドライランなど）

### 4. 複雑な作業はチェックリスト化する

多段のワークフローは、チェックリストで「抜け」を防ぎます。

```markdown
## Workflow

進捗チェックリスト:
- [ ] Step 1: 入力を確認
- [ ] Step 2: プラン（変更案）を作成
- [ ] Step 3: 検証スクリプトでチェック
- [ ] Step 4: 実行
- [ ] Step 5: 出力を検証
```

### 5. 参照は「1階層」までにする

深い参照チェーン（`SKILL.md` → `advanced.md` → `details.md`）は、部分読み（`head` など）を誘発して情報欠落の原因になります。

---

## references/ のベストプラクティス

`references/` は「必要なときだけ読む」前提で分割します。

- 1ファイル1テーマ（小さく、探しやすく）
- 100行を超える参考資料には **目次（Contents）** を付ける
- ドメイン別（finance/sales/product 等）に整理し、不要な文脈を読ませない

例:

```
bigquery-skill/
├── SKILL.md
└── references/
    ├── finance.md
    ├── sales.md
    └── product.md
```

---

## scripts/ のベストプラクティス

仕様では scripts は「実行可能コード」を置く場所です。

### 1. スクリプトは自己完結 or 依存を明記

- 必要パッケージ（PyPI / npm 等）を `SKILL.md` に明記する
- 実行環境（ネットワーク可否、インストール可否）を想定して設計する

### 2. “Solve, don’t punt”（エラー処理を丸投げしない）

- エラー条件を握りつぶさず、**具体的なメッセージ** を出す
- 可能なら代替手段を提示する

### 3. 中間成果物を「検証可能」にする

複雑なバッチ処理は、

- まず `plan.json` などの中間出力を作る
- それを `validate_*.py` で検証
- OKなら適用

という段階に分けると安全です。

### 4. マジックナンバーを避け、根拠を書く

タイムアウトやリトライ回数は根拠をコメントや定数名に反映します。

---

## セキュリティのベストプラクティス

スクリプト実行を伴うスキルは、設計時点で安全策を組み込みます。

- **Sandboxing**: 可能なら隔離環境で実行（コンテナ、権限制限など）
- **Allowlisting**: 信頼されたスキルのみ実行、または `allowed-tools` 等で最小権限
- **Confirmation**: 破壊的操作（削除、移行、課金影響）前にユーザー確認
- **Logging/Audit**: 何を実行したかを記録（再現・監査）
- **パス表記**: Windows風の `\` を避け、`scripts/run.sh` のように `/` を使う

共有スキル（外部リポジトリから取得したスキル）については、少なくとも次を徹底します。

- `SKILL.md` と同梱スクリプトをレビューし、意図しない操作（外部送信、破壊的変更、過剰な権限要求）がないか確認する
- 実行前の確認ステップ（確認プロンプト、ドライラン、差分確認）をスキル側に入れる
- 自動承認（auto-approve）や許可リストは最小に保つ（クライアントの安全機能に合わせる）

---

## エージェントへの統合（クライアント実装）

Agent Skills 対応クライアントは概ね次を実装します。

### Discovery

- 設定されたディレクトリをスキャンし `SKILL.md` を持つフォルダを検出

### Metadata loading

- 起動時は `SKILL.md` の **frontmatterのみ** を読む（トークン節約）
- `name` / `description` / `path` を保持

### Prompt injection

- 利用可能スキルの一覧を system prompt に注入する
- 各スキルのメタデータは概ね **50〜100 tokens 程度**に収める

（例: XML形式）

```xml
<available_skills>
  <skill>
    <name>pdf-processing</name>
    <description>Extracts text and tables from PDF files...</description>
    <location>/path/to/skills/pdf-processing/SKILL.md</location>
  </skill>
</available_skills>
```

- ファイルシステム型（bash等が使える）では `location` が有用
- ツール型（ツール呼び出しでスキルを起動する）では `location` を省略できる場合がある

### Activation / Execution

- マッチしたスキルのみ `SKILL.md` 全体を読む
- `references/` は必要時に読む
- `scripts/` は出力のみがトークンを消費する（= 長いコードを読み込ませずに済む）

### クライアント実装の差分に注意

- Claude Code のように「スキル有効化時に確認を挟む」実装がある（安全のため）。運用に合わせて手順を設計する
- Cursor のように「スキルはエージェント判断で自動適用（常時適用/手動適用にできない）」という制約がある場合がある
- VS Code（Copilot）は preview/設定フラグが必要な場合があるため、チーム導入時は有効化手順も別途用意する

---

## 検証と運用（skills-ref）

公式の参照実装として `skills-ref` があり、次の用途に使えます。

- `skills-ref validate <path>`: `SKILL.md` の規約（命名など）を検証
- `skills-ref to-prompt <path>...`: `<available_skills>` ブロック生成
- `skills-ref read-properties <path>`: プロパティを JSON で取得

注意: 参照実装は「デモ目的」で production 利用を意図しない旨の注意があるため、運用採用時は自前実装/フォーク/固定バージョン運用などの判断が必要です。

---

## よくある落とし穴と回避

- **descriptionが曖昧**: 「何を/いつ」を入れてキーワードも含める
- **SKILL.mdが長すぎる**: 500行目安を超えそうなら `references/` に分割
- **参照チェーンが深い**: 参照は `SKILL.md` から1階層でリンクする
- **選択肢を並べすぎる**: デフォルト1つ + 例外時のみ代替案
- **スキルが読み込まれない**: `SKILL.md` のファイル名（大文字小文字）と配置場所を確認する（多くの実装で `SKILL.md` はケースセンシティブ）
- **frontmatter の YAML が壊れている**: `---` が1行目にあり（空行なし）、インデントはスペースで統一する（タブは避ける）
- **スキルが発火しない**: ユーザーが実際に言いそうなトリガー語（拡張子/システム名/成果物名）を `description` に入れ、類似スキルと差別化する
- **依存の前提が暗黙**: 必要コマンド/パッケージ、ネットワーク要件を明記
- **スクリプトが失敗を丸投げ**: 例外処理と具体的エラーメッセージ
- **危険操作が無確認**: destructive 操作前の確認、ログ、ドライラン
- **Windowsパス**: `\` を避け `/` を使う

---

## チェックリスト

### メタデータ

- `name` が規約どおり（小文字/ハイフン、長さ、ディレクトリ一致）
- `description` が具体的で「何を/いつ」を含み、キーワードが入っている

### 構造

- `SKILL.md` が簡潔で、長い詳細は `references/` に分離
- 参照は `SKILL.md` から1階層、長文referenceには目次
- ワークフローはチェックリスト化され、検証ステップがある

### スクリプト

- 依存が明記され、エラーが具体的
- 中間成果物が検証可能（validate→apply の流れ）

### セキュリティ

- 実行の権限境界（allowlist / 確認 / ログ）がある
- 破壊的操作にガードレールがある
