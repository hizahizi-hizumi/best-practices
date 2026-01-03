---
description: 'uvプロジェクト管理のベストプラクティス規則'
applyTo: 'pyproject.toml, uv.lock, **/*.py'
---

# uv プロジェクト管理ベストプラクティス

GitHub Copilot が uv を使用した Python プロジェクト開発時に従うべきガイドラインとベストプラクティス。

## プロジェクトコンテキスト

- **対象ツール**: uv 0.9.21以降（Astral社の高速Pythonパッケージマネージャー）
- **主な特徴**: Rust実装、統合ツールチェーン、ユニバーサル解決
- **対応環境**: Python 3.10+、クロスプラットフォーム

## 一般的な指示

- すべての Python プロジェクトで uv を標準のパッケージマネージャーとして使用
- `pip`, `pip-tools`, `poetry` の代わりに uv コマンドを使用
- ロックファイル (`uv.lock`) を必ずバージョン管理に含める
- `.venv/` ディレクトリは `.gitignore` に追加
- Python バージョンは `.python-version` ファイルで固定
- CI/CD では必ず `--locked` フラグを使用して再現性を確保

## プロジェクト構造

推奨されるディレクトリ構造：

```
my-project/
├── .python-version          # Pythonバージョン固定（必須）
├── .gitignore              # .venvを除外
├── pyproject.toml          # プロジェクトメタデータと依存関係
├── uv.lock                 # ロックファイル（必ずVCS管理）
├── README.md
├── src/                    # パッケージコード
│   └── my_project/
│       ├── __init__.py
│       └── main.py
├── tests/                  # テストコード
│   └── test_main.py
└── docs/                   # ドキュメント
```

重要なファイル管理規則：
- `.venv/` → `.gitignore` に追加
- `uv.lock` → バージョン管理に含める（必須）
- `.python-version` → バージョン管理に含める（必須）

## 依存関係管理のベストプラクティス

### 依存関係の追加

```bash
# 本番依存関係
uv add requests
uv add "httpx>=0.24.0"

# 開発依存関係
uv add --dev pytest ruff mypy

# オプショナル依存関係
uv add --optional ml numpy pandas

# グループ依存関係（推奨）
uv add --group lint ruff mypy
uv add --group test pytest pytest-cov
```

### pyproject.toml のベストプラクティス

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "明確な説明"
requires-python = ">=3.10"
dependencies = [
    # 必ず下限バージョンを指定
    "requests>=2.31.0",
    "pandas>=2.0.0,<3.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "ruff>=0.1.0",
]

# グループ依存関係（推奨）
[dependency-groups]
lint = ["ruff>=0.1.0", "mypy>=1.0.0"]
test = ["pytest>=7.0.0", "pytest-cov>=4.0.0"]

[tool.uv]
# マネージドPythonを優先
python-preference = "managed"
# デフォルトグループの設定
default-groups = ["lint", "test"]
```

### バージョン指定の原則

- **必須**: すべての直接依存関係に下限バージョンを指定
- **推奨**: 上限バージョンは互換性が明確でない限り設定しない
- **避ける**: `==` での固定（ロックファイルに任せる）

### Good Example - 依存関係の適切な指定

```toml
[project]
dependencies = [
    "requests>=2.31.0",           # 下限のみ
    "pandas>=2.0.0,<3.0.0",       # メジャーバージョン制限
    "numpy>=1.24.0",              # 下限のみ
]
```

### Bad Example - 避けるべき指定

```toml
[project]
dependencies = [
    "requests",                    # バージョン指定なし
    "pandas==2.1.3",              # 固定（ロックファイルの役割）
    "numpy",                      # バージョン指定なし
]
```

## Python バージョン管理

### Pythonのインストールと固定

```bash
# 特定バージョンのインストール
uv python install 3.12

# プロジェクトのPythonバージョン固定（必須）
uv python pin 3.12

# 複数バージョンでのテスト
uv python install 3.10 3.11 3.12
```

### pyproject.toml での Python バージョン指定

```toml
[project]
# 過度に広範囲にしない
requires-python = ">=3.10"

[tool.uv]
# マネージドPythonを優先（推奨）
python-preference = "managed"
```

## 環境同期とロックファイル

### 基本的なワークフロー

```bash
# 依存関係を追加した後
uv add package-name

# ロックファイルを更新（自動で実行される）
uv lock

# 環境を同期
uv sync

# CI/CDでの実行（必須フラグ）
uv sync --locked

# コマンドを実行
uv run python script.py
uv run pytest
```

### 同期オプションの使い分け

```bash
# 開発環境: すべての依存関係
uv sync --all-extras

# 本番環境: 開発依存関係を除外
uv sync --no-dev --locked

# CI/CD: ロックファイルの整合性確認
uv sync --locked --frozen

# 特定グループのみ
uv sync --group lint --group test
```

### Good Example - CI/CD での使用

```yaml
- name: Install dependencies
  run: uv sync --locked --no-dev

- name: Run tests
  run: uv run pytest
```

### Bad Example - 避けるべきパターン

```yaml
# ロックファイルを無視している
- name: Install dependencies
  run: uv sync

# 開発依存関係を本番に含めている
- name: Install dependencies
  run: uv sync --all-extras
```

## CI/CD での使用

### GitHub Actions の推奨設定

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

### マトリックステストの推奨設定

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
          version: "0.9.21"
          enable-cache: true
      
      - name: Set up Python
        run: uv python install ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: uv sync --locked
      
      - name: Run tests
        run: uv run pytest
```

### キャッシング戦略（重要）

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

## Docker での使用

### 推奨される Dockerfile

```dockerfile
FROM python:3.12-slim-bookworm

# uvのインストール（バージョン固定推奨）
COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

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

CMD ["python", "-m", "myapp"]
```

### 本番環境向け最適化 Dockerfile

```dockerfile
# マルチステージビルド
FROM python:3.12-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

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

### Docker のベストプラクティス

- uv バージョンを明示的に固定
- レイヤーキャッシングを活用（依存関係とコードを分離）
- 本番環境では `--no-dev` を使用
- バイトコードコンパイルを有効化: `UV_COMPILE_BYTECODE=1`
- non-editable インストール: `--no-editable`
- マルチステージビルドでイメージサイズを最小化

## ワークスペースとモノレポ

### ワークスペースの設定

```toml
# ルートの pyproject.toml
[tool.uv.workspace]
members = ["packages/*", "apps/*"]
exclude = ["packages/experimental"]

# メンバーの pyproject.toml
[project]
name = "my-package"
dependencies = [
    "common-lib",  # ワークスペースメンバー
]

[tool.uv.sources]
common-lib = { workspace = true }
```

### ワークスペースを使用すべき場合

✅ 密結合した複数パッケージの開発
✅ モノレポ構成のプロジェクト
✅ 共通の依存関係を共有したい場合

### ワークスペースを避けるべき場合

❌ 競合する依存関係がある
❌ メンバーごとに独立した環境が必要
❌ 異なる Python バージョンをサポート

## よくあるパターン

### パターン1: プロジェクトの初期化

```bash
# 新規プロジェクトの作成
uv init my-project
cd my-project

# Pythonバージョンの固定
uv python pin 3.12

# 依存関係の追加
uv add requests pandas
uv add --group dev pytest ruff mypy
uv add --group test pytest pytest-cov

# Gitの初期化
git init
echo ".venv/" >> .gitignore
git add .
git commit -m "Initial commit"
```

### パターン2: 既存プロジェクトへの移行

```bash
# requirements.txt から移行
uv add --requirements requirements.txt

# poetry から移行（pyproject.tomlがある場合）
uv sync

# ロックファイルの生成
uv lock

# Gitに追加
git add pyproject.toml uv.lock .python-version
git commit -m "Migrate to uv"
```

### パターン3: 依存関係のアップグレード

```bash
# すべての依存関係をアップグレード
uv lock --upgrade

# 特定のパッケージをアップグレード
uv lock --upgrade-package requests

# 環境に同期
uv sync

# テストを実行して確認
uv run pytest
```

### パターン4: 複数バージョンでのテスト

```bash
# 最低バージョンでテスト
uv sync --resolution lowest
uv run pytest

# 最新バージョンでテスト
uv sync --resolution highest
uv run pytest

# 通常に戻す
uv sync
```

## 避けるべきパターン

### アンチパターン1: バージョン管理の不適切な実施

❌ **Bad**
```bash
# .venv をgitに含める
git add .venv/

# uv.lock を .gitignore に追加
echo "uv.lock" >> .gitignore

# .python-version を作成しない
```

✅ **Good**
```bash
# .venv を .gitignore に追加
echo ".venv/" >> .gitignore

# uv.lock をバージョン管理に含める
git add uv.lock .python-version

# .python-version で固定
uv python pin 3.12
```

### アンチパターン2: CI/CDでのロックファイル無視

❌ **Bad**
```yaml
- name: Install dependencies
  run: uv sync  # ロックファイルを無視
```

✅ **Good**
```yaml
- name: Install dependencies
  run: uv sync --locked  # ロックファイルを厳密に使用
```

### アンチパターン3: 依存関係のバージョン指定なし

❌ **Bad**
```toml
[project]
dependencies = [
    "requests",
    "pandas",
    "numpy",
]
```

✅ **Good**
```toml
[project]
dependencies = [
    "requests>=2.31.0",
    "pandas>=2.0.0,<3.0.0",
    "numpy>=1.24.0",
]
```

### アンチパターン4: 本番環境での開発依存関係の使用

❌ **Bad**
```dockerfile
RUN uv sync --all-extras  # 開発依存関係も含まれる
```

✅ **Good**
```dockerfile
RUN uv sync --locked --no-dev  # 本番依存関係のみ
```

### アンチパターン5: pip や poetry との混在使用

❌ **Bad**
```bash
uv add requests
pip install pandas  # 混在させない
poetry add numpy    # 混在させない
```

✅ **Good**
```bash
uv add requests pandas numpy  # uvのみを使用
```

## トラブルシューティング

### 問題1: ロックファイルが古い

```bash
# エラー: The lockfile is out of date
uv lock
uv sync
```

### 問題2: 依存関係の競合

```bash
# 詳細なエラー情報を表示
uv sync --verbose

# 最低バージョンで試す
uv sync --resolution lowest

# 特定パッケージをアップグレード
uv lock --upgrade-package problematic-package
```

### 問題3: キャッシュの問題

```bash
# キャッシュを完全にクリア
uv cache clean

# キャッシュを無効化して実行
uv sync --no-cache

# 特定パッケージを再インストール
uv sync --reinstall-package numpy
```

### 問題4: ビルドの失敗

```toml
[tool.uv]
# ビルド分離を無効化
no-build-isolation-package = ["problem-package"]

# ビルド依存関係を追加
[tool.uv.extra-build-dependencies]
problem-package = ["setuptools", "wheel"]
```

## 検証方法

プロジェクトが uv のベストプラクティスに従っているか確認：

```bash
# 1. 必須ファイルの存在確認
test -f pyproject.toml && echo "✓ pyproject.toml exists"
test -f uv.lock && echo "✓ uv.lock exists"
test -f .python-version && echo "✓ .python-version exists"

# 2. .gitignore の確認
grep -q ".venv/" .gitignore && echo "✓ .venv/ is ignored"

# 3. ロックファイルの整合性確認
uv lock --check && echo "✓ Lock file is up to date"

# 4. 環境の同期確認
uv sync --locked && echo "✓ Environment is synced"

# 5. テストの実行
uv run pytest && echo "✓ Tests pass"

# 6. リンターの実行
uv run ruff check && echo "✓ Linting passes"
```

## セキュリティと再現性

### 再現可能なインストール

```bash
# 特定日付以降のパッケージを除外（セキュリティ対策）
uv sync --exclude-newer 2024-01-01
```

```toml
[tool.uv]
# グローバルなクールダウン（推奨）
exclude-newer = "1 week"

# パッケージ固有のクールダウン
[tool.uv.exclude-newer-package]
requests = "30 days"
numpy = "14 days"
```

### セキュリティのベストプラクティス

- 新しいバージョンを即座に使用せず、クールダウン期間を設定
- CI/CD で必ず `--locked` を使用
- プライベート依存関係には適切な認証を設定
- 定期的に `pip-audit` などで脆弱性をスキャン

## まとめ

### 必須チェックリスト

✅ `.venv/` を `.gitignore` に追加  
✅ `uv.lock` をバージョン管理に含める  
✅ `.python-version` で Python バージョンを固定  
✅ すべての依存関係に下限バージョンを指定  
✅ CI/CD で `--locked` を使用  
✅ 本番環境では `--no-dev` を使用  
✅ Docker で適切なキャッシング戦略を使用  
✅ マネージドPythonを優先: `python-preference = "managed"`

### 参考リソース

- [公式ドキュメント](https://docs.astral.sh/uv/)
- [GitHub リポジトリ](https://github.com/astral-sh/uv)
- [setup-uv アクション](https://github.com/astral-sh/setup-uv)
