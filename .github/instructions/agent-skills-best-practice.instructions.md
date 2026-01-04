---
description: 'Agent Skills（ディレクトリ/参照ファイル/スクリプト/資産）とSKILL.mdの作成ベストプラクティス（発見性・可搬性・安全性・検証性）'
applyTo: '**/SKILL.md, **/.github/skills/**, **/.claude/skills/**'
---

# Agent Skills 作成ベストプラクティス（SKILL.md / ディレクトリ構成）

この指示は、Agent Skills のディレクトリと `SKILL.md` を「発見されやすく」「読みやすく」「安全に」「検証可能に」作るためのルール集です。

## 基本原則（発見性・可搬性・最小コンテキスト）

- 1スキル = 1つの明確な能力/ワークフローに絞る（万能スキルにしない）。
- **段階的開示（progressive disclosure）**を前提に設計する。
  - 起動前: エージェントは主に `name` と `description`（メタデータ）だけでマッチングする。
  - 起動後: `SKILL.md` 本文が読み込まれる。必要なら参照ファイルを追加で読む。
- 起動前メタデータはトークン予算を食うため、短く、具体的に書く（「何をするか」+「いつ使うか」）。
- スキルは複数環境（VS Code Copilot / GitHub Copilot / Claude Code など）で使われ得る前提で、**標準仕様に寄せる**。

## 配置場所（スキル発見のためのディレクトリ）

- リポジトリスコープのスキルは原則 `.github/skills/<skill-name>/` 配下に置く。
- クライアントによっては `.claude/skills/<skill-name>/` もサポートされる。
- 1スキルは必ず単一ディレクトリに閉じる（別スキルの中身に依存しない）。

### Good Example
```text
.github/
  skills/
    api-contract-review/
      SKILL.md
      references/
        checklist.md
      scripts/
        validate-openapi.sh
      assets/
        examples/
          sample-request.json
```

### Bad Example
```text
.github/skills/api-contract-review/
  SKILL.md
../shared-skill-assets/   # スキル外の共有資産に依存（移植性/監査性が下がる）
```

## スキルディレクトリの必須/推奨構成

- 必須: スキルディレクトリ直下に `SKILL.md`。
- 任意（必要な場合のみ）:
  - `references/`: 詳細説明、チェックリスト、手順、規約、FAQ。
  - `scripts/`: 実行可能スクリプト/補助コード（最小・安全第一）。
  - `assets/`: テンプレ、例、サンプルデータ（秘密情報は入れない）。
- `SKILL.md` からの参照は、原則「同一ディレクトリ配下」に限定する。
- 参照パスは **深いネストを避ける**（読み取り漏れ/参照ミスを減らす）。
  - 可能なら「`references/<file>.md`」程度の浅さに揃える。

## `SKILL.md` のフロントマター（最重要）

- `SKILL.md` は **1行目からYAMLフロントマター**を開始する（先頭に空行やコメントを置かない）。
- インデントは **タブ禁止（スペースのみ）**。
- 必須フィールド:
  - `name`: スキル識別子
  - `description`: スキルの用途説明（発見性の鍵）
- 互換性重視の注意:
  - `description` は **単一行**にする（YAMLの複数行ブロック `|` / `>` やインデント段落は、クライアントによって未対応/非発見の原因になり得る）。
  - 自動整形（例: Prettier の折返し）で `description` が複数行にならないよう注意する。
- 任意フィールド（環境依存）:
  - `allowed-tools`: 一部クライアントでサポートされる場合がある。使う場合は最小権限で列挙する。
  - `license`, `metadata`, `compatibility` など: 仕様や運用で必要なときだけ追加する（多用しない）。

### Good Example
```markdown
---
name: api-contract-review
description: OpenAPI/Swaggerの差分レビューと破壊的変更チェックを行う。API仕様変更のPRレビュー時に使う。
metadata:
  owner: platform-team
  version: '1.0'
---

# API Contract Review

## When to use
- OpenAPI/Swaggerの変更PRをレビューするとき
- クライアント互換性が重要な変更（breaking changes）が疑われるとき
```

### Bad Example
```markdown

---
name: API Contract Review  # 空白/大文字混在で識別子が不安定
description: |
  OpenAPIをレビューします。
  いろいろなときに使ってください。
---

# API Contract Review
```

## `name` のルール（識別子・検索性）

- 小文字 + ハイフン区切りのスラッグ形式にする（例: `webapp-testing`, `api-contract-review`）。
- スキルの実体と一致させる（ディレクトリ名も `name` に揃える）。
- 汎用すぎる名前（例: `helper`, `general`, `dev`）は避ける。

## `description` のルール（発見性・適用範囲の明確化）

- 必ず「何をするか」+「いつ使うか」を1文〜2文で書く。
- トリガーワードを含める（エージェントがマッチしやすい）。
  - 例: 「PRレビュー」「OpenAPI」「破壊的変更」「マイグレーション」「インシデント」など
- 広げすぎない（「何でも手伝う」系はマッチング精度を下げる）。
- 互換性のため、折り返し/複数行にしない。

### Good Example
```yaml
description: Reactコンポーネントのアクセシビリティ（a11y）監査と修正提案を行う。UI変更のPR作成/レビュー時に使う。
```

### Bad Example
```yaml
description: あらゆるフロントエンドの問題を解決します。いつでも使えます。
```

## `SKILL.md` 本文の書き方（Anthropic系ベストプラクティス準拠）

- 簡潔に、手順は番号/チェックリストで書く。
- **自由度（degrees of freedom）を適切に制限**する。
  - 入力/出力/判断基準を明示し、曖昧なまま丸投げしない。
- 選択肢を出しすぎない（「A/B/C/D/Eから選べ」にならないようにする）。
  - 分岐は最大でも2〜3個、優先順位つき。
- 深い参照チェーンを作らない。
  - `SKILL.md` → `references/guide.md` → `references/details.md` → … のように連鎖させない。
- 長文が必要なら「目次付きの参照ファイル」に逃がす（本文は要点 + 参照先）。

## 参照ファイル（references/）の設計

- `SKILL.md` 本文は「方針/手順/成果物の定義」までに留め、詳細な規約や長い例は `references/` に置く。
- 参照ファイルには以下を入れると運用しやすい:
  - チェックリスト（レビュー観点、完了条件）
  - よくある失敗と回避策
  - 成果物テンプレ（issue/prコメント例など）
  - 目次（長文の場合）
- 参照先は明示的に指定する（「必要なら読んで」ではなく「次を読む」）。

### Good Example
```markdown
## 手順
1. まず `references/checklist.md` を読んで観点を確認する。
2. 次に、変更差分から breaking change の有無を判定する。
3. 破壊的変更がある場合は、`assets/examples/migration-note.md` を雛形に移行手順を書き起こす。
```

### Bad Example
```markdown
## 手順
- 必要に応じていろいろ調べる。
- 参照は適当に辿る。
```

## スクリプト（scripts/）と資産（assets/）の扱い

- `scripts/` は「再現性のある作業」を支えるために置く。スクリプトがないと成立しない設計にはしない（ツール非対応環境もある）。
- `assets/` はテンプレやサンプルなどの「参照用」に限定する。
- 参照時は、**ファイル名と目的**を併記して誘導する。

### Good Example
```markdown
## 実行（可能な場合）
- OpenAPI lint: `scripts/validate-openapi.sh path/to/openapi.yaml`

## 参考
- 破壊的変更メモ雛形: `assets/examples/migration-note.md`
```

### Bad Example
```markdown
- スクリプトを走らせてください（どれ？どこ？引数は？結果の見方は？が不明）
```

## セキュリティ（skills統合・メタデータ注入・スクリプト実行）

- スキル内容は **プロンプト注入（prompt injection）**の媒介になり得る。
  - 参照ファイル/資産に「指示を上書きする文言」や「秘密情報の収集」を紛れ込ませない。
- `description`/`metadata` は統合実装によりシステムプロンプト等へ注入され得る。
  - メタデータに「機密」「個人情報」「トークン」「社内URL（秘匿前提）」などを入れない。
  - メタデータは短く・安全に（必要最小限）。
- スクリプト実行はリスクが高い。以下を前提に設計する:
  - サンドボックス/隔離（可能なら）
  - 実行許可の明示（ユーザー確認が必要な運用を想定）
  - 許可リスト（信頼できるスキルのみ）
  - ログ/監査（実行内容と結果の記録）
- 秘密情報の取り扱い:
  - `assets/` や `references/` に資格情報、秘密鍵、トークン、実在の個人情報を置かない。
  - 例示が必要ならダミー値（`YOUR_TOKEN_HERE`）にする。

### Good Example
```markdown
## 安全上の注意
- 資格情報はファイルに書かない。環境変数やシークレットストアを使う。
- 破壊的コマンド（削除/公開/課金）を伴う場合は、必ずユーザー確認を取る。
```

### Bad Example
```markdown
## Tips
- まず `assets/prod-token.txt` のトークンを使ってAPIを叩く。
```

## 検証・バリデーション（必ず行う）

- 作成/更新後は、スキルが「発見できる」ことを確認する。
  - `name` と `description` が正しく読み取られること。
  - `SKILL.md` がフロントマターから開始していること。
  - `description` が複数行になっていないこと。
- 可能なら skills 仕様の検証ツール（例: `skills-ref validate <path>`）でディレクトリを検証する。
- 参照ファイルのパスが実在し、深すぎないことを確認する。

## 仕上げチェックリスト（短く・効く）

- `SKILL.md` は1行目が `---` で始まる
- `name` は小文字スラッグ、ディレクトリ名と一致
- `description` は単一行で「何をする+いつ使う」になっている
- 本文は手順/チェックリスト中心で、自由度が適切に制限されている
- 長文は `references/` に逃がし、参照は浅く明示的
- `scripts/` は最小・安全（危険操作は確認前提）
- 資格情報/個人情報/秘密の社内データが含まれていない
