# Ruff コード品質管理ベストプラクティス

Ruff を使用したコード品質管理のベストプラクティスをまとめたドキュメントです。

## 目次

1. [Ruff とは](#ruff-とは)
2. [インストールと基本設定](#インストールと基本設定)
3. [Linter のベストプラクティス](#linter-のベストプラクティス)
4. [Formatter のベストプラクティス](#formatter-のベストプラクティス)
5. [設定ファイルの構造](#設定ファイルの構造)
6. [ルール選択の推奨事項](#ルール選択の推奨事項)
7. [CI/CD 統合](#cicd-統合)
8. [エディタ統合](#エディタ統合)
9. [移行ガイド](#移行ガイド)
10. [トラブルシューティング](#トラブルシューティング)

---

## Ruff とは

Ruff は Rust で実装された非常に高速な Python linter および formatter です。主な特徴：

- **高速性**: Flake8 や Black より 10-100 倍高速
- **統合性**: Linter と Formatter が統合された単一ツール
- **互換性**: Black、Flake8、isort などのツールと互換性あり
- **800 以上のルール**: 50 以上のプラグインを内蔵

### 置き換え可能なツール

Ruff は以下のツールを置き換えることができます：

- Flake8 とその多数のプラグイン
- Black（フォーマッター）
- isort（import ソート）
- pydocstyle（docstring チェック）
- pyupgrade（Python バージョンアップグレード）
- autoflake（未使用コード削除）

---

## インストールと基本設定

### インストール方法

```bash
# uv を使用（推奨）
uv tool install ruff@latest

# または uv でプロジェクトに追加
uv add --dev ruff

# pip を使用
pip install ruff

# pipx を使用
pipx install ruff
```

### 基本的な実行方法

```bash
# Linter を実行
ruff check .

# 自動修正を適用
ruff check --fix .

# Formatter を実行
ruff format .

# フォーマットのチェックのみ（変更なし）
ruff format --check .
```

---

## Linter のベストプラクティス

### 1. ルール選択の段階的アプローチ

初めて Ruff を導入する際は、デフォルトのルールセットから始めることを推奨します：

```toml
[tool.ruff.lint]
# デフォルトでは E4, E7, E9, F が有効
select = ["E4", "E7", "E9", "F"]
```

段階的にルールを追加：

```toml
[tool.ruff.lint]
select = [
    # Pyflakes
    "F",
    # pycodestyle
    "E",
    "W",
    # pyupgrade
    "UP",
    # flake8-bugbear
    "B",
    # flake8-simplify
    "SIM",
    # isort
    "I",
]
```

### 2. select より extend-select を優先

新しいルールを追加する場合は `extend-select` を使用します：

```toml
[tool.ruff.lint]
extend-select = [
    "B",   # flake8-bugbear
    "UP",  # pyupgrade
]
```

### 3. fix の安全性の理解

Ruff は fix を "safe" と "unsafe" に分類します：

```bash
# Safe fix のみ適用（デフォルト）
ruff check --fix

# Unsafe fix も含めて適用
ruff check --fix --unsafe-fixes
```

設定ファイルでの制御：

```toml
[tool.ruff]
# プロジェクト全体で unsafe fix を有効化
unsafe-fixes = true

[tool.ruff.lint]
# 特定のルールを safe に昇格
extend-safe-fixes = ["F601"]

# 特定のルールを unsafe に降格
extend-unsafe-fixes = ["UP034"]
```

### 4. エラー抑制の適切な使用

#### 行レベルの抑制

```python
import os  # noqa: F401

# 複数のルールを無視
x = 1  # noqa: E741, F841
```

#### ファイルレベルの抑制

```python
# ruff: noqa: F841
# または
# ruff: noqa
```

#### ブロックレベルの抑制（preview モード）

```python
# ruff: disable[E501]
VALUE = "very long string..."
# ruff: enable[E501]
```

### 5. per-file-ignores の活用

特定のファイルパターンに対してルールを無視：

```toml
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]  # __init__.py で未使用 import を許可
"tests/**/*.py" = ["S101"]  # テストファイルで assert を許可
"**/{tests,docs,tools}/*" = ["E402"]
```

### 6. 未使用の noqa の検出

```bash
# 未使用の noqa コメントを検出
ruff check --extend-select RUF100

# 未使用の noqa を自動削除
ruff check --extend-select RUF100 --fix
```

### 7. 新しいルールの段階的導入

```bash
# 既存の違反に noqa を追加
ruff check --select B --add-noqa
```

### 8. 設定の検証

```bash
# 使用されている設定を確認
ruff check --show-settings path/to/file.py

# 設定ファイルのパスを確認
ruff check --show-files

# キャッシュをクリア
ruff clean
```

---

## Formatter のベストプラクティス

### 1. Black 互換性の確保

Ruff の formatter は Black と互換性があります（> 99.9% の一致率）：

```toml
[tool.ruff]
# Black と同じ設定
line-length = 88
indent-width = 4

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "auto"
```

### 2. クォートスタイルの選択

```toml
[tool.ruff.format]
# ダブルクォート（デフォルト、Black 互換）
quote-style = "double"

# シングルクォート
quote-style = "single"

# 既存のクォートを保持
quote-style = "preserve"
```

### 3. Docstring のコードフォーマット

```toml
[tool.ruff.format]
# Docstring 内のコード例を自動フォーマット
docstring-code-format = true

# Docstring 内のコードの行長制限
docstring-code-line-length = "dynamic"  # または固定値（例: 60）
```

### 4. フォーマット抑制

```python
# fmt: off
not_formatted = [1,2,3,4]
# fmt: on

# または単一の文をスキップ
formatted = [1, 2, 3]
not_formatted = [1,2,3,4]  # fmt: skip
```

### 5. Linter との競合回避

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

---

## 設定ファイルの構造

### pyproject.toml での設定

```toml
[project]
# Python バージョンの指定（重要）
requires-python = ">=3.10"

[tool.ruff]
# トップレベル設定
line-length = 88
indent-width = 4
target-version = "py310"

# ファイル検索の設定
extend-exclude = [
    ".venv",
    "build",
    "dist",
]

# ソースディレクトリの指定
src = ["src", "tests"]

[tool.ruff.lint]
# Linter の設定
select = ["E", "F", "B", "UP", "I"]
ignore = ["E501"]

# Fix の設定
fixable = ["ALL"]
unfixable = ["B"]

# ダミー変数のパターン
dummy-variable-rgx = "^(_+|(_+[a-zA-Z0-9_]*[a-zA-Z0-9]+?))$"

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
"tests/**/*.py" = ["S101"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.format]
# Formatter の設定
quote-style = "double"
indent-style = "space"
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
# 親ディレクトリの設定を継承
extend = "../pyproject.toml"

# ただし行長は上書き
line-length = 100
```

---

## ルール選択の推奨事項

### 基本セット（最小限）

```toml
[tool.ruff.lint]
select = [
    "E4",  # Import errors
    "E7",  # Statement errors
    "E9",  # Runtime errors
    "F",   # Pyflakes
]
```

### 推奨セット（バランス型）

```toml
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
```

### 厳格セット（高品質維持）

```toml
[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "UP",     # pyupgrade
    "B",      # flake8-bugbear
    "SIM",    # flake8-simplify
    "I",      # isort
    "N",      # pep8-naming
    "ASYNC",  # flake8-async
    "C4",     # flake8-comprehensions
    "DTZ",    # flake8-datetimez
    "T10",    # flake8-debugger
    "EM",     # flake8-errmsg
    "ISC",    # flake8-implicit-str-concat
    "ICN",    # flake8-import-conventions
    "PIE",    # flake8-pie
    "PT",     # flake8-pytest-style
    "Q",      # flake8-quotes
    "RSE",    # flake8-raise
    "RET",    # flake8-return
    "SLF",    # flake8-self
    "TCH",    # flake8-type-checking
    "ARG",    # flake8-unused-arguments
    "PTH",    # flake8-use-pathlib
    "ERA",    # eradicate
    "PL",     # Pylint
    "TRY",    # tryceratops
    "RUF",    # Ruff-specific rules
]

ignore = [
    "PLR0913",  # Too many arguments
    "TRY003",   # Avoid specifying long messages outside exception class
]
```

### Docstring スタイルの強制

```toml
[tool.ruff.lint]
select = ["D"]

[tool.ruff.lint.pydocstyle]
# Google, NumPy, PEP 257 から選択
convention = "google"
```

### タイプヒント関連

```toml
[tool.ruff.lint]
select = [
    "ANN",  # flake8-annotations
    "TCH",  # flake8-type-checking
]

[tool.ruff.lint]
# future annotations を自動追加
future-annotations = true
```

---

## CI/CD 統合

### GitHub Actions

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

より詳細な設定：

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

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install ruff

      # GitHub のインラインアノテーションを有効化
      - name: Run Ruff Linter
        run: ruff check --output-format=github .

      - name: Run Ruff Formatter
        run: ruff format --check .
```

### GitLab CI/CD

```yaml
.base_ruff:
  stage: build
  interruptible: true
  image:
    name: ghcr.io/astral-sh/ruff:latest-alpine
  before_script:
    - cd $CI_PROJECT_DIR
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
      # Linter
      - id: ruff-check
        args: [--fix]
      # Formatter
      - id: ruff-format
```

重要: `--fix` を使用する場合、`ruff-check` は `ruff-format` の前に配置する必要があります。

---

## エディタ統合

### VS Code

1. Ruff 拡張機能をインストール：`charliermarsh.ruff`

2. `.vscode/settings.json` で設定：

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

### Vim / Neovim

#### ALE

```vim
let g:ale_linters = {'python': ['ruff']}
let g:ale_fixers = {'python': ['ruff', 'ruff_format']}
let g:ale_fix_on_save = 1
```

#### Language Server Protocol (LSP)

```lua
require('lspconfig').ruff_lsp.setup {}
```

---

## 移行ガイド

### Black からの移行

Ruff の formatter は Black との互換性が > 99.9% あるため、スムーズに移行可能です：

```bash
# Black の設定を削除
# pyproject.toml から [tool.black] セクションを削除

# Ruff の設定を追加
[tool.ruff]
line-length = 88  # Black のデフォルト

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### Flake8 からの移行

1. Flake8 の設定を Ruff に変換：

```toml
# Flake8 の .flake8 または setup.cfg
[flake8]
max-line-length = 88
ignore = E203, W503
per-file-ignores =
    __init__.py:F401

# Ruff の pyproject.toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
ignore = ["E203", "W503"]

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
```

2. Flake8 プラグインを Ruff のルールにマッピング：

- `flake8-bugbear` → `B`
- `flake8-comprehensions` → `C4`
- `flake8-simplify` → `SIM`
- `pep8-naming` → `N`

### isort からの移行

```toml
# isort の設定
[tool.isort]
profile = "black"
known_first_party = ["myproject"]

# Ruff の設定
[tool.ruff.lint]
select = ["I"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]
```

---

## トラブルシューティング

### 問題 1: formatter と linter が競合する

**解決策**: 競合するルールを無視：

```toml
[tool.ruff.lint]
ignore = [
    "W191", "E111", "E114", "E117",
    "D206", "D300",
    "Q000", "Q001", "Q002", "Q003",
    "COM812", "COM819",
    "ISC002",
]
```

### 問題 2: 行長違反が残る

**原因**: Formatter は行長のハードリミットではなくベストエフォート。

**解決策**: `E501` を無視するか、コードを手動で調整：

```toml
[tool.ruff.lint]
ignore = ["E501"]
```

### 問題 3: import の順序が isort と異なる

**解決策**: `src` パスを正しく設定：

```toml
[tool.ruff]
src = ["src", "tests"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]
```

### 問題 4: Jupyter Notebook のサポート

Ruff は Jupyter Notebook をデフォルトでサポートします：

```toml
# Notebook をフォーマットから除外
[tool.ruff.format]
exclude = ["*.ipynb"]

# Notebook を linting から除外
[tool.ruff.lint]
exclude = ["*.ipynb"]

# 特定のルールのみ Notebook で無視
[tool.ruff.lint.per-file-ignores]
"*.ipynb" = ["E402", "T20"]
```

### 問題 5: 設定が反映されない

**確認事項**:

1. 設定ファイルの位置を確認：
```bash
ruff check /path/to/file.py --show-settings
```

2. 設定ファイルの構文エラーをチェック：
```bash
ruff check --config /path/to/ruff.toml
```

3. キャッシュをクリア：
```bash
ruff clean
```

### 問題 6: Preview モードのルールが不安定

**解決策**: 本番環境では preview モードを避ける：

```toml
[tool.ruff]
# preview モードを無効化（デフォルト）
preview = false

[tool.ruff.lint]
# 特定の preview ルールを明示的に有効化
explicit-preview-rules = true
```

---

## まとめ

### 推奨される最小設定

```toml
[project]
requires-python = ">=3.10"

[tool.ruff]
line-length = 88
target-version = "py310"

[tool.ruff.lint]
select = ["E", "F", "B", "UP", "I"]
ignore = ["E501"]

[tool.ruff.format]
quote-style = "double"
docstring-code-format = true
```

### 推奨される実行コマンド

```bash
# Linting + 自動修正
ruff check --fix .

# Formatting
ruff format .

# 両方を一度に実行（推奨順序）
ruff check --fix . && ruff format .
```

**注意**: Ruff のバージョン管理のため、`pyproject.toml` で required-version を指定することを推奨：

```toml
[tool.ruff]
required-version = ">=0.14.0"
```

### 実行チェックリスト

1. ✓ プロジェクトに Ruff をインストール
2. ✓ `pyproject.toml` に基本設定を追加
3. ✓ CI/CD パイプラインに統合
4. ✓ エディタに Ruff 拡張機能をインストール
5. ✓ pre-commit フックを設定
6. ✓ チームメンバーに変更を通知

### 段階的導入プラン

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
- ドキュメントを更新

**Week 4**:
- チーム全体でレビュー
- 設定の微調整
- 既存の Flake8/Black 設定を削除

---

## 参考リンク

- [Ruff 公式ドキュメント](https://docs.astral.sh/ruff/)
- [ルールリファレンス](https://docs.astral.sh/ruff/rules/)
- [設定オプション](https://docs.astral.sh/ruff/settings/)
- [GitHub リポジトリ](https://github.com/astral-sh/ruff)
- [FAQ](https://docs.astral.sh/ruff/faq/)
