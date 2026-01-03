---
description: 'Ruff を使用した Python コード品質管理のベストプラクティスとガイドライン'
applyTo: '**/*.py, **/pyproject.toml, **/ruff.toml, **/.ruff.toml'
---

# Ruff コード品質管理

Ruff は Rust で実装された高速な Python linter および formatter です。本ファイルでは Ruff を使用したコード品質管理のベストプラクティスを説明します。

## プロジェクトコンテキスト

- **対象**: Python 3.10 以降のプロジェクト
- **ツール**: Ruff（linter + formatter）
- **置き換え可能**: Black, Flake8, isort, pydocstyle, pyupgrade
- **特徴**: 10-100 倍高速、800 以上のルール、自動修正対応

## 基本設定

### インストール

```bash
# uv を使用（推奨）
uv tool install ruff@latest
uv add --dev ruff

# または pip
pip install ruff
```

### 基本実行コマンド

```bash
# Linter を実行（自動修正付き）
ruff check --fix .

# Formatter を実行
ruff format .

# 両方を実行（推奨順序）
ruff check --fix . && ruff format .
```

## Linter のベストプラクティス

### ルール選択

段階的にルールを追加することを推奨：

```toml
# 最小限の設定
[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]

# 推奨設定（バランス型）
[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "F",      # Pyflakes
    "UP",     # pyupgrade
    "B",      # flake8-bugbear
    "SIM",    # flake8-simplify
    "I",      # isort
    "N",      # pep8-naming
    "ASYNC",  # flake8-async
]

# 厳格な設定
[tool.ruff.lint]
select = [
    "E", "W", "F", "UP", "B", "SIM", "I", "N", "ASYNC",
    "C4", "DTZ", "T10", "EM", "ISC", "ICN", "PIE", "PT",
    "Q", "RSE", "RET", "SLF", "TCH", "ARG", "PTH", "ERA",
    "PL", "TRY", "RUF",
]
ignore = ["PLR0913", "TRY003"]
```

### 新しいルールの追加

`select` より `extend-select` を使用：

```toml
[tool.ruff.lint]
extend-select = ["B", "UP", "I"]
```

### ファイルごとの例外設定

```toml
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]              # 未使用 import を許可
"tests/**/*.py" = ["S101"]            # assert を許可
"**/{tests,docs,tools}/*" = ["E402"]  # import 順序を許可
```

### エラー抑制

```python
# 行レベルの抑制
import os  # noqa: F401

# 複数のルールを無視
x = 1  # noqa: E741, F841

# ブロックレベルの抑制（preview モード）
# ruff: disable[E501]
VALUE = "very long string..."
# ruff: enable[E501]
```

### 未使用の noqa 検出

```bash
# 未使用の noqa を検出・削除
ruff check --extend-select RUF100 --fix
```

### Fix の安全性

```bash
# Safe fix のみ（デフォルト）
ruff check --fix

# Unsafe fix も適用
ruff check --fix --unsafe-fixes
```

設定での制御：

```toml
[tool.ruff]
unsafe-fixes = true  # プロジェクト全体で有効化

[tool.ruff.lint]
extend-safe-fixes = ["F601"]      # safe に昇格
extend-unsafe-fixes = ["UP034"]   # unsafe に降格
```

## Formatter のベストプラクティス

### Black 互換設定

```toml
[tool.ruff]
line-length = 88
indent-width = 4

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "auto"
```

### Docstring のフォーマット

```toml
[tool.ruff.format]
# Docstring 内のコード例を自動フォーマット
docstring-code-format = true

# Docstring 内のコードの行長制限
docstring-code-line-length = "dynamic"
```

### フォーマット抑制

```python
# fmt: off
not_formatted = [1,2,3,4]
# fmt: on

# 単一の文をスキップ
not_formatted = [1,2,3,4]  # fmt: skip
```

### Formatter と Linter の競合回避

以下のルールは formatter と競合するため無効化を推奨：

```toml
[tool.ruff.lint]
ignore = [
    "W191",   # tab-indentation
    "E111",   # indentation-with-invalid-multiple
    "E114",   # indentation-with-invalid-multiple-comment
    "E117",   # over-indented
    "D206",   # docstring-tab-indentation
    "D300",   # triple-single-quotes
    "Q000",   # bad-quotes-inline-string
    "Q001",   # bad-quotes-multiline-string
    "Q002",   # bad-quotes-docstring
    "Q003",   # avoidable-escaped-quote
    "COM812", # missing-trailing-comma
    "COM819", # prohibited-trailing-comma
    "ISC002", # multi-line-implicit-string-concatenation
]
```

## 設定ファイル構造

### 推奨される最小設定（pyproject.toml）

```toml
[project]
requires-python = ">=3.10"

[tool.ruff]
line-length = 88
target-version = "py310"
required-version = ">=0.14.0"

# ファイル検索の設定
extend-exclude = [".venv", "build", "dist"]
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "B", "UP", "I"]
ignore = ["E501"]

fixable = ["ALL"]
unfixable = ["B"]

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
"tests/**/*.py" = ["S101"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.format]
quote-style = "double"
docstring-code-format = true
```

### ruff.toml での設定

`ruff.toml` または `.ruff.toml` を使用する場合、`[tool.ruff]` ヘッダーは不要：

```toml
line-length = 88

[lint]
select = ["E", "F"]

[format]
quote-style = "double"
```

### 設定の継承

```toml
[tool.ruff]
extend = "../pyproject.toml"
line-length = 100  # 親設定を上書き
```

## Docstring とタイプヒント

### Docstring スタイルの強制

```toml
[tool.ruff.lint]
select = ["D"]

[tool.ruff.lint.pydocstyle]
convention = "google"  # または "numpy", "pep257"
```

### タイプヒント関連ルール

```toml
[tool.ruff.lint]
select = [
    "ANN",  # flake8-annotations
    "TCH",  # flake8-type-checking
]

# future annotations を自動追加
future-annotations = true
```

## CI/CD 統合

### GitHub Actions

基本設定：

```yaml
name: Ruff
on: [push, pull_request]

jobs:
  ruff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/ruff-action@v3
```

詳細な設定：

```yaml
name: CI
on: push

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      
      - name: Install Ruff
        run: pip install ruff
      
      - name: Run Ruff Linter
        run: ruff check --output-format=github .
      
      - name: Run Ruff Formatter
        run: ruff format --check .
```

### GitLab CI/CD

```yaml
.base_ruff:
  stage: build
  image: ghcr.io/astral-sh/ruff:latest-alpine
  before_script:
    - ruff --version

Ruff Check:
  extends: .base_ruff
  script:
    - ruff check --output-format=gitlab --output-file=code-quality-report.json
  artifacts:
    reports:
      codequality: $CI_PROJECT_DIR/code-quality-report.json

Ruff Format:
  extends: .base_ruff
  script:
    - ruff format --diff
```

### pre-commit

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.10
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
```

**重要**: `ruff-check` は `ruff-format` の前に配置する必要があります。

## エディタ統合

### VS Code

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll": "explicit",
      "source.organizeImports": "explicit"
    }
  },
  "ruff.lint.run": "onSave"
}
```

### PyCharm / IntelliJ

1. Settings → Tools → Ruff
2. Enable Ruff
3. Ruff executable のパスを設定

## 移行ガイド

### Black からの移行

```toml
# Black の [tool.black] セクションを削除

# Ruff の設定を追加
[tool.ruff]
line-length = 88

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### Flake8 からの移行

```toml
# Flake8 の .flake8 設定
# [flake8]
# max-line-length = 88
# ignore = E203, W503

# Ruff の設定
[tool.ruff]
line-length = 88

[tool.ruff.lint]
ignore = ["E203", "W503"]
```

プラグインのマッピング：

| Flake8 プラグイン | Ruff ルール |
|------------------|------------|
| flake8-bugbear   | B          |
| flake8-simplify  | SIM        |
| pep8-naming      | N          |
| flake8-comprehensions | C4    |

### isort からの移行

```toml
# isort の [tool.isort] セクションを削除

# Ruff の設定
[tool.ruff.lint]
select = ["I"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]
```

## トラブルシューティング

### 行長違反が残る

Formatter は行長のベストエフォートのため、`E501` を無視：

```toml
[tool.ruff.lint]
ignore = ["E501"]
```

### 設定が反映されない

```bash
# 設定ファイルの位置を確認
ruff check --show-settings path/to/file.py

# キャッシュをクリア
ruff clean
```

### Jupyter Notebook のサポート

```toml
# Notebook で特定のルールを無視
[tool.ruff.lint.per-file-ignores]
"*.ipynb" = ["E402", "T20"]
```

### Preview モードのルールが不安定

```toml
[tool.ruff]
# 本番環境では preview モードを無効化
preview = false
```

## 設定の検証

```bash
# 使用されている設定を確認
ruff check --show-settings path/to/file.py

# 設定ファイルのパスを確認
ruff check --show-files

# キャッシュをクリア
ruff clean
```

## 段階的導入プラン

**Week 1**: 
- Ruff をインストール
- デフォルト設定で実行
- 自動修正可能なエラーを修正

**Week 2**:
- 追加のルールセットを有効化
- CI/CD に統合
- pre-commit フックを設定

**Week 3**:
- Formatter を有効化
- エディタ統合を設定

**Week 4**:
- 設定の微調整
- 既存の Flake8/Black 設定を削除

## チェックリスト

- [ ] プロジェクトに Ruff をインストール
- [ ] `pyproject.toml` に基本設定を追加
- [ ] CI/CD パイプラインに統合
- [ ] エディタに Ruff 拡張機能をインストール
- [ ] pre-commit フックを設定
- [ ] チームメンバーに変更を通知

## 参考リンク

- [Ruff 公式ドキュメント](https://docs.astral.sh/ruff/)
- [ルールリファレンス](https://docs.astral.sh/ruff/rules/)
- [設定オプション](https://docs.astral.sh/ruff/settings/)
- [GitHub リポジトリ](https://github.com/astral-sh/ruff)
