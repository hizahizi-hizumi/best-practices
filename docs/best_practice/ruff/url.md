# Ruff ベストプラクティス参考URL集

Ruff公式ドキュメントからベストプラクティス作成に参考となるURLをカテゴリ別にまとめました。

## 基本情報・概要

### メインドキュメント
- **Ruff 公式トップページ**: https://docs.astral.sh/ruff/
  - Ruffの全体概要と機能説明
  
- **チュートリアル**: https://docs.astral.sh/ruff/tutorial/
  - 初めてRuffを使う際の入門ガイド
  - 基本的な使い方とベストプラクティスの紹介

- **インストール方法**: https://docs.astral.sh/ruff/installation/
  - 様々なインストール方法（pip, uv, cargo, standalone installer等）

## Linter（リンター）

### Linterの基本
- **Ruff Linter**: https://docs.astral.sh/ruff/linter/
  - Linterの使い方、ルール選択、修正機能
  - Safe/Unsafe fixesの解説
  - Error suppressionの方法（noqa コメント）

### ルール一覧
- **Rules（ルール一覧）**: https://docs.astral.sh/ruff/rules/
  - 800以上のルールの完全なリファレンス
  - カテゴリ別（F, E, W, B, など）のルール
  - 各ルールの説明とコード例

## Formatter（フォーマッター）

- **Ruff Formatter**: https://docs.astral.sh/ruff/formatter/
  - Blackとの互換性
  - フォーマットの哲学とスタイルガイド
  - Docstringのフォーマット
  - Conflicting lint rulesの扱い

## 設定・Configuration

### 設定ファイル
- **Configuring Ruff**: https://docs.astral.sh/ruff/configuration/
  - pyproject.toml / ruff.toml / .ruff.toml の使い方
  - Config file discovery（設定ファイルの検索方法）
  - 階層的な設定の扱い
  - Python versionの推論
  - Jupyter Notebookのサポート
  - コマンドラインインターフェース
  - Shell autocompletion

### 設定オプション
- **Settings（設定一覧）**: https://docs.astral.sh/ruff/settings/
  - 全設定オプションの完全なリファレンス
  - Top-levelオプション（line-length, target-version, etc.）
  - Linter固有の設定
  - Formatter固有の設定
  - プラグイン別設定（flake8-*, isort, pydocstyle, etc.）

## エディタ統合

### エディタ連携
- **Editor Integration**: https://docs.astral.sh/ruff/editors/
  - Language Server Protocol (LSP) の説明
  - `ruff server` の使い方

- **Setup**: https://docs.astral.sh/ruff/editors/setup/
  - VS Code, PyCharm, Neovim等の設定方法

- **Features**: https://docs.astral.sh/ruff/editors/features/
  - エディタで利用可能な機能一覧

- **Settings**: https://docs.astral.sh/ruff/editors/settings/
  - エディタ統合の設定オプション

## CI/CD統合

- **Integrations**: https://docs.astral.sh/ruff/integrations/
  - GitHub Actions での使い方
  - GitLab CI/CD での使い方
  - pre-commit での使い方
  - Docker イメージの使い方

## FAQ・トラブルシューティング

- **FAQ**: https://docs.astral.sh/ruff/faq/
  - よくある質問と回答
  - Blackとの互換性
  - Flake8/Pylintとの比較
  - Type checker（Mypy, Pyright）との使い分け
  - Import sortingの動作
  - Jupyter Notebookのサポート
  - NumPy/Google-style docstringsのサポート

## 詳細トピック

### Preview機能
- **Preview**: https://docs.astral.sh/ruff/preview/
  - 実験的・不安定な機能の説明
  - Previewルールの一覧

### Versioning
- **Versioning**: https://docs.astral.sh/ruff/versioning/
  - Ruffのバージョニング方針
  - 後方互換性に関する情報

## 特定のルールセット・プラグイン

### Pyflakes (F)
- **Rules - Pyflakes (F)**: https://docs.astral.sh/ruff/rules/#pyflakes-f
  - 未使用import、未定義変数等の検出

### pycodestyle (E, W)
- **Rules - pycodestyle (E, W)**: https://docs.astral.sh/ruff/rules/#pycodestyle-e-w
  - PEP 8準拠のスタイルチェック

### flake8-bugbear (B)
- **Rules - flake8-bugbear (B)**: https://docs.astral.sh/ruff/rules/#flake8-bugbear-b
  - 一般的なバグパターンの検出

### isort (I)
- **Rules - isort (I)**: https://docs.astral.sh/ruff/rules/#isort-i
  - Import文のソート
- **Settings - isort**: https://docs.astral.sh/ruff/settings/#lintisort
  - isort設定の詳細

### pydocstyle (D)
- **Rules - pydocstyle (D)**: https://docs.astral.sh/ruff/rules/#pydocstyle-d
  - Docstringのスタイルチェック
- **Settings - pydocstyle**: https://docs.astral.sh/ruff/settings/#lintpydocstyle
  - Convention設定（google, numpy, pep257）

### pyupgrade (UP)
- **Rules - pyupgrade (UP)**: https://docs.astral.sh/ruff/rules/#pyupgrade-up
  - Python versionに応じた自動アップグレード

### Pylint (PL)
- **Rules - Pylint (PL)**: https://docs.astral.sh/ruff/rules/#pylint-pl
  - Pylintスタイルのチェック

### その他の有用なプラグイン
- **flake8-annotations (ANN)**: https://docs.astral.sh/ruff/rules/#flake8-annotations-ann
- **flake8-bandit (S)**: https://docs.astral.sh/ruff/rules/#flake8-bandit-s
- **flake8-comprehensions (C4)**: https://docs.astral.sh/ruff/rules/#flake8-comprehensions-c4
- **flake8-simplify (SIM)**: https://docs.astral.sh/ruff/rules/#flake8-simplify-sim
- **flake8-type-checking (TC)**: https://docs.astral.sh/ruff/rules/#flake8-type-checking-tc

## 実装・開発者向け

- **Contributing**: https://docs.astral.sh/ruff/contributing/
  - Ruffへの貢献方法
  - 開発環境のセットアップ

## 公式リポジトリ

- **GitHub Repository**: https://github.com/astral-sh/ruff
  - ソースコード
  - Issue報告
  - Discussions

---

## ベストプラクティス作成時の重点ポイント

1. **設定ファイルの構造**: Configuration と Settings を参照
2. **ルール選択の方針**: Rules と FAQ を参照
3. **Formatter設定**: Formatter と Settings（format section）を参照
4. **CI/CD統合**: Integrations を参照
5. **エディタ統合**: Editor Integration を参照
6. **移行ガイド**: FAQ（Blackからの移行、Flake8からの移行）を参照

## 注意事項

- このURL集はRuff v0.14.10時点の情報を基にしています
- 最新情報は公式ドキュメント（https://docs.astral.sh/ruff/）を確認してください
- Previewルールは不安定な可能性があるため、本番環境での使用は慎重に検討してください
