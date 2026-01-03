# uvプロジェクト管理ベストプラクティス - 参考URL集

uvのプロジェクト管理に関するベストプラクティスを作成する際に参考にするべき重要なドキュメントのURL一覧です。

## 概要・導入

### メインドキュメント
- [uv公式ドキュメント](https://docs.astral.sh/uv/)
  - uvの全体的な概要と主要機能の説明
  - Getting Started、Guides、Concepts、Referenceなどの主要セクションへのアクセス

### 入門
- [Getting Started - First Steps](https://docs.astral.sh/uv/getting-started/first-steps/)
  - uvの基本的な使い方とクイックスタートガイド
  - プロジェクト作成から依存関係管理までの基本フロー

- [Installation](https://docs.astral.sh/uv/getting-started/installation/)
  - uvのインストール方法（各種OS、パッケージマネージャー）
  - スタンドアロンインストーラー、Homebrew、pipx等による導入

## プロジェクト管理

### プロジェクトの基礎
- [Projects - Concepts](https://docs.astral.sh/uv/concepts/projects/)
  - プロジェクトとは何か、uvがどのようにプロジェクトを管理するかの説明
  - プロジェクトベースのワークフローの理解

- [Working on Projects - Guide](https://docs.astral.sh/uv/guides/projects/)
  - プロジェクトでの実際の作業方法
  - 開発からテスト、ビルドまでの実践的なガイド

- [Project Structure and Files](https://docs.astral.sh/uv/concepts/projects/layout/)
  - プロジェクトのディレクトリ構造とファイル配置
  - pyproject.toml、uv.lock、src/などの標準レイアウト

- [Creating Projects](https://docs.astral.sh/uv/concepts/projects/init/)
  - `uv init`によるプロジェクト初期化
  - テンプレート、パッケージ、アプリケーションの作成

### 依存関係管理
- [Managing Dependencies](https://docs.astral.sh/uv/concepts/projects/dependencies/)
  - 依存関係の追加、削除、更新方法
  - `uv add`、`uv remove`コマンドの使用法
  - オプショナル依存関係、依存関係グループの管理

- [Locking and Syncing](https://docs.astral.sh/uv/concepts/projects/sync/)
  - `uv.lock`ファイルによる依存関係のロック
  - `uv sync`による環境の同期と更新
  - ロックファイルの役割と重要性

- [Resolution - Concepts](https://docs.astral.sh/uv/concepts/resolution/)
  - 依存関係解決のメカニズム
  - プラットフォーム固有解決とユニバーサル解決
  - 解決戦略（lowest、latest）、制約、オーバーライド
  - マルチバージョン解決、競合する依存関係の処理

### プロジェクト設定
- [Configuring Projects](https://docs.astral.sh/uv/concepts/projects/config/)
  - pyproject.tomlでのプロジェクト設定
  - ビルド設定、環境変数、カスタムスクリプト
  - tool.uvセクションの活用

- [Configuration Files - Concepts](https://docs.astral.sh/uv/concepts/configuration-files/)
  - 設定ファイルの種類と優先順位
  - ユーザーレベル、プロジェクトレベルの設定
  - 環境変数による設定のオーバーライド

### ワークスペース・モノレポ
- [Using Workspaces](https://docs.astral.sh/uv/concepts/projects/workspaces/)
  - ワークスペースによる複数パッケージの管理
  - モノレポ構造の実装
  - ワークスペースメンバー間の依存関係

### コマンド実行
- [Running Commands](https://docs.astral.sh/uv/concepts/projects/run/)
  - `uv run`によるコマンド実行
  - プロジェクト環境でのスクリプト実行
  - 環境の自動作成と管理

## ビルド・公開

### パッケージビルド
- [Building Distributions](https://docs.astral.sh/uv/concepts/projects/build/)
  - wheelとsource distributionのビルド
  - `uv build`コマンドの使用法
  - ビルド設定とカスタマイズ

- [Publishing Packages - Guide](https://docs.astral.sh/uv/guides/package/)
  - PyPIへのパッケージ公開手順
  - `uv publish`の使用方法
  - 認証とトークン管理

### ロックファイルエクスポート
- [Exporting Lockfiles](https://docs.astral.sh/uv/concepts/projects/export/)
  - uv.lockからrequirements.txtへの変換
  - 他のツールとの互換性
  - デプロイメント用のエクスポート

## CI/CD統合

### GitHub Actions
- [GitHub Actions Integration](https://docs.astral.sh/uv/guides/integration/github/)
  - GitHub ActionsでのuvのセットアップとCI/CD構築
  - setup-uvアクションの使用
  - キャッシング戦略とマトリックステスト
  - PyPIへのpublish設定（trusted publishers）

### GitLab CI/CD
- [GitLab CI/CD Integration](https://docs.astral.sh/uv/guides/integration/gitlab/)
  - GitLab CI/CDでのuvの使用
  - Dockerイメージの選択とキャッシング設定
  - パイプライン構成のベストプラクティス

### Docker
- [Docker Integration](https://docs.astral.sh/uv/guides/integration/docker/)
  - DockerでのuvのインストールとPython環境構築
  - 公式Dockerイメージの使用（distroless、alpine、debian）
  - マルチステージビルドによる最適化
  - キャッシュマウント、バイトコードコンパイル、非編集可能インストール
  - イメージサイズの最小化テクニック

## Python管理

### Pythonバージョン
- [Python Versions - Concepts](https://docs.astral.sh/uv/concepts/python-versions/)
  - Pythonバージョンの管理と自動ダウンロード
  - `uv python install`、`uv python list`の使用
  - システムPythonとマネージドPythonの違い
  - .python-versionファイルによるバージョン固定
  - free-threadedやdebugビルドなどのバリアント管理

- [Installing Python - Guide](https://docs.astral.sh/uv/guides/install-python/)
  - Pythonインストールの実践ガイド
  - 複数バージョンの管理
  - 実行可能ファイルのPATHへの追加

## スクリプト・ツール

### スクリプト実行
- [Running Scripts - Guide](https://docs.astral.sh/uv/guides/scripts/)
  - スタンドアロンPythonスクリプトの実行
  - インラインスクリプトメタデータ（PEP 723）による依存関係宣言
  - shebangによる実行可能スクリプト
  - スクリプトロックとreproducibility

### ツール管理
- [Using Tools - Guide](https://docs.astral.sh/uv/guides/tools/)
  - `uvx`/`uv tool run`による一時的なツール実行
  - `uv tool install`によるツールのインストール
  - ツールバージョン管理とアップグレード
  - プラグインや追加依存関係の指定

- [Tools - Concepts](https://docs.astral.sh/uv/concepts/tools/)
  - ツール環境の管理と分離
  - ツールバージョンとアップグレード戦略
  - 実行 vs インストールの使い分け

## パッケージインデックス・認証

### パッケージインデックス
- [Package Indexes - Concepts](https://docs.astral.sh/uv/concepts/indexes/)
  - PyPI以外の代替インデックスの使用
  - プライベートインデックスの設定
  - インデックスの優先順位と検索戦略
  - パッケージのインデックスへのピン止め
  - flat indexesの使用

### 認証
- [Authentication - Concepts](https://docs.astral.sh/uv/concepts/authentication/)
  - プライベートリポジトリへの認証
  - HTTP認証、Git認証、TLS証明書
  - 認証情報の提供方法（環境変数、keyring、netrc）

## キャッシュ・パフォーマンス

### キャッシング
- [Caching - Concepts](https://docs.astral.sh/uv/concepts/cache/)
  - uvのキャッシュメカニズム
  - 依存関係キャッシュのセマンティクス
  - `uv cache clean`、`uv cache prune`の使用
  - CI環境でのキャッシング戦略（`--ci`フラグ）
  - キャッシュディレクトリの設定とバージョニング

## 移行ガイド

### 他ツールからの移行
- [Migration from pip to uv project](https://docs.astral.sh/uv/guides/migration/pip-to-project/)
  - pip/pip-toolsからuvプロジェクトへの移行
  - requirements.txtからpyproject.tomlへの変換
  - 既存プロジェクトのuvプロジェクト化

## リファレンス

### コマンドリファレンス
- [Commands Reference](https://docs.astral.sh/uv/reference/cli/)
  - 全uvコマンドの詳細リファレンス
  - オプション、引数、使用例

### 設定リファレンス
- [Settings Reference](https://docs.astral.sh/uv/reference/settings/)
  - 設定オプションの完全なリスト
  - pyproject.tomlでの設定方法
  - 環境変数との対応

### ストレージ
- [Storage Reference](https://docs.astral.sh/uv/reference/storage/)
  - uvのファイルストレージ構造
  - キャッシュディレクトリ、ツールディレクトリ
  - データディレクトリの配置とクリーンアップ

## 追加のインテグレーション

### その他の統合
- [Alternative Package Indexes](https://docs.astral.sh/uv/guides/integration/alternative-indexes/)
  - AWS、Azure、GCPなどのプライベートインデックスとの統合
  - 特定プロバイダーでの認証方法

- [FastAPI Integration](https://docs.astral.sh/uv/guides/integration/fastapi/)
  - FastAPIプロジェクトでのuvの使用
  - Web開発ワークフローとの統合

- [PyTorch Integration](https://docs.astral.sh/uv/guides/integration/pytorch/)
  - PyTorchなど大規模MLライブラリの管理
  - CUDA、ROCmなどのバリアントの扱い

---

## 参考情報

- **公式リポジトリ**: [astral-sh/uv on GitHub](https://github.com/astral-sh/uv)
- **ドキュメントルート**: [https://docs.astral.sh/uv/](https://docs.astral.sh/uv/)
- **パッケージ**: [uv on PyPI](https://pypi.org/project/uv/)

このURL集は、uvを使用したPythonプロジェクト管理のベストプラクティスを構築する際の包括的なリファレンスとして活用できます。各URLには、プロジェクトライフサイクル全体をカバーする詳細な情報が含まれています。
