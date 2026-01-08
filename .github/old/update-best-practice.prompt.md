---
description: '指定されたツールや技術のベストプラクティスをまとめてinstructionsファイルを作成'
argument-hint: tool, url
tools: ['read/problems', 'read/readFile', 'edit/createFile', 'edit/editFiles', 'web/fetch', 'agent', 'context7/*', 'todo', 'ms-vscode.vscode-websearchforcopilot/websearch']
---

# ベストプラクティスをまとめてinstructionsファイルを作成

特定のツールやフレームワークのベストプラクティスを公式ドキュメントおよびウェブ検索から収集し、instructionsファイルとして体系化します。

## 主な目的

指定されたツール・技術について、公式ドキュメントとウェブ上の一般的なベストプラクティスを統合し、GitHub Copilot用の高品質なinstructionsファイルを作成します。

## 対象範囲と前提条件

- 対象: パッケージマネージャー、フレームワーク、リンター、テストツールなど、開発ツール全般
- 前提: 公式ドキュメントのURLが既知であること
- 出力先: `docs/${input:tool}/` および `.github/instructions/` ディレクトリ

## 既存ファイルがある場合の扱い

- 作成対象のファイルが既に存在する場合は、新規作成ではなく**更新作業**を行ってください。
- 更新では、既存内容を尊重しつつ、重複排除・構成整理・追記・修正を行い、必要に応じて情報を最新化してください。

## 必須入力

- `${input:tool}`: ツールまたは技術の名前 (例: Bun, Biome, Vitest)
- `${input:url}`: 公式ドキュメントのベースURL (例: https://bun.com/docs)

入力が不足している場合は、ユーザーに確認してから作業を開始してください。

## 命名規則

instructions ファイル名は `${input:tool}` からケバブケースを導出して使用してください。

- 変換ルール: 小文字化 → 英数字以外は `-` に置換 → `-` の連続は1つに圧縮 → 先頭末尾の `-` を除去
- 例: `Google Cloud` → `google-cloud`、`MyTool` → `my-tool`、`my_tool` → `my-tool`

## 実行ワークフロー

次の手順を #tool:todo で管理し、各ステップを順番に実行してください:

### 1. 公式ドキュメントの探索
- **タスク**: #tool:agent/runSubagent を使用してサブエージェントに委譲し、公式ドキュメントを再帰的に探索
- **詳細**: `#tool:web/fetch を使用し ${input:tool} の公式ドキュメント ${input:url} を再帰的に探索し、ベストプラクティス作成に参考にするべきドキュメントのURLをまとめる。まとめた結果を docs/best_practice/${input:tool}/url.md に記載する`
- **検証**: `docs/best_practice/${input:tool}/url.md` が作成され、関連URLが10件以上リストされていること
- **既存ファイル**: `docs/best_practice/${input:tool}/url.md` が存在する場合は、内容を更新して反映する

### 2. URL一覧の読み込み
- **タスク**: #tool:read/readFile を使用し `docs/best_practice/${input:tool}/url.md` を読み込む
- **検証**: ファイルの内容が取得できること

### 3. 公式ドキュメントの取得
- **タスク**: #tool:web/fetch を使用し url.md に記載されたURLからウェブサイトを読み込む
- **注意**: 複数のURLがある場合は、重要度の高いものから優先的に取得する
- **検証**: 主要なドキュメントページが正常に取得できること

### 4. ベストプラクティスドキュメントの作成
- **タスク**: 読み込んだウェブサイトの内容をまとめ `docs/best_practice/${input:tool}/best_practice.md` を日本語で作成する
- **内容要件**:
  - 公式推奨の設定やパターン
  - セキュリティとパフォーマンスの考慮事項
  - よくある落とし穴と回避方法
- **検証**: ドキュメントが論理的に構造化され、実用的なガイダンスを含むこと
- **既存ファイル**: `docs/best_practice/${input:tool}/best_practice.md` が存在する場合は、内容を更新して反映する

### 5. ウェブ検索によるレビュー
- **タスク**: #tool:ms-vscode.vscode-websearchforcopilot/websearch を使用してウェブ上の一般的な ${input:tool} のベストプラクティスを検索し、作成したベストプラクティスがそれらと比較し劣る点をレビューする
- **検索キーワード例**:
  - "${input:tool} best practices"
  - "${input:tool} production guide"
  - "${input:tool} security recommendations"
- **検証**: 少なくとも3つ以上の外部ソースと比較し、不足している重要なポイントを特定すること

### 6. ベストプラクティスドキュメントの更新
- **タスク**: レビューに基づき `docs/best_practice/${input:tool}/best_practice.md` を更新する
- **更新内容**:
  - 不足していたベストプラクティスの追加
  - 情報の正確性と最新性の確認
  - 構造とフォーマットの改善
- **検証**: 更新後のドキュメントが包括的で、実務に即していること

### 7. instructionsファイルの作成
- **タスク**: #tool:agent/runSubagent を使用してサブエージェントに委譲し、instructionsファイルを作成
- **詳細**: `.github/instructions/instructions.instructions.md を参考に ${input:tool} のベストプラクティス規則 .github/instructions/<kebab-case(${input:tool})>-best-practice.instructions.md を日本語で作成する。applyTo フィールドには、このツールが主に適用されるファイルパターンを指定すること`
- **検証**: instructionsファイルが以下を満たすこと:
  - 正しい frontmatter (description, applyTo)
  - 明確で実行可能なガイドライン
  - 具体的な例とアンチパターン
- **既存ファイル**: `.github/instructions/<kebab-case(${input:tool})>-best-practice.instructions.md` が存在する場合は、内容を更新して反映する

## 出力期待値

### 成果物
1. `docs/best_practice/${input:tool}/url.md`: 参考URLの一覧
2. `docs/best_practice/${input:tool}/best_practice.md`: 包括的なベストプラクティスドキュメント (日本語)
3. `.github/instructions/<kebab-case(${input:tool})>-best-practice.instructions.md`: Copilot用instructions規則ファイル (日本語)

### フォーマット要件
- すべてのMarkdownファイルは `.github/instructions/markdown.instructions.md` の標準に準拠
- instructionsファイルは `.github/instructions/instructions.instructions.md` のガイドラインに準拠
- コードサンプルは適切なシンタックスハイライトを使用

### 完了条件
- 3つの成果物がすべて作成されている
- 各ファイルが品質基準を満たしている
- エラーや警告が残っていない

## 品質保証チェックリスト

実行後、以下を確認してください:

- [ ] `docs/best_practice/${input:tool}/url.md` に10件以上の関連URLがリストされている
- [ ] `docs/best_practice/${input:tool}/best_practice.md` が包括的で実践的な内容である
- [ ] ベストプラクティスドキュメントに公式ドキュメントとウェブ検索の両方の知見が反映されている
- [ ] instructionsファイルに適切な frontmatter (description, applyTo) が含まれている
- [ ] instructionsファイルのガイドラインが具体的で実行可能である
- [ ] すべてのマークダウンファイルが正しくフォーマットされている
- [ ] 日本語の文章が自然で読みやすい

## 失敗時の対応

- 公式ドキュメントへのアクセスが失敗した場合: ユーザーにURLの確認を求め、代替URLを試行
- ウェブ検索で十分な情報が得られない場合: 検索キーワードを調整し再試行
- サブエージェントのタスクが不完全な場合: 不足部分を特定し、追加の指示で補完

## 追加リソース

- [Prompt Files Documentation](https://code.visualstudio.com/docs/copilot/customization/prompt-files)
- [GitHub Copilot Custom Instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- リポジトリ内の既存instructions: `.github/instructions/` ディレクトリを参照
