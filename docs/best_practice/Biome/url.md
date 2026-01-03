# Biome ベストプラクティス作成のための参考URL

Biomeの公式ドキュメントから、ベストプラクティスを作成する際に参考にすべきドキュメントURLをまとめました。

## 基本情報

### はじめに・セットアップ
- [はじめる](https://biomejs.dev/ja/guides/getting-started)
  - インストール方法、初期設定、基本的な使い方
- [手動インストール](https://biomejs.dev/ja/guides/manual-installation)
  - Node.jsを使わないスタンドアロンインストール方法

## 設定

### 設定ファイル (biome.json)
- [Biome の設定](https://biomejs.dev/ja/guides/configure-biome)
  - 設定ファイルの構成、解決方法、処理するファイルの指定
- [設定リファレンス](https://biomejs.dev/ja/reference/configuration)
  - すべての設定オプションの詳細リファレンス
  - `$schema`, `extends`, `files`, `vcs`, `linter`, `formatter`, `organizeImports`, `javascript`, `json`, `css`, `graphql`, `html`, `overrides` 等

### 大規模プロジェクト
- [大規模プロジェクトでのBiomeの使用方法](https://biomejs.dev/ja/guides/big-projects)
  - ネストされた設定ファイル、パフォーマンスの最適化

## Formatter（フォーマッター）

### フォーマッターの基本
- [Formatter イントロダクション](https://biomejs.dev/ja/formatter)
  - オプション、CLI使用方法、設定、コードのフォーマット無効化

### Formatter詳細
- [Prettier との違い](https://biomejs.dev/ja/formatter/differences-with-prettier)
  - Prettierとの動作の違い
- [Formatterオプションに対する考え方](https://biomejs.dev/ja/formatter/option-philosophy)
  - オプション設計の哲学

### 主要オプション
- インデントスタイル (`indentStyle`, `indentWidth`)
- 行幅 (`lineWidth`)
- 改行コード (`lineEnding`)
- JavaScript固有: `quoteStyle`, `semicolons`, `trailingCommas`, `arrowParentheses` 等

## Linter（リンター）

### リンターの基本
- [Linter イントロダクション](https://biomejs.dev/ja/linter)
  - CLI使用方法、ルール、設定、コードの無視、安全な修正/安全ではない修正

### リンタールール
- [ドメイン](https://biomejs.dev/ja/linter/domains)
  - ルールのドメイン分類
- [プラグイン](https://biomejs.dev/ja/linter/plugins)
  - プラグインシステム
- [JavaScriptルール](https://biomejs.dev/ja/linter/javascript/rules)
  - JavaScript用リントルール一覧
- [JavaScriptルールのソース](https://biomejs.dev/ja/linter/javascript/sources)
  - 各ルールの由来（ESLint等との対応）
- [CSSルール](https://biomejs.dev/ja/linter/css/rules)
  - CSS用リントルール
- [JSONルール](https://biomejs.dev/ja/linter/json/rules)
  - JSON用リントルール
- [GraphQLルール](https://biomejs.dev/ja/linter/graphql/rules)
  - GraphQL用リントルール
- [HTMLルール](https://biomejs.dev/ja/linter/html/rules)
  - HTML用リントルール

### ルール設定
- 推奨ルール (`rules.recommended`)
- ルールの重大度 (`error`, `warn`, `info`, `off`)
- グループ単位の設定 (`accessibility`, `complexity`, `correctness`, `nursery`, `performance`, `security`, `style`, `suspicious`)

## Assist（アシスト機能）

- [Assist イントロダクション](https://biomejs.dev/ja/assist)
  - コードアクションとリファクタリング支援
- [JavaScriptアクション](https://biomejs.dev/ja/assist/javascript/actions)
  - JavaScript用アクション一覧

## エディタ統合

### 公式拡張機能
- [Biome 公式拡張機能](https://biomejs.dev/ja/guides/editors/first-party-extensions)
  - VS Code, IntelliJ, Zed の公式拡張機能のインストール方法
- [VS Code拡張機能リファレンス](https://biomejs.dev/ja/reference/vscode)
  - VS Code拡張機能の詳細設定
- [Zed拡張機能リファレンス](https://biomejs.dev/ja/reference/zed)
  - Zed拡張機能の詳細設定

### サードパーティ
- [サードパーティの拡張機能](https://biomejs.dev/ja/guides/editors/third-party-extensions)
  - コミュニティによる拡張機能
- [エディタ拡張機能への Biome の統合](https://biomejs.dev/ja/guides/editors/create-an-extension)
  - 独自エディタ拡張機能の作成方法

## CI/CD統合

### 継続的インテグレーション
- [継続的インテグレーション](https://biomejs.dev/ja/recipes/continuous-integration)
  - GitHub Actions等でのBiome実行
  - サードパーティアクション (reviewdog-action-biome)

### Git統合
- [Git Hooks](https://biomejs.dev/ja/recipes/git-hooks)
  - pre-commit hookでのBiome使用
- [Biome をVCSと統合する](https://biomejs.dev/ja/guides/integrate-in-vcs)
  - .gitignore ファイルの使用
  - 変更されたファイルのみを処理 (`--changed`, `--since`)
  - ステージされたファイルのみを処理 (`--staged`)

### その他CI関連
- [Renovate](https://biomejs.dev/ja/recipes/renovate)
  - 依存関係の自動更新

## 既存プロジェクトへの移行

### ESLint/Prettierからの移行
- [ESLintとPrettierからの移行](https://biomejs.dev/ja/guides/migrate-eslint-prettier)
  - `biome migrate eslint` コマンド
  - `biome migrate prettier` コマンド
  - 設定の自動変換

### バージョンアップグレード
- [Biome v2へのアップグレード](https://biomejs.dev/ja/guides/upgrade-to-biome-v2)
  - v1からv2への移行ガイド

## パフォーマンス最適化

### パフォーマンス調査
- [パフォーマンスの問題を調査する](https://biomejs.dev/ja/guides/investigate-slowness)
  - トレーシング機能の使用方法
  - プロジェクトルールのパフォーマンスへの影響
  - `files.includes` での `!!` 構文の使用
  - ログレベルとトレーシング (`--log-level=tracing`, `--log-kind=json`, `--log-file`)

### パフォーマンスのヒント
- `dist/`, `build/` フォルダの除外
- `node_modules` の適切な除外設定
- プロジェクトルールの影響理解

## CLIリファレンス

### 主要コマンド
- [CLI リファレンス](https://biomejs.dev/ja/reference/cli)
  - `biome check` - フォーマット、リント、インポート整理を実行
  - `biome format` - フォーマット実行
  - `biome lint` - リント実行
  - `biome ci` - CI環境用コマンド（読み取り専用）
  - `biome init` - 設定ファイルの初期化
  - `biome migrate` - 設定の移行

### 主要オプション
- `--write` - 変更を書き込む
- `--unsafe` - 安全でない修正も適用
- `--staged` - ステージされたファイルのみ処理
- `--changed` - 変更されたファイルのみ処理
- `--since=REF` - 指定ブランチからの変更を処理
- `--only=RULE` - 特定のルールのみ実行
- `--skip=RULE` - 特定のルールをスキップ

## その他リファレンス

### 診断とログ
- [診断](https://biomejs.dev/ja/reference/diagnostics)
  - エラーメッセージの理解
- [環境変数](https://biomejs.dev/ja/reference/environment-variables)
  - Biomeの動作を制御する環境変数
- [リポータ](https://biomejs.dev/ja/reference/reporters)
  - 診断出力フォーマット

### 内部仕様
- [アーキテクチャ](https://biomejs.dev/ja/internals/architecture)
  - Biomeの内部アーキテクチャ
- [言語サポート](https://biomejs.dev/ja/internals/language-support)
  - サポートされている言語とファイル形式
- [理念](https://biomejs.dev/ja/internals/philosophy)
  - Biomeの設計思想
- [バージョニング](https://biomejs.dev/ja/internals/versioning)
  - バージョン管理ポリシー
- [Changelog](https://biomejs.dev/ja/internals/changelog/)
  - 変更履歴

### 高度な機能
- [GritQL](https://biomejs.dev/ja/reference/gritql)
  - コード検索とリファクタリングのためのクエリ言語
- [抑制](https://biomejs.dev/ja/analyzer/suppressions)
  - リントエラーの抑制方法

## その他のリソース

- [ソーシャルバッジ](https://biomejs.dev/ja/recipes/badges)
  - README用バッジの追加
- [プレイグラウンド](https://biomejs.dev/playground)
  - オンラインでBiomeを試す
- [Discord コミュニティ](https://biomejs.dev/chat)
  - コミュニティサポート

## 推奨される学習順序

1. **基本を理解する**
   - はじめる → 設定 → Formatter/Linter イントロダクション

2. **設定を深掘りする**
   - 設定リファレンス → ルール一覧 → CLIリファレンス

3. **実践的な統合**
   - エディタ統合 → CI/CD統合 → VCS統合

4. **既存プロジェクトへの導入**
   - ESLint/Prettier移行ガイド → 大規模プロジェクト

5. **最適化とトラブルシューティング**
   - パフォーマンス調査 → 診断 → アーキテクチャ
