---
description: 'uvを用いたPythonプロジェクト管理のベストプラクティス'
applyTo: 'pyproject.toml, uv.lock, **/*.py'
---

# uv プロジェクト管理ベストプラクティス

## 目的と適用範囲

すべてのPythonプロジェクトにおいて、pip、pip-tools、poetryの代わりに、標準のPythonパッケージマネージャーとしてuv(v0.9.21以降)を使用する。

## ツールとバージョン

- **uv**: 0.9.21以降(AstralのRustベースのパッケージマネージャー)
- **Python**: 3.10以降
- **プラットフォーム**: クロスプラットフォーム(Linux、macOS、Windows)

## 基本原則

- 依存関係管理にはuvコマンドのみを使用する
- バージョン管理に`uv.lock`を含める
  **根拠**: すべての環境で再現可能なビルドを保証する
- `.gitignore`に`.venv/`を追加する
  **根拠**: 仮想環境は環境固有であり、共有すべきではない
- `.python-version`ファイルでPythonバージョンを固定する
  **根拠**: チームメンバー間で一貫したPythonバージョンを保証する
- CI/CDで`--locked`フラグを使用する
  **根拠**: 環境間の依存関係解決の違いを防止する

## プロジェクト構造

```
my-project/
├── .python-version          # Pythonバージョンを固定(必須)
├── .gitignore              # .venv/を除外
├── pyproject.toml          # プロジェクトメタデータと依存関係
├── uv.lock                 # ロックファイル(VCSに必須)
├── src/
│   └── my_project/
│       ├── __init__.py
│       └── main.py
└── tests/
    └── test_main.py
```

バージョン管理ルール:
- `.gitignore`に`.venv/`を追加する
- バージョン管理に`uv.lock`を含める
- バージョン管理に`.python-version`を含める

## 依存関係管理

### 依存関係の追加

```bash
# 本番依存関係
uv add "httpx>=0.24.0"

# 開発依存関係
uv add --group dev pytest ruff mypy

# オプション依存関係
uv add --optional ml numpy pandas

# グループ依存関係(推奨)
uv add --group lint ruff mypy
uv add --group test pytest pytest-cov
```

### pyproject.toml設定

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "明確な説明"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31.0",      # 下限を指定
    "pandas>=2.0.0,<3.0.0",  # メジャーバージョン制約
]

[dependency-groups]
lint = ["ruff>=0.1.0", "mypy>=1.0.0"]
test = ["pytest>=7.0.0", "pytest-cov>=4.0.0"]

[tool.uv]
python-preference = "managed"  # 管理されたPythonを優先
default-groups = ["lint", "test"]
```

### バージョン指定ルール

- すべての直接依存関係に下限を指定する
  **根拠**: 最小限の機能の可用性を保証する
- 互換性が明示的に制限されない限り、上限を避ける
  **根拠**: バグ修正とセキュリティパッチを受け取れるようにする
- `==`による固定を避ける(正確なバージョンはロックファイルに任せる)
  **根拠**: ロックファイルが再現性を提供しつつ、`pyproject.toml`では柔軟性を許容する

良い例:
```toml
dependencies = [
    "requests>=2.31.0",
    "pandas>=2.0.0,<3.0.0",
]
```

悪い例:
```toml
dependencies = [
    "requests",           # バージョン指定なし
    "pandas==2.1.3",      # 過度に制約
]
```

## Pythonバージョン管理

```bash
# 特定バージョンのインストール
uv python install 3.12

# プロジェクトのPythonバージョンを固定(必須)
uv python pin 3.12

# 複数バージョンのテスト
uv python install 3.10 3.11 3.12
```

pyproject.tomlでの設定:
```toml
[project]
requires-python = ">=3.10"  # 過度に広い範囲を避ける

[tool.uv]
python-preference = "managed"  # 管理されたPythonを使用
```

## 環境の同期とロックファイル

### 基本ワークフロー

```bash
# 依存関係の追加(自動的にロックを更新)
uv add package-name

# 環境の同期
uv sync

# CI/CD(必須)
uv sync --locked

# コマンドの実行
uv run python script.py
uv run pytest
```

### 同期オプション

```bash
# 開発: すべての依存関係
uv sync --all-extras

# 本番: 開発依存関係を除外
uv sync --no-dev --locked

# CI/CD: ロックの整合性を検証
uv sync --locked --frozen

# 特定のグループのみ
uv sync --group lint --group test
```

良い例(CI/CD):
```yaml
- name: Install dependencies
  run: uv sync --locked --no-dev
- name: Run tests
  run: uv run pytest
```

悪い例(CI/CD):
```yaml
# ロックファイルを無視
- run: uv sync

# 本番に開発依存関係を含める
- run: uv sync --all-extras
```

## CI/CD設定

### GitHub Actions

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Install uv
        uses: astral-sh/setup-uv@v7
        with:
          version: "0.9.21"
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

### マトリックステスト

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
      - uses: astral-sh/setup-uv@v7
        with:
          version: "0.9.21"
          enable-cache: true
      - run: uv python install ${{ matrix.python-version }}
      - run: uv sync --locked
      - run: uv run pytest
```

### キャッシング戦略

```yaml
- uses: actions/cache@v4
  with:
    path: /tmp/.uv-cache
    key: uv-${{ runner.os }}-${{ hashFiles('uv.lock') }}
    restore-keys: uv-${{ runner.os }}-

- run: uv sync --locked
  env:
    UV_CACHE_DIR: /tmp/.uv-cache

- run: uv cache prune --ci
```

## Docker設定

### 本番用Dockerfile

```dockerfile
FROM python:3.12-slim-bookworm

# uvのインストール(バージョンを固定)
COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

WORKDIR /app

# 依存関係のインストール(レイヤーキャッシュの最適化)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev

COPY . /app

RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

ENV PATH="/app/.venv/bin:$PATH"

CMD ["python", "-m", "myapp"]
```

### 最適化されたマルチステージビルド

```dockerfile
FROM python:3.12-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

WORKDIR /app

# バイトコードコンパイルを有効化
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev --no-editable

COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-editable

FROM python:3.12-slim

COPY --from=builder --chown=app:app /app /app

WORKDIR /app
ENV PATH="/app/.venv/bin:$PATH"

CMD ["python", "-m", "myapp"]
```

Dockerベストプラクティス:
- uvのバージョンを明示的に固定する
- レイヤーキャッシングを使用する(依存関係とコードを分離)
  **根拠**: リビルド時間とイメージサイズを削減する
- 本番では`--no-dev`を使用する
- バイトコードコンパイルを有効化する: `UV_COMPILE_BYTECODE=1`
- 編集可能でないインストールを使用する: `--no-editable`
- イメージサイズを最小化するためにマルチステージビルドを使用する

## ワークスペースとモノレポ

ルートの`pyproject.toml`で設定:
```toml
[tool.uv.workspace]
members = ["packages/*", "apps/*"]

[tool.uv.sources]
common-lib = { workspace = true }
```

使用対象: 密結合されたパッケージ、モノレポ、共有依存関係
使用を避けるべき場合: 競合する依存関係、異なるPythonバージョン

## 一般的なワークフロー

### 新規プロジェクト
```bash
uv init my-project && cd my-project
uv python pin 3.12
uv add requests pandas
uv add --group dev pytest ruff mypy
git init && echo ".venv/" >> .gitignore
git add . && git commit -m "Initial commit"
```

### pip/poetryからの移行
```bash
uv add --requirements requirements.txt  # requirements.txtから
uv sync                                  # poetryから
uv lock
git add pyproject.toml uv.lock .python-version
```

### 依存関係のアップグレード
```bash
uv lock --upgrade                # すべての依存関係
uv lock --upgrade-package pkg    # 特定のパッケージ
uv sync && uv run pytest         # テスト
```

### 複数バージョンのテスト
```bash
uv sync --resolution lowest && uv run pytest
uv sync --resolution highest && uv run pytest
uv sync  # 復元
```

## アンチパターン

### バージョン管理のエラー
悪い例: `git add .venv/` または `echo "uv.lock" >> .gitignore`
良い例: `echo ".venv/" >> .gitignore && git add uv.lock .python-version`

### CI/CDのエラー
悪い例: `uv sync`
良い例: `uv sync --locked`

### バージョン指定のエラー
悪い例: `dependencies = ["requests", "pandas"]`
良い例: `dependencies = ["requests>=2.31.0", "pandas>=2.0.0,<3.0.0"]`

### 本番設定のエラー
悪い例: `RUN uv sync --all-extras`
良い例: `RUN uv sync --locked --no-dev`

### パッケージマネージャーの混在
悪い例: `uv add requests && pip install pandas`
良い例: `uv add requests pandas`

## トラブルシューティング

```bash
# ロックファイルが古い
uv lock && uv sync

# 依存関係の競合
uv sync --verbose
uv sync --resolution lowest
uv lock --upgrade-package problematic-package

# キャッシュの問題
uv cache clean
uv sync --no-cache
uv sync --reinstall-package numpy

# ビルド失敗(pyproject.tomlに追加)
[tool.uv]
no-build-isolation-package = ["problem-package"]
```

## 検証

```bash
test -f pyproject.toml && test -f uv.lock && test -f .python-version && \
grep -q ".venv/" .gitignore && \
uv lock --check && \
uv sync --locked && \
uv run pytest && \
echo "✓ すべてのチェックに合格"
```

## セキュリティ

```toml
[tool.uv]
exclude-newer = "1 week"  # 新しいパッケージのクールダウン期間

[tool.uv.exclude-newer-package]
requests = "30 days"
```

ベストプラクティス:
- 新しいバージョンにクールダウン期間を設定する(脆弱性を発見する時間を確保)
- CI/CDで`--locked`を使用する
- `pip-audit`で脆弱性を定期的にスキャンする

## 必須チェックリスト

- [ ] `.gitignore`に`.venv/`を記載
- [ ] バージョン管理に`uv.lock`を含める
- [ ] `.python-version`を固定
- [ ] すべての依存関係に下限を設定
- [ ] CI/CDで`--locked`を使用
- [ ] 本番では`--no-dev`を使用
- [ ] pyproject.tomlで`python-preference = "managed"`を設定

## 参考文献

- [公式ドキュメント](https://docs.astral.sh/uv/)
- [GitHub](https://github.com/astral-sh/uv)
- [setup-uv Action](https://github.com/astral-sh/setup-uv)
