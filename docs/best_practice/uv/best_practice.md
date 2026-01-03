# uvプロジェクト管理ベストプラクティス

**最終更新:** 2026年1月3日
**uvバージョン:** 0.9.21以降対応

## 目次

1. [概要](#概要)
2. [プロジェクト初期化](#プロジェクト初期化)
3. [依存関係管理](#依存関係管理)
4. [Python バージョン管理](#pythonバージョン管理)
5. [ロックファイルと環境同期](#ロックファイルと環境同期)
6. [ワークスペースとモノレポ](#ワークスペースとモノレポ)
7. [CI/CD統合](#cicd統合)
8. [Docker統合](#docker統合)
9. [設定とカスタマイズ](#設定とカスタマイズ)
10. [キャッシング戦略](#キャッシング戦略)
11. [セキュリティと再現性](#セキュリティと再現性)
12. [トラブルシューティング](#トラブルシューティング)

---

## 概要

uvは、Astral社が開発した超高速なPythonパッケージマネージャーおよびプロジェクト管理ツールです。Rustで実装されており、pip、pip-tools、poetryなどの従来のツールと比較して10-100倍高速です。

### 主な特徴

- **統合されたツールチェーン**: Pythonバージョン管理、依存関係解決、仮想環境管理を1つのツールで実現
- **高速**: Rust実装による高速な依存関係解決とインストール
- **ユニバーサル解決**: クロスプラットフォーム対応のロックファイル
- **pip互換性**: 既存のpip/pip-toolsワークフローからの移行が容易

---

## プロジェクト初期化

### 新規プロジェクトの作成

```bash
# 基本的なプロジェクトの作成
uv init my-project
cd my-project

# パッケージプロジェクト（ライブラリ）の作成
uv init --package my-library

# アプリケーションプロジェクトの作成
uv init --app my-app

# 特定のビルドバックエンドを指定
uv init --build-backend hatchling my-project
```

### プロジェクト構造のベストプラクティス

**推奨されるディレクトリ構造:**

```
my-project/
├── .python-version          # Pythonバージョン固定
├── .gitignore              # .venvを除外
├── pyproject.toml          # プロジェクトメタデータと依存関係
├── uv.lock                 # ロックファイル（バージョン管理に含める）
├── README.md
├── src/                    # パッケージの場合
│   └── my_project/
│       ├── __init__.py
│       └── main.py
├── tests/                  # テストコード
│   └── test_main.py
└── docs/                   # ドキュメント
```

**重要なポイント:**

- `.venv/` は `.gitignore` に追加（プラットフォーム固有）
- `uv.lock` はバージョン管理に含める（再現性のため）
- `.python-version` でPythonバージョンを固定

---

## 依存関係管理

### 依存関係の追加と削除

```bash
# 本番依存関係の追加
uv add requests
uv add "httpx>=0.24.0"
uv add "pandas>=2.0,<3.0"

# 開発依存関係の追加
uv add --dev pytest
uv add --dev ruff black mypy

# オプショナル依存関係の追加
uv add --optional extra1 matplotlib

# 特定のグループに追加
uv add --group lint ruff
uv add --group test pytest

# 依存関係の削除
uv remove requests
```

### 依存関係のソース指定

```bash
# Git リポジトリから
uv add git+https://github.com/user/repo
uv add git+https://github.com/user/repo --tag v1.0.0
uv add git+https://github.com/user/repo --branch main

# ローカルパスから
uv add --editable ./path/to/package

# URL から
uv add https://files.pythonhosted.org/packages/.../package.whl

# 特定のインデックスから
uv add torch --index pytorch=https://download.pytorch.org/whl/cpu
```

### pyproject.toml での依存関係管理

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "Project description"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31.0",
    "pandas>=2.0.0,<3.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "ruff>=0.1.0",
]
ml = [
    "scikit-learn>=1.3.0",
    "numpy>=1.24.0",
]

[dependency-groups]
lint = ["ruff>=0.1.0", "mypy>=1.0.0"]
test = ["pytest>=7.0.0", "pytest-cov>=4.0.0"]

[tool.uv.sources]
# ローカルパッケージの指定
my-lib = { path = "../my-lib", editable = true }

# Git ソースの指定
experimental = { git = "https://github.com/org/experimental", tag = "v0.1.0" }
```

### プラットフォーム固有の依存関係

```toml
[project]
dependencies = [
    # Linux のみ
    "jax; sys_platform == 'linux'",

    # macOS のみ
    "pyobjc; sys_platform == 'darwin'",

    # Windows のみ
    "pywin32; sys_platform == 'win32'",

    # Python 3.11 以降
    "numpy>=2.0.0; python_version >= '3.11'",
]
```

### 下限バージョンの重要性

**ベストプラクティス:**

- すべての直接依存関係に下限バージョンを指定
- ライブラリ開発時は特に重要
- `--resolution lowest` でテストして互換性を確認

```bash
# 最低バージョンでテスト
uv sync --resolution lowest
uv run pytest

# 最新バージョンでテスト
uv sync --resolution latest
uv run pytest
```

---

## Pythonバージョン管理

### Pythonのインストールと管理

```bash
# 特定バージョンのインストール
uv python install 3.12
uv python install 3.11.7

# 複数バージョンのインストール
uv python install 3.10 3.11 3.12

# インストール済みバージョンの確認
uv python list --only-installed

# 利用可能なバージョンの確認
uv python list

# Pythonバージョンの検索
uv python find 3.12
```

### プロジェクトでのPythonバージョン固定

```bash
# プロジェクトのPythonバージョンを固定
uv python pin 3.12

# グローバルなデフォルトを設定
uv python pin --global 3.12
```

**pyproject.toml での指定:**

```toml
[project]
requires-python = ">=3.10"
```

### Python管理のベストプラクティス

1. **プロジェクトごとにバージョンを固定**: `.python-version` ファイルを使用
2. **requires-python の範囲を適切に設定**: 過度に広範囲にしない
3. **複数バージョンでテスト**: CI/CDでマトリックステスト
4. **マネージドPython を優先**: システムPythonに依存しない

```toml
# 設定例
[tool.uv]
python-preference = "managed"  # マネージドPythonを優先
```

---

## ロックファイルと環境同期

### ロックファイルの管理

```bash
# ロックファイルの作成/更新
uv lock

# ロックファイルの整合性確認
uv lock --check

# 環境の同期
uv sync

# ロックファイル整合性を確認して同期
uv sync --locked

# 変更を無視して同期（CI用）
uv sync --frozen
```

### 同期のカスタマイズ

```bash
# すべてのextraを含める
uv sync --all-extras

# 特定のextraのみ
uv sync --extra ml --extra dev

# 開発依存関係を除外
uv sync --no-dev

# 特定のグループのみ
uv sync --only-group lint

# 複数のグループ
uv sync --group lint --group test
```

### パッケージのアップグレード

```bash
# すべての依存関係をアップグレード
uv lock --upgrade

# 特定のパッケージをアップグレード
uv lock --upgrade-package requests
uv lock --upgrade-package "pandas>=2.1"

# 最新情報を取得（メタデータのリフレッシュ）
uv sync --refresh

# 特定パッケージのキャッシュをクリア
uv sync --refresh-package numpy
```

### 環境の実行

```bash
# プロジェクト環境でコマンド実行
uv run python script.py
uv run pytest
uv run python -m myapp

# 自動同期を無効化
uv run --no-sync pytest

# ロックファイル確認を強制
uv run --locked python script.py
```

---

## ワークスペースとモノレポ

### ワークスペースの設定

**ルートの `pyproject.toml`:**

```toml
[project]
name = "my-workspace"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = []

[tool.uv.workspace]
members = ["packages/*", "apps/*"]
exclude = ["packages/experimental"]

[tool.uv.sources]
# ワークスペース共通のソース設定
common-lib = { workspace = true }
```

**メンバーの `pyproject.toml`:**

```toml
[project]
name = "my-package"
version = "0.1.0"
dependencies = [
    "common-lib",  # ワークスペースメンバー
    "requests",
]

[tool.uv.sources]
common-lib = { workspace = true }
```

### ワークスペースのコマンド

```bash
# ワークスペース全体をロック
uv lock

# 特定パッケージで実行
uv run --package my-package python script.py

# ワークスペースルートで実行（デフォルト）
uv run pytest
```

### ワークスペース使用のガイドライン

**使用すべき場合:**

- 密結合した複数パッケージの開発
- モノレポ構成のプロジェクト
- 共通の依存関係を共有したい場合

**使用を避けるべき場合:**

- 競合する依存関係がある
- メンバーごとに独立した仮想環境が必要
- 異なる Python バージョンをサポートする必要がある

---

## CI/CD統合

### GitHub Actions

**基本的なセットアップ:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v5

      - name: Install uv
        uses: astral-sh/setup-uv@v7
        with:
          version: "0.9.21"  # バージョン固定を推奨
          enable-cache: true

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --locked --all-extras

      - name: Run tests
        run: uv run pytest

      - name: Run linter
        run: uv run ruff check
```

**マトリックステスト:**

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, macos-latest, windows-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v5

      - name: Install uv
        uses: astral-sh/setup-uv@v7
        with:
          python-version: ${{ matrix.python-version }}
```

**キャッシング戦略:**

```yaml
- name: Restore uv cache
  uses: actions/cache@v4
  with:
    path: /tmp/.uv-cache
    key: uv-${{ runner.os }}-${{ hashFiles('uv.lock') }}
    restore-keys: |
      uv-${{ runner.os }}-

- name: Install dependencies
  run: uv sync --locked
  env:
    UV_CACHE_DIR: /tmp/.uv-cache

- name: Minimize cache
  run: uv cache prune --ci
```

### GitLab CI/CD

```yaml
image: ghcr.io/astral-sh/uv:python3.12-bookworm

variables:
  UV_CACHE_DIR: ${CI_PROJECT_DIR}/.cache/uv
  UV_PYTHON_PREFERENCE: "only-managed"

cache:
  paths:
    - .cache/uv

stages:
  - test
  - build

test:
  stage: test
  script:
    - uv sync --locked
    - uv run pytest
    - uv run ruff check

build:
  stage: build
  script:
    - uv build
  artifacts:
    paths:
      - dist/
```

---

## Docker統合

### 基本的な Dockerfile

```dockerfile
FROM python:3.12-slim-bookworm

# uv のインストール
COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

# 作業ディレクトリの設定
WORKDIR /app

# 依存関係のインストール（レイヤーキャッシュ最適化）
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev

# プロジェクトのコピー
COPY . /app

# プロジェクトのインストール
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

# 環境変数の設定
ENV PATH="/app/.venv/bin:$PATH"

# アプリケーションの起動
CMD ["python", "-m", "myapp"]
```

### 本番環境向けの最適化

```dockerfile
# マルチステージビルド
FROM python:3.12-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# バイトコードコンパイル有効化
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev --no-editable

COPY . /app

RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-editable

# 実行イメージ
FROM python:3.12-slim

COPY --from=builder --chown=app:app /app /app

WORKDIR /app
ENV PATH="/app/.venv/bin:$PATH"

CMD ["python", "-m", "myapp"]
```

### Dockerのベストプラクティス

1. **バージョン固定**: uvバージョンをSHA256で固定
2. **レイヤーキャッシング**: 依存関係とアプリコードを分離
3. **バイトコードコンパイル**: `UV_COMPILE_BYTECODE=1`
4. **non-editable インストール**: 本番では `--no-editable`
5. **マルチステージビルド**: イメージサイズを最小化

---

## 設定とカスタマイズ

### プロジェクト設定

**pyproject.toml:**

```toml
[tool.uv]
# Python設定
python-preference = "managed"
python-downloads = "manual"  # 自動ダウンロードを無効化

# 依存関係解決
resolution = "highest"  # または "lowest-direct"
prerelease = "if-necessary"

# インデックス設定
[[tool.uv.index]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cpu"
explicit = true

# 環境設定
environments = [
    "sys_platform == 'linux'",
    "sys_platform == 'darwin'",
]

# デフォルトグループ
default-groups = ["dev", "lint"]

# 競合する依存関係の宣言
conflicts = [
    [
        { extra = "gpu" },
        { extra = "cpu" },
    ],
]
```

### ユーザーレベル設定

**~/.config/uv/uv.toml:**

```toml
# グローバルなPython設定
python-preference = "managed"

# キャッシュ設定
cache-dir = "~/.cache/uv"

# デフォルトインデックス
[[index]]
url = "https://pypi.org/simple"
default = true

# プライベートインデックス
[[index]]
name = "private"
url = "https://private.pypi.org/simple"
```

### 環境変数での設定

```bash
# Python設定
export UV_PYTHON_PREFERENCE=managed
export UV_PYTHON_DOWNLOADS=manual

# キャッシュ設定
export UV_CACHE_DIR=/path/to/cache
export UV_NO_CACHE=1

# システムPython使用
export UV_SYSTEM_PYTHON=1

# プロジェクト環境のカスタマイズ
export UV_PROJECT_ENVIRONMENT=.venv
```

---

## キャッシング戦略

### ローカル開発

```bash
# キャッシュのクリア
uv cache clean

# 特定パッケージのキャッシュをクリア
uv cache clean numpy

# 未使用のキャッシュをクリーンアップ
uv cache prune

# キャッシュディレクトリの確認
uv cache dir
```

### CI/CD でのキャッシング

**GitHub Actions:**

```yaml
- name: Cache uv
  uses: actions/cache@v4
  with:
    path: ~/.cache/uv
    key: uv-${{ runner.os }}-${{ hashFiles('uv.lock') }}

- name: Minimize cache
  run: uv cache prune --ci
```

**GitLab CI:**

```yaml
cache:
  key:
    files:
      - uv.lock
  paths:
    - .cache/uv
```

### キャッシュの最適化

```toml
[tool.uv]
# 動的メタデータのキャッシュキー
cache-keys = [
    { file = "pyproject.toml" },
    { git = { commit = true } },
    { env = "MY_ENV_VAR" },
]

# 常に再ビルドが必要なパッケージ
reinstall-package = ["my-dynamic-package"]
```

---

## セキュリティと再現性

### 再現可能なインストール

```bash
# 特定日付以降のパッケージを除外
uv sync --exclude-newer 2024-01-01

# 特定パッケージのクールダウン期間設定
uv sync --exclude-newer-package "requests=7 days"
```

**pyproject.toml での設定:**

```toml
[tool.uv]
# グローバルなクールダウン
exclude-newer = "1 week"

# パッケージ固有のクールダウン
[tool.uv.exclude-newer-package]
requests = "30 days"
numpy = "14 days"
```

### プライベートリポジトリの認証

**Git認証:**

```toml
[tool.uv.sources]
private-pkg = {
    git = "https://github.com/org/private-repo",
    tag = "v1.0.0"
}
```

**環境変数での認証:**

```bash
# GitHub Actions
- name: Configure Git credentials
  run: |
    echo "${{ secrets.GITHUB_TOKEN }}" | gh auth login --with-token
    gh auth setup-git
```

**HTTP認証:**

```bash
# 環境変数
export UV_INDEX_my_index_USERNAME=user
export UV_INDEX_my_index_PASSWORD=pass
```

### セキュリティのベストプラクティス

1. **依存関係のクールダウン**: 新しいバージョンを即座に使用しない
2. **ロックファイルの検証**: `--locked` を CI で使用
3. **プライベート依存関係**: 適切な認証設定
4. **監査ツールの使用**: `pip-audit` などとの統合

---

## トラブルシューティング

### よくある問題と解決策

**1. ロックファイルが古い**

```bash
# エラー: The lockfile is out of date
uv lock
uv sync
```

**2. 依存関係の競合**

```bash
# 詳細なエラー情報を表示
uv sync --verbose

# 最低バージョンで試す
uv sync --resolution lowest

# 特定パッケージをアップグレード
uv lock --upgrade-package problematic-package
```

**3. ビルドの失敗**

```toml
# ビルド分離を無効化
[tool.uv]
no-build-isolation-package = ["problem-package"]

# ビルド依存関係を追加
[tool.uv.extra-build-dependencies]
problem-package = ["setuptools", "wheel"]
```

**4. キャッシュの問題**

```bash
# キャッシュを完全にクリア
uv cache clean

# キャッシュを無効化して実行
uv sync --no-cache

# 特定パッケージを再インストール
uv sync --reinstall-package numpy
```

**5. Python バージョンの問題**

```bash
# Python を再インストール
uv python uninstall 3.12
uv python install 3.12

# システムPythonを使用
uv venv --python /usr/bin/python3.12
```

### デバッグのヒント

```bash
# 詳細ログ
uv --verbose sync

# トレースレベルログ
uv -vv sync

# 環境情報の確認
uv python list --only-installed
uv cache dir
uv --version
```

---

## まとめ

### uvを使う主な利点

1. **速度**: 従来ツールの10-100倍高速
2. **統合性**: 1つのツールですべてを管理
3. **信頼性**: ユニバーサル解決とロックファイル
4. **互換性**: 既存のエコシステムとの互換性

### 重要なベストプラクティス

✅ `.venv` を `.gitignore` に追加
✅ `uv.lock` をバージョン管理に含める
✅ Python バージョンを `.python-version` で固定
✅ すべての依存関係に下限バージョンを指定
✅ CI/CD で `--locked` を使用
✅ 本番環境では `--no-dev` を使用
✅ Docker で適切なキャッシング戦略を使用
✅ 複数の Python バージョンでテスト

### 参考リソース

- [公式ドキュメント](https://docs.astral.sh/uv/)
- [GitHub リポジトリ](https://github.com/astral-sh/uv)
- [Discord コミュニティ](https://discord.com/invite/astral-sh)
- [PyPI](https://pypi.org/project/uv/)

---

**このドキュメントは、uv 0.9.21 の公式ドキュメントに基づいて作成されています。**
