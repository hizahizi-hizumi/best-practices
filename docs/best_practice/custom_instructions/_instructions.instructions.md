---
description: 'GitHub Copilot Custom Instructions ファイル作成のベストプラクティス（要点版）'
applyTo: '**/*.instructions.md, .github/copilot-instructions.md'
---

# GitHub Copilot Custom Instructions ベストプラクティス

VS Code の Custom Instructions を効果的に活用するための要点をまとめたガイドラインです。

## コア原則

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

#### 2. 命令形と具体性（Imperative & Specific）

コミュニティ実例から学ぶ重要原則:

- **命令形で書く**: "Use", "Avoid", "Validate" などの動詞で開始
- **曖昧語を排除**: 「適切に」「よしなに」「できれば」を使わない
- **出力形式を明示**: 箇条書き、表、JSON、チェックリストなど

**推奨**:
```markdown
- 変数名には camelCase を使用する
- クラス名には PascalCase を使用する
- 定数には UPPER_SNAKE_CASE を使用する
- 行の長さは最大100文字に制限する
```

**非推奨**:
```markdown
- 適切な命名規則を使用する
- コードを読みやすくする
- 良い習慣に従う
```

#### 3. 自己完結性と参照の活用

- 各 instructions ファイルは独立して理解可能にする
- 外部の詳細情報はファイル参照や URL リンクで提供する
- 共通ルールは1箇所に定義し、他のファイルから参照する

#### 4. 変更の最小化（Surgical Changes）

コミュニティで強調される原則:

- **依頼されていないリファクタ・整形・命名変更をしない**
- **置換ではなく「既存に統合」を優先する**
- **最小差分で目的を達成する**

狙い: 不要な差分増大、レビュー負担増、予期せぬ副作用を抑える

#### 5. 優先順位の明示

複数の instructions が合成される環境では、衝突時の判断基準を明記:

```markdown
## 指示の優先順位

1. ユーザーの明示的な指示が最優先
2. セキュリティ要件は常に優先
3. プロジェクト固有のルール
4. 一般的なベストプラクティス
5. 不確かな外部情報はツールで検証
```

## ファイルタイプの選択

VS Code は3種類の Markdown ベースの instructions ファイルをサポートしています。

### 1. `.github/copilot-instructions.md`

#### 特徴
- ワークスペースのルートに単一ファイルとして配置
- ワークスペース内のすべてのチャットリクエストに自動適用
- GitHub.com や Visual Studio でも認識される

#### 推奨用途
- プロジェクト全体に適用される基本的なコーディング標準
- チーム全体で共有する一般的なガイドライン
- プロジェクトの基本方針や技術スタック情報

#### 設定方法
```json
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true
}
```

#### ファイル構造例
```markdown
# プロジェクトコーディング標準

## 一般原則
- コードの可読性と明確性を優先する
- 各関数に明確で簡潔なコメントを記述する
- DRY（Don't Repeat Yourself）原則に従う

## TypeScript 規約
- 常に型アノテーションを使用する
- `any` 型の使用を避ける
- 関数には明確な戻り値の型を指定する
```

### 2. `.instructions.md` ファイル

#### 特徴
- 複数ファイルの作成が可能
- glob パターンによる条件付き適用
- ワークスペースまたはユーザープロファイルに保存可能

#### 推奨用途
- 言語固有のコーディング規約（例: Python、TypeScript、Rust）
- フレームワーク固有のパターン（例: React、Vue、Next.js）
- プロジェクト内の特定ディレクトリに適用するルール
- 複数プロジェクトで再利用するユーザーレベルの設定

#### スコープの使い分け

**ワークスペーススコープ（`.github/instructions/`）**
- プロジェクト固有のルールと規約
- チームメンバー間で共有する設定
- バージョン管理に含めるべき instructions

**ユーザースコープ（プロファイルフォルダ）**
- 個人的なコーディングスタイル
- 複数プロジェクトで使用する汎用的なルール
- 個人の生産性向上のための設定

#### ファイル配置の推奨構造

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

### 3. `AGENTS.md` ファイル

#### 特徴
- ワークスペースのルートに配置
- 複数の AI エージェントを使用する環境向け
- 実験的にサブフォルダでの配置もサポート

#### 推奨用途
- 複数のエージェントに共通の指示
- エージェント間で共有するコンテキスト情報
- プロジェクト構造に関する全体的なガイダンス

#### 設定方法
```json
{
  "chat.useAgentsMdFile": true,
  "chat.useNestedAgentsMdFiles": true
}
```

### ファイルタイプ選択のガイドライン

| ユースケース | ファイルタイプ | スコープ | applyTo 例 |
|------------|---------------|---------|-----------|
| プロジェクト全体のルール | `copilot-instructions.md` | ワークスペース | 自動適用 |
| 言語固有のルール | `.instructions.md` | ワークスペース | `**/*.py` |
| 個人的な設定 | `.instructions.md` | ユーザー | 任意 |
| 再利用可能なタスク | `.prompt.md` | ワークスペース/ユーザー | 任意 |
| 専門化された能力 | `SKILL.md` | ワークスペース | N/A |

## YAML Frontmatter の規約

### 必須フィールド

```yaml
---
description: 'instructions ファイルの目的と範囲を明確に記述（1-500文字）'
applyTo: 'glob パターンで対象ファイルを指定'
---
```

### `.instructions.md` の Frontmatter

```yaml
---
name: "React コンポーネント標準"
description: "React コンポーネント開発のベストプラクティス"
applyTo: "**/*.{jsx,tsx}"
---
```

#### フィールドガイドライン

- **`name`** (オプション): UI に表示されるファイル名
- **`description`** (必須): シングルクォートで囲む、1-500文字、目的を明確に記述
- **`applyTo`** (オプション): glob パターンで対象ファイルを指定
  - 指定しない場合は手動追加のみ
  - ワークスペースルートからの相対パス

#### applyTo パターンの例

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
name: 'react-form-generator'
description: 'React フォームコンポーネントの生成'
argument-hint: 'フォーム名と必要なフィールドを指定'
agent: 'agent'
model: 'gpt-4'
tools:
  - 'read_file'
  - 'write_file'
  - 'githubRepo/*'
---
```

#### フィールド説明

- **`agent`**: 使用するエージェント（`ask`, `edit`, `agent`, カスタム名）
- **`model`**: 使用する言語モデル（未指定時は現在選択中を使用）
- **`tools`**: プロンプトで使用可能なツールのリスト（最小権限の原則に従う）
- **`argument-hint`**: チャット入力フィールドに表示されるヒント

## Instructions の書き方

### 推奨セクション構成

効果的な instructions ファイルには以下の要素を含めることを推奨します:

#### 1. 目的とスコープ (Purpose & Scope)

```markdown
# 目的
このファイルは、React コンポーネント開発におけるコーディング規約と
安全なデータ処理をガイドします。
```

#### 2. 適用範囲の明示 (Location & Scope Rules)

```yaml
---
applyTo: "src/components/**/*.{jsx,tsx}"
---

# 適用範囲
- React コンポーネント (.jsx, .tsx ファイル)
- src/components/ ディレクトリ配下のみ

# 適用外
- テストファイル
- Storybook ファイル
```

#### 3. プロジェクト概要 (Project Overview)

```markdown
## プロジェクト概要
- **目的**: エンタープライズ向け在庫管理システム
- **対象ユーザー**: 倉庫管理者、在庫担当者
- **主要機能**: リアルタイム在庫追跡、自動発注
- **技術スタック**: React 18, TypeScript 5, Vite
```

#### 4. ツールとバージョン (Tooling & Versions)

```markdown
## 開発環境
- Node.js: 20.x LTS
- npm: 10.x
- TypeScript: 5.x

## コマンド
\`\`\`bash
npm run dev      # 開発サーバー起動
npm run build    # プロダクションビルド
npm test         # テスト実行
\`\`\`
```

#### 5. コーディング規約

```markdown
## 命名規則
- 変数名: camelCase
- クラス名: PascalCase
- 定数: UPPER_SNAKE_CASE
- プライベートフィールド: _prefix

## スタイル
- インデント: 2スペース
- 最大行長: 100文字
- セミコロン: 必須
```

#### 6. API 契約とインターフェース

プロジェクト内の型定義や API 仕様を参照:

```markdown
## 型定義
<!-- プロジェクトの型定義ファイルへのパスを記載 -->
詳細は types/api.ts を参照してください。

## API エンドポイント
- `GET /api/users` - ユーザー一覧取得
- `POST /api/users` - ユーザー作成
```

### 具体例の提供方法

#### Good / Bad の対比（コミュニティ推奨パターン）

良い例と悪い例を必ず対で提示します:

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

#### テーブルによる構造化

```markdown
| 問題               | 解決策                | 例                           |
| ------------------ | --------------------- | ---------------------------- |
| マジックナンバー   | 名前付き定数を使用    | `const MAX_RETRIES = 3`      |
| 深いネスト         | 関数抽出              | ネストした if 文をリファクタ |
| ハードコード値     | 設定ファイルを使用    | API URL を config に保存     |
```

#### WHY（理由）の簡潔な添付

重要なルールには「なぜ」を1〜2文で補足:

```markdown
### セキュリティルール

- SQL クエリにはパラメタライズドクエリを使用する
  **理由**: 文字列連結は SQL インジェクション攻撃の原因となる

- パスワードは必ずハッシュ化して保存する
  **理由**: 平文保存はデータ漏洩時に致命的なリスクとなる
```

### 参照とリンク

#### プロジェクト内ファイルへの参照

```markdown
# TypeScript コーディング標準

<!-- プロジェクトのファイルパスに合わせて調整してください -->
プロジェクトの型定義については types/common.ts を参照してください。

詳細なスタイルガイドは docs/style-guide.md を確認してください。
```

#### 外部リソースへの参照

```markdown
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html) に従う
- [React 公式ドキュメント](https://react.dev/learn) のベストプラクティスを参考にする
```

#### ツール参照

GitHub リポジトリ情報を取得する場合は `#tool:githubRepo` を使用してください。

その他の利用可能なツールについては、プロジェクトで定義されているツール一覧を参照してください。

#### コードファイルを Instructions として使用

設定で既存のコードファイルを参照することも可能:

```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "database/schema.sql" },
    { "file": "types/api-contracts.ts" }
  ]
}
```

### チェックリストの活用（コミュニティパターン）

コードレビュー、性能、セキュリティなど、実行可能な受け入れ基準としてチェックリストを最後に配置:

```markdown
## コードレビューチェックリスト

### CRITICAL（マージブロック）
- [ ] セキュリティ脆弱性がない
- [ ] API 契約に準拠している
- [ ] すべてのテストがパスしている

### IMPORTANT（議論必須）
- [ ] エラーハンドリングが適切
- [ ] パフォーマンスへの影響を検証済み
- [ ] ドキュメントが更新されている

### SUGGESTION（改善提案）
- [ ] 変数名が明確
- [ ] コメントが適切
- [ ] 重複コードがない
```

## スコープ管理

### applyTo での条件付き適用（コミュニティ推奨）

技術領域ごとにファイルを分け、`applyTo` で必要な時だけ効かせる:

```yaml
# 広く使う共通ガイド
applyTo: '**'

# 特定の拡張子に狙い撃ち
applyTo: '**/*.prompt.md'

# テストファイルのみ
applyTo: '**/*.{test,spec}.{ts,js}'
```

**狙い**: 無関係な指示の混入を減らし、コンテキスト消費も抑える

### ワークスペーススコープ vs ユーザースコープ

#### ワークスペーススコープ（`.github/instructions/`）

**メリット**:
- チーム全体で一貫性を保てる
- プロジェクトのドキュメントとして機能
- 新しいメンバーのオンボーディングが容易

#### ユーザースコープ（プロファイルフォルダ）

**メリット**:
- すべてのワークスペースで利用可能
- Settings Sync で同期可能
- プロジェクトに依存しない

### 階層的な構造（推奨パターン）

```markdown
# .github/copilot-instructions.md
- DRY 原則に従う
- コードの可読性を優先する

# .github/instructions/python-standards.instructions.md
---
applyTo: '**/*.py'
---
<!-- プロジェクト全体のルール（上位ファイル）を参照 -->
プロジェクト全体のルールに加えて以下を守る:
- PEP 8 に準拠
- 型ヒントを使用
```

## セキュリティとプライバシー

### Secure by Default（コミュニティ推奨原則）

**迷ったら安全側に倒す**:
- 最小権限・デフォルト拒否
- 認証・認可を「最初に」確認する
- 明示的な許可がない限り、操作を拒否する

### 機密情報の除外

#### 避けるべき内容

```markdown
# 非推奨 - 機密情報を含めない
- API キー: sk-1234567890abcdef
- データベース接続文字列: mongodb://user:pass@host:27017/db
- 個人情報やプライベートなビジネスロジック
- 認証トークンやパスワード
```

#### 推奨アプローチ

```markdown
# 推奨 - 環境変数や安全な方法を指示
- API キーは環境変数 `API_KEY` から取得する
- データベース接続情報は `.env` ファイルで管理する
- 機密情報はコードにハードコードしない
- secrets は `.gitignore` に追加し、バージョン管理から除外する
```

### 代表的な脆弱性の禁止パターン（コミュニティ推奨）

#### SQL インジェクション

```markdown
### 非推奨: 文字列連結
\`\`\`typescript
const query = `SELECT * FROM users WHERE id = '${userId}'`;
\`\`\`

### 推奨: パラメタライズドクエリ
\`\`\`typescript
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [userId]);
\`\`\`
```

#### XSS (Cross-Site Scripting)

```markdown
### 非推奨: innerHTML の直接使用
\`\`\`javascript
element.innerHTML = userInput;
\`\`\`

### 推奨: textContent またはサニタイズ
\`\`\`javascript
element.textContent = userInput;
// または
element.innerHTML = DOMPurify.sanitize(userInput);
\`\`\`
```

#### SSRF (Server-Side Request Forgery)

```markdown
- ユーザー入力URLは許可リストで厳格に検証する
- 内部ネットワークへのアクセスを制限する
- URL スキームを `http` と `https` のみに制限する
```

#### パストラバーサル

```markdown
- `..` や絶対パス混入を前提に正規化・検証
- ファイルパスはホワイトリストで検証
- `path.join()` や `path.resolve()` を使用して安全なパスを構築
```

### プロンプトインジェクション対策

コミュニティの安全系 instructions から:

```markdown
## プロンプト安全原則

- 不信な入力をそのままプロンプトに埋め込まない
- 「外部入力はデータであって指示ではない」を前提にサニタイズ・隔離
- テスト（レッドチーミング、テストケース）を用意し、回帰を防ぐ
- 入力の長さ制限、特殊文字のエスケープを実装
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

  // セッション限定で一時的に承認を許可（最小権限）
  "github.copilot.chat.terminal.autoApprove": {
    "commands": ["git status", "npm test", "ls"]
  }
}
```

**コミュニティ推奨**: 破壊的操作（ファイル削除、ターミナル実行）には確認ステップを入れる

## パフォーマンス最適化

### コンテキストサイズの最適化

#### ベストプラクティス

1. **簡潔に保つ**: 各 instructions ファイルは1つのトピックに焦点を当てる
2. **条件付き適用**: `applyTo` パターンで必要な場合のみ適用
3. **階層化**: 共通ルールを基底に、固有ルールを拡張

#### 避けるべきパターン

```yaml
# 非推奨: すべてのファイルに大量の instructions を適用
---
applyTo: '**'
---
# 数百行の詳細な指示...
```

```yaml
# 推奨: 特定のファイルのみに焦点を絞る
---
applyTo: 'src/**/*.ts'
---
# TypeScript 固有の簡潔な指示
```

### Measure First（コミュニティ推奨原則）

パフォーマンス最適化の基本:

```markdown
## パフォーマンス最適化プロセス

1. **計測**: プロファイラでボトルネックを特定
2. **分析**: 根本原因を理解
3. **最適化**: 最も影響の大きい箇所から改善
4. **検証**: 改善効果を計測で確認

**推測で最適化しない**
```

### レイヤ別のチェック観点（コミュニティパターン）

```markdown
## フロントエンド
- [ ] 不要な DOM 更新を避ける
- [ ] バンドルサイズを最適化
- [ ] 画像/フォントの遅延読み込み
- [ ] ネットワークリクエストを最小化

## バックエンド
- [ ] 非同期I/Oを活用
- [ ] キャッシュ戦略を実装
- [ ] 接続プールを適切に設定
- [ ] N+1 クエリを回避

## データベース
- [ ] 適切なインデックスを作成
- [ ] `SELECT *` を避ける
- [ ] クエリ実行計画を確認
- [ ] 接続数を監視
```

### 典型的アンチパターン

```markdown
## 避けるべきパターン

### フロントエンド
- ループ内での DOM 更新
- 過度な再レンダリング
- 同期的な重い処理

### バックエンド
- 同期I/O（特にNodeのホットパス）
- 無限に増える配列/ログ
- グローバル状態の乱用

### データベース
- N+1 クエリ
- インデックスなしの検索
- 過度に複雑な JOIN
```

## コミュニティから学ぶパターン

このセクションでは、awesome-copilot の実例から抽出した共通パターンをまとめます。

### テンプレートアプローチ

汎用テンプレートを提供し、プロジェクト固有に拡張させる:

```markdown
# コードレビュー標準（テンプレート）

以下は汎用的なチェック項目です。
**プロジェクト固有の項目をここに追加してください。**

## 共通チェック項目
- [ ] コードが動作する
- [ ] テストがパスする
- [ ] ドキュメントが更新されている

## プロジェクト固有（ここに追加）
- [ ] [your-specific-check]
```

**狙い**: 最初から完璧を目指さず、運用で育てられる

### コードレビューの型化

優先度と形式を統一:

```markdown
## レビューコメント形式

### 優先度
- **CRITICAL**: マージブロック。セキュリティ、バグ、破壊的変更
- **IMPORTANT**: 議論必須。設計、パフォーマンス、保守性
- **SUGGESTION**: 改善提案。可読性、スタイル、最適化

### コメント構造
1. 何が問題か（What）
2. なぜ問題か（Why）
3. どう直すか（How）
4. 参照（標準、ドキュメント）

### 例
**CRITICAL**: SQL インジェクション脆弱性
- **問題**: ユーザー入力を直接SQLに連結している
- **リスク**: 攻撃者がデータベースを操作可能
- **修正**: パラメタライズドクエリを使用
- **参照**: OWASP SQL Injection ガイド
```

### 条件分岐による指示

プロジェクトサイズや状況に応じた適応:

```markdown
## アーキテクチャ選択

**小規模プロジェクト**:
- Minimal API アプローチを使用
- シンプルなフォルダ構造
- モノリシック構成

**大規模プロジェクト**:
- コントローラーベースのアーキテクチャを使用
- レイヤード構造（プレゼンテーション、ビジネス、データ）
- モジュール分割

**マイクロサービス**:
- ドメイン駆動設計パターンを検討
- イベント駆動アーキテクチャ
- API Gateway パターン
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

### 定期的なメンテナンス（コミュニティ推奨）

Instructions はコードと同じく変更管理:

```markdown
## メンテナンススケジュール

### 3〜6ヶ月ごと
- [ ] 不要になったルールの削除
- [ ] 新しいベストプラクティスの追加
- [ ] チームのフィードバックの反映
- [ ] プロジェクトの変化に応じた更新
- [ ] 依存関係更新や設計変更との整合性確認
```

### オンボーディングドキュメント

新しいチームメンバー向けガイド:

```markdown
# Copilot Instructions ガイド

## 概要
このプロジェクトは Custom Instructions を使用して、
一貫したコード生成とレビューをサポートしています。

## 有効化方法
1. VS Code で settings.json を開く
2. 以下を追加:
   \`\`\`json
   {
     "github.copilot.chat.codeGeneration.useInstructionFiles": true
   }
   \`\`\`

## ファイル構成
- `.github/copilot-instructions.md`: プロジェクト全体のルール
- `.github/instructions/`: 言語・フレームワーク固有のルール
- `.github/prompts/`: 再利用可能なタスクテンプレート

## 利用可能な Prompts
- `/create-component`: React コンポーネント生成
- `/generate-tests`: テストコード生成
- `/code-review`: コードレビュー実行
```

### Pull Request テンプレートへの統合

```markdown
## Copilot 使用チェックリスト

- [ ] コードは `.github/copilot-instructions.md` のルールに準拠している
- [ ] Copilot が生成したコードをレビューした
- [ ] 自動生成されたテストを確認した
- [ ] セキュリティガイドラインに従っている
- [ ] すべての提案を確認してから適用した
- [ ] 意図しない変更が含まれていない（最小変更原則）
```

### テストと検証を指示に組み込む（コミュニティ推奨）

```markdown
## 検証要件

- 例は実行可能であること（サンプルコードは動作確認済み）
- glob パターンが意図した対象に当たることを確認
- 変更後にテスト/リンタを必ず実行
- CI パイプラインがパスすることを確認
```

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

1. **Instructions の明確性**: 指示が具体的で明確か確認（曖昧語を排除）
2. **例の提供**: Good/Bad 例を含める
3. **コンテキストの追加**: 関連ファイルを参照する
4. **モデルの選択**: タスクに適したモデルを使用する
5. **優先順位の明示**: 衝突時の判断基準を追加

### 共通アンチパターン

#### アンチパターン1: 過度に冗長な説明

```markdown
# 非推奨
- すべての関数には適切なドキュメントを記述し、引数の説明と戻り値の説明を含め、
  さらに使用例も提供し、エラーケースについても記述する必要があります。
  また、型アノテーションも忘れずに追加してください。

# 推奨
- すべての関数に JSDoc を記述する
- 引数と戻り値の型を明示する
- エラーケースを documented exceptions として記載する
```

#### アンチパターン2: 曖昧な指示

```markdown
# 非推奨
- 良いコードを書く
- バグを避ける
- ベストプラクティスに従う

# 推奨
- 関数は単一責任原則に従う
- 入力検証をすべての公開APIで実施する
- DRY 原則: 同じロジックを3回以上繰り返さない
```

#### アンチパターン3: 矛盾する指示

```markdown
# 問題例（同じスコープ内）

# ファイル1
- 常に詳細なコメントを記述する

# ファイル2
- コメントは最小限にし、コードで意図を表現する

# 解決: 優先順位を明示
## コメント方針（優先順位順）
1. 自己説明的なコードを書く（命名で意図を表現）
2. 「なぜ」を説明する必要がある場合のみコメント
3. 「何を」するかの説明は避ける（コードから明らか）
```

## 品質チェックリスト

新しい instructions ファイルを作成する際の最終確認:

- [ ] YAML frontmatter が正しく記述されている
- [ ] `description` が明確で簡潔（1-500文字）
- [ ] `applyTo` パターンが適切に設定されている（必要な範囲のみ）
- [ ] Instructions は簡潔で自己完結的
- [ ] 具体的で実行可能な指示を含む（曖昧語を排除）
- [ ] 機密情報を含まない
- [ ] Good/Bad 例を含む
- [ ] WHY（理由）を簡潔に添えている
- [ ] 他のファイルへの参照を適切に使用
- [ ] チェックリストで実行可能な基準を提供
- [ ] チーム開発を考慮した構造
- [ ] 定期的なレビューとメンテナンスの計画がある
- [ ] 優先順位や衝突時の判断基準が明示されている
- [ ] セキュリティ要件が適切に含まれている
- [ ] パフォーマンスへの影響を考慮している

## 参考リソース

### 公式ドキュメント
- [VS Code - Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [VS Code - Prompt Files](https://code.visualstudio.com/docs/copilot/customization/prompt-files)
- [VS Code - Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [VS Code - Security](https://code.visualstudio.com/docs/copilot/security)

### コミュニティリソース
- [awesome-copilot](https://github.com/github/awesome-copilot)
- [awesome-copilot/instructions](https://github.com/github/awesome-copilot/tree/main/instructions)

### 関連ガイド
- セキュリティ: OWASP Top 10, Secure by Default
- パフォーマンス: Measure First, Profile Before Optimize
- コードレビュー: CRITICAL/IMPORTANT/SUGGESTION 分類
