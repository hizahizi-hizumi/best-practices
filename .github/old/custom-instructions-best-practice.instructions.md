---
description: 'GitHub Copilot カスタム instructions ファイル作成のベストプラクティスとガイドライン'
applyTo: '**/*.instructions.md, .github/copilot-instructions.md'
---

# GitHub Copilot Custom Instructions ベストプラクティス

このファイルは、効果的で保守性の高い custom instructions を作成するためのガイドラインを定義します。

## 基本原則

### 1. 簡潔性と明確性

- 各指示は単一のシンプルな文で表現する
- 複数の概念は複数の指示に分割する
- 具体的で実行可能な指示を記述する
- 抽象的な表現を避け、明確なアクションを指定する

**推奨例**:
```markdown
- すべての関数に JSDoc コメントを付ける
- 型アノテーションを常に使用する
- エラーハンドリングには try-catch を使用する
```

**非推奨例**:
```markdown
- すべての関数には JSDoc コメントを付け、型アノテーションも必ず使用し、
  さらにエラーハンドリングを適切に行うこと
```

### 2. 自己完結性

- 各 instructions ファイルは独立して理解可能にする
- 外部の詳細情報はファイル参照や URL リンクで提供する
- 共通ルールは1箇所に定義し、他のファイルから参照する

### 3. スコープの適切な管理

- `applyTo` パターンで適用範囲を明確に定義する
- プロジェクト全体のルールとファイル固有のルールを分離する
- ワークスペーススコープとユーザースコープを使い分ける

## YAML Frontmatter の規約

### 必須フィールド

```yaml
---
description: 'instructions ファイルの目的と範囲を明確に記述（1-500文字）'
applyTo: 'glob パターンで対象ファイルを指定'
---
```

### フィールドガイドライン

#### description
- シングルクォートで囲む
- 1-500文字の範囲
- instructions の目的を明確に記述
- 読み手が内容を即座に理解できる説明

#### applyTo
- glob パターンで対象ファイルを指定
- ワークスペースルートからの相対パス
- 複数パターンはカンマ区切りで指定可能
- 指定しない場合は手動追加のみ

**パターン例**:
```yaml
# 単一の拡張子
applyTo: '**/*.py'

# 複数の拡張子
applyTo: '**/*.{jsx,tsx}'

# 特定ディレクトリ
applyTo: 'src/api/**/*'

# テストファイル
applyTo: '**/*.test.{ts,js}'

# すべてのファイル（慎重に使用）
applyTo: '**'
```

### Prompt Files の追加フィールド

```yaml
---
name: 'prompt-identifier'
description: 'プロンプトの目的'
argument-hint: 'ユーザーへの入力ヒント'
agent: 'agent'
model: 'gpt-4'
tools:
  - 'read_file'
  - 'write_file'
---
```

## ファイル構造のベストプラクティス

### 推奨セクション構成

1. **目的とスコープ** - instructions の適用範囲と目的を説明
2. **プロジェクト概要** - 技術スタック、対象ユーザー、主要機能
3. **コーディング規約** - 命名規則、スタイル、フォーマット
4. **アーキテクチャガイドライン** - プロジェクト構造、設計パターン
5. **ベストプラクティス** - 推奨パターンと避けるべきパターン
6. **セキュリティ考慮事項** - セキュリティ要件とガイドライン（該当する場合）
7. **パフォーマンス最適化** - パフォーマンスに関する推奨事項（該当する場合）
8. **テスト要件** - テストの書き方と要件（該当する場合）

### ファイル配置の推奨構造

```
.github/
├── copilot-instructions.md           # プロジェクト全体の基本ルール
├── instructions/
│   ├── python-standards.instructions.md
│   ├── react-standards.instructions.md
│   ├── api-standards.instructions.md
│   └── security-guidelines.instructions.md
├── prompts/
│   ├── create-component.prompt.md
│   ├── generate-tests.prompt.md
│   └── code-review.prompt.md
└── skills/
    └── webapp-testing/
        └── SKILL.md
```

## コンテンツ作成のガイドライン

### 具体的な指示の記述

**推奨**: 明確で測定可能な指示
```markdown
- 変数名には camelCase を使用する
- クラス名には PascalCase を使用する
- 定数には UPPER_SNAKE_CASE を使用する
- 行の長さは最大100文字に制限する
- インデントには2スペースを使用する
```

**非推奨**: 曖昧で抽象的な指示
```markdown
- 適切な命名規則を使用する
- コードを読みやすくする
- 良い習慣に従う
```

### 例とコード片の提供

コード例を含めることで、AI の理解を大幅に向上できます。

#### 良い例と悪い例の対比

```markdown
### 推奨: TypeScript インターフェースの使用

\`\`\`typescript
interface User {
  id: string;
  name: string;
  email: string;
}

function getUser(id: string): User {
  // 実装
}
\`\`\`

### 非推奨: any 型の使用

\`\`\`typescript
function getUser(id: any): any {
  // 型安全性が失われる
}
\`\`\`
```

#### テーブルによる構造化情報

```markdown
| 問題               | 解決策                | 例                           |
| ------------------ | --------------------- | ---------------------------- |
| マジックナンバー   | 名前付き定数を使用    | `const MAX_RETRIES = 3`      |
| 深いネスト         | 関数抽出              | ネストした if 文をリファクタ |
| ハードコード値     | 設定ファイルを使用    | API URL を config に保存     |
```

### ファイルと URL の参照

#### プロジェクト内ファイルへの参照

```markdown
# TypeScript コーディング標準

プロジェクトの型定義については [types.ts](../types/common.ts) を参照してください。

詳細なスタイルガイドは [スタイルガイド](../../docs/style-guide.md) を確認してください。
```

#### 外部リソースへの参照

```markdown
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html) に従う
- [React 公式ドキュメント](https://react.dev/learn) のベストプラクティスを参考にする
```

### ツール参照の活用

Agent tools を参照する場合は `#tool:<tool-name>` 構文を使用します。

```markdown
GitHub リポジトリ情報を取得する場合は #tool:githubRepo を使用してください。

ファイルシステム操作には以下のツールが利用可能です:
- #tool:read_file - ファイルの読み込み
- #tool:write_file - ファイルの書き込み
- #tool:list_directory - ディレクトリの一覧
```

## スコープ管理とファイルタイプ

### ワークスペーススコープ vs ユーザースコープ

#### ワークスペーススコープ（`.github/instructions/`）

**用途**:
- プロジェクト固有のルールと規約
- チームメンバー間で共有する設定
- バージョン管理に含めるべき instructions

**メリット**:
- チーム全体で一貫性を保てる
- プロジェクトのドキュメントとして機能
- 新しいメンバーのオンボーディングが容易

#### ユーザースコープ（プロファイルフォルダ）

**用途**:
- 個人的なコーディングスタイル
- 複数プロジェクトで使用する汎用的なルール
- 個人の生産性向上のための設定

**メリット**:
- すべてのワークスペースで利用可能
- Settings Sync で同期可能
- プロジェクトに依存しない

### ファイルタイプの選択

#### `.github/copilot-instructions.md`
- プロジェクト全体に適用される基本的なコーディング標準
- チーム全体で共有する一般的なガイドライン
- GitHub.com や Visual Studio でも認識される

#### `.instructions.md` ファイル
- 言語固有のコーディング規約
- フレームワーク固有のパターン
- プロジェクト内の特定ディレクトリに適用するルール
- 複数プロジェクトで再利用するユーザーレベルの設定

#### `AGENTS.md` ファイル
- 複数の AI エージェントに共通の指示
- エージェント間で共有するコンテキスト情報
- プロジェクト構造に関する全体的なガイダンス

## セキュリティとプライバシー

### 機密情報の除外

**避けるべき内容**:
```markdown
# 非推奨 - 機密情報を含めない
- API キー: sk-1234567890abcdef
- データベース接続文字列: mongodb://user:pass@host:27017/db
- 個人情報やプライベートなビジネスロジック
- 認証トークンやパスワード
```

**推奨アプローチ**:
```markdown
# 推奨 - 環境変数や安全な方法を指示
- API キーは環境変数 `API_KEY` から取得する
- データベース接続情報は `.env` ファイルで管理する
- 機密情報はコードにハードコードしない
- secrets は `.gitignore` に追加し、バージョン管理から除外する
```

### 信頼境界の理解

- **Workspace Trust**: 信頼されていないワークスペースでは特定の機能が無効化される
- **Extension Publisher Trust**: 拡張機能のインストールには Publisher の信頼が必要
- **MCP Server Trust**: MCP サーバーは起動前に信頼の確認が必要

### ツール自動承認のリスク管理

```json
{
  // デフォルト: 承認を要求（推奨）
  "github.copilot.chat.tool.approval": "user",

  // セッション限定で一時的に承認を許可
  "github.copilot.chat.terminal.autoApprove": {
    "commands": ["git status", "npm test", "ls", "find"]
  }
}
```

## パフォーマンス最適化

### コンテキストサイズの最適化

#### ベストプラクティス

1. **簡潔に保つ**: 各 instructions ファイルは1つのトピックに焦点を当てる
2. **条件付き適用**: `applyTo` パターンで必要な場合のみ適用

#### 避けるべきパターン

```yaml
# 非推奨: すべてのファイルに適用
---
applyTo: '**'
---
# 大量の instructions...
```

```yaml
# 推奨: 特定のファイルのみに適用
---
applyTo: 'src/**/*.ts'
---
# 焦点を絞った instructions
```

## 共通パターンとアンチパターン

### 推奨パターン

#### パターン1: 階層的な構造

```markdown
# .github/copilot-instructions.md
- DRY 原則に従う
- コードの可読性を優先する

# .github/instructions/python-standards.instructions.md
---
applyTo: '**/*.py'
---
[プロジェクト全体のルール](../copilot-instructions.md) に加えて以下を守る:
- PEP 8 に準拠
- 型ヒントを使用
```

#### パターン2: 条件分岐による指示

```markdown
## フレームワーク選択

- **小規模プロジェクト**: Minimal API アプローチを使用
- **大規模プロジェクト**: コントローラーベースのアーキテクチャを使用
- **マイクロサービス**: ドメイン駆動設計パターンを検討
```

#### パターン3: コードファイルを Instructions として使用

```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "database/schema.sql" },
    { "file": "types/api-contracts.ts" }
  ]
}
```

### アンチパターン

#### アンチパターン1: 過度に冗長な説明

```markdown
# 非推奨
- すべての関数には適切なドキュメントを記述し、引数の説明と戻り値の説明を含め、
  さらに使用例も提供し、エラーケースについても記述する必要があります。
  また、型アノテーションも忘れずに追加してください。
```

#### アンチパターン2: 曖昧な指示

```markdown
# 非推奨
- 良いコードを書く
- バグを避ける
- ベストプラクティスに従う
```

#### アンチパターン3: 矛盾する指示

```markdown
# ファイル1
- 常に詳細なコメントを記述する

# ファイル2（同じスコープ）
- コメントは最小限にし、コードで意図を表現する
```

## チーム開発での運用

### バージョン管理のベストプラクティス

#### `.gitignore` 設定

```gitignore
# ユーザー固有の設定は除外
.vscode/settings.json

# チーム共有の instructions は含める
!.github/copilot-instructions.md
!.github/instructions/
!.github/prompts/
!.github/skills/
```

### オンボーディングドキュメント

新しいチームメンバー向けに、instructions の使い方をドキュメント化します:

- Instructions ファイルの場所と目的
- 有効化に必要な設定
- 利用可能な prompts とその使い方
- ベストプラクティスと注意事項

### レビュープロセス

#### Pull Request テンプレートに含める項目

```markdown
## Copilot 使用チェックリスト

- [ ] コードは `.github/copilot-instructions.md` のルールに準拠している
- [ ] Copilot が生成したコードをレビューした
- [ ] 自動生成されたテストを確認した
- [ ] セキュリティガイドラインに従っている
- [ ] すべての提案を確認してから適用した
```

### 定期的なメンテナンス

3〜6ヶ月ごとに instructions の内容をレビュー:

- 不要になったルールの削除
- 新しいベストプラクティスの追加
- チームのフィードバックの反映
- プロジェクトの変化に応じた更新

## トラブルシューティング

### Instructions が適用されない場合

#### チェック項目1: 設定の確認

```json
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,
  "chat.useAgentsMdFile": true
}
```

#### チェック項目2: ファイル配置の確認

- `.github/copilot-instructions.md` はワークスペースルートに配置されているか
- `.instructions.md` ファイルは適切なフォルダにあるか
- カスタムフォルダを使用している場合、設定で指定されているか

#### チェック項目3: applyTo パターンの確認

```yaml
# 正しい例
applyTo: '**/*.py'              # すべての Python ファイル
applyTo: 'src/components/**/*'  # components 配下のすべて

# 間違いやすい例
applyTo: '*.py'                 # ルート直下のみ（意図しない制限）
applyTo: '/src/components/**/*' # 先頭の / は不要
```

### 生成されたコードが期待と異なる場合

#### 改善アプローチ

1. **Instructions の明確性**: 指示が具体的で明確か確認
2. **例の提供**: 期待する出力の例を含める
3. **コンテキストの追加**: 関連ファイルを参照する
4. **モデルの選択**: タスクに適したモデルを使用する

## クイックリファレンス

### ユースケース別推奨ファイルタイプ

| ユースケース | ファイルタイプ | スコープ | applyTo 例 |
|------------|---------------|---------|-----------|
| プロジェクト全体のルール | `copilot-instructions.md` | ワークスペース | 自動適用 |
| 言語固有のルール | `.instructions.md` | ワークスペース | `**/*.py` |
| 個人的な設定 | `.instructions.md` | ユーザー | 任意 |
| 再利用可能なタスク | `.prompt.md` | ワークスペース/ユーザー | 任意 |
| 専門化された能力 | `SKILL.md` | ワークスペース | N/A |

### 品質チェックリスト

- [ ] YAML frontmatter が正しく記述されている
- [ ] `description` が明確で簡潔（1-500文字）
- [ ] `applyTo` パターンが適切に設定されている
- [ ] Instructions は簡潔で自己完結的
- [ ] 具体的で実行可能な指示を含む
- [ ] 機密情報を含まない
- [ ] 良い例と悪い例を含む
- [ ] 他のファイルへの参照を適切に使用
- [ ] チーム開発を考慮した構造
- [ ] 定期的なレビューとメンテナンスの計画がある

## 参考リソース

- [VS Code 公式ドキュメント - Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [VS Code 公式ドキュメント - Security](https://code.visualstudio.com/docs/copilot/security)
