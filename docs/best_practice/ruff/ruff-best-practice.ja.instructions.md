---
description: 'Ruff を使用した Python コード品質管理のベストプラクティスとガイドライン'
applyTo: '**/*.py, **/pyproject.toml, **/ruff.toml, **/.ruff.toml'
---

# Ruff コード品質管理

Ruff は Rust で実装された高性能な Python linter および formatter である。本ファイルでは Ruff を使用したコード品質管理のベストプラクティスを説明する。

## 適用範囲とツール概要

- Python 3.10+ プロジェクトで Ruff を統合 linter および formatter として使用する
- Black、Flake8、isort、pydocstyle、pyupgrade を Ruff で置き換える
- Ruff の 10〜100 倍の速度向上と 800 以上のルールを活用する

## 基本設定

### インストール

- `uv tool install ruff@latest` および `uv add --dev ruff` を使用して Ruff をインストールする
- uv が利用できない場合は `pip install ruff` を使用する

```bash
uv tool install ruff@latest
uv add --dev ruff
```

### 基本実行コマンド

- `ruff check --fix .` を使用して自動修正付きで linter を実行する
- `ruff format .` を使用して formatter を実行する
- 両方を順番に実行する: `ruff check --fix . && ruff format .`

```bash
ruff check --fix . && ruff format .
```

## Linter のベストプラクティス

### ルール選択

- 最小限のルールから始める: `["E4", "E7", "E9", "F"]`
- 推奨されるバランスの取れたセット: `["E", "F", "UP", "B", "SIM", "I", "N", "ASYNC"]`
- 過度な変更を避けるためにルールを段階的に追加する

```toml
# 推奨設定
[tool.ruff.lint]
select = ["E", "F", "UP", "B", "SIM", "I", "N", "ASYNC"]
```

### 新しいルールの追加

- ルールを段階的に追加するには `select` の代わりに `extend-select` を使用する

```toml
[tool.ruff.lint]
extend-select = ["B", "UP", "I"]
```

### ファイルごとの例外設定

- `[tool.ruff.lint.per-file-ignores]` を使用してファイルごとの無視設定を構成する
- `__init__.py` で `F401` を使用して未使用のインポートを許可する
- テストファイルで `S101` を使用して `assert` を許可する

```toml
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
"tests/**/*.py" = ["S101"]
```

### エラー抑制

- `# noqa: RULE` を使用して行レベルのエラーを抑制する
- カンマ区切りリストで複数のルールを抑制する: `# noqa: E741, F841`

```python
import os  # noqa: F401
x = 1  # noqa: E741, F841
```

### 未使用の noqa コメント検出

- `ruff check --extend-select RUF100 --fix` を使用して未使用の `noqa` コメントを検出・削除する

### 修正の安全性

- デフォルトで `ruff check --fix` を使用して安全な修正のみを適用する
- `--unsafe-fixes` フラグを使用して安全でない修正を含める
- `[tool.ruff] unsafe-fixes = true` を使用してグローバルに制御する

```toml
[tool.ruff]
unsafe-fixes = true
```

## Formatter のベストプラクティス

### Black 互換設定

- Black 互換性のために `line-length = 88` および `quote-style = "double"` を設定する
- Black の動作に合わせるために `skip-magic-trailing-comma = false` を使用する

```toml
[tool.ruff]
line-length = 88

[tool.ruff.format]
quote-style = "double"
```

### Docstring のフォーマット

- `docstring-code-format = true` で docstring のコードフォーマットを有効にする
- 自動的な行長設定のために `docstring-code-line-length = "dynamic"` を使用する

```toml
[tool.ruff.format]
docstring-code-format = true
docstring-code-line-length = "dynamic"
```

### フォーマット抑制

- ブロックレベルの抑制には `# fmt: off` と `# fmt: on` を使用する
- 単一行の抑制には `# fmt: skip` を使用する

### Formatter と Linter の競合回避

- formatter と競合するルールを無視する: `W191`、`E111`、`E114`、`E117`、`D206`、`D300`、`COM812`、`COM819`、`ISC002`

```toml
[tool.ruff.lint]
ignore = ["W191", "E111", "E114", "E117", "D206", "D300", "COM812", "COM819", "ISC002"]
```

## 設定ファイル構造

### 推奨される最小設定（pyproject.toml）

- `pyproject.toml` に最小限の必要な設定で Ruff を構成する
- Python バージョン、行長、コアルールセットを指定する
- 無視するディレクトリには `extend-exclude` を使用する

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

- スタンドアロン設定には `ruff.toml` または `.ruff.toml` を使用する（`[tool.ruff]` ヘッダーを省略）

```toml
line-length = 88

[lint]
select = ["E", "F"]

[format]
quote-style = "double"
```

### 設定の継承

- `extend = "../pyproject.toml"` を使用して親設定を拡張する
- 子設定で特定の設定を上書きする

```toml
[tool.ruff]
extend = "../pyproject.toml"
line-length = 100  # 親設定を上書き
```

## Docstring と型ヒント

### Docstring スタイルの強制

- `select = ["D"]` で docstring ルールを有効にする
- 規約を指定する: `google`、`numpy`、または `pep257`

```toml
[tool.ruff.lint]
select = ["D"]

[tool.ruff.lint.pydocstyle]
convention = "google"
```

### 型ヒント関連ルール

- `select = ["ANN", "TCH"]` で型ヒントルールを有効にする
- `future-annotations = true` を使用して `from __future__ import annotations` を有効にする

```toml
[tool.ruff.lint]
select = ["ANN", "TCH"]
future-annotations = true
```

## CI/CD 統合

### GitHub Actions

- 基本セットアップには `astral-sh/ruff-action@v3` を使用する
- 適切なアノテーションのために `--output-format=github` を使用する
- `ruff format --check` で formatter チェックを実行する

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

### pre-commit

- `ruff-check` を `ruff-format` の前に配置する（順序が重要）
- 自動修正のために `args: [--fix]` を使用する

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.10
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
```

## エディタ統合

### VS Code

- `editor.defaultFormatter` を `charliermarsh.ruff` に設定する
- `editor.formatOnSave` および保存時の `source.fixAll` を有効にする
- `ruff.lint.run` を `onSave` に設定する

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll": "explicit"
    }
  }
}
```

### PyCharm / IntelliJ

- Settings → Tools → Ruff で Ruff を有効にし、実行可能ファイルのパスを設定する

## 移行ガイド

### Black からの移行

- pyproject.toml から `[tool.black]` セクションを削除する
- `line-length = 88` および `quote-style = "double"` で Ruff format 設定を追加する

```toml
[tool.ruff]
line-length = 88

[tool.ruff.format]
quote-style = "double"
```

### Flake8 からの移行

- `max-line-length` を `[tool.ruff] line-length` に転送する
- 無視されたコードを `[tool.ruff.lint] ignore` 配列に変換する
- プラグインをマッピングする: flake8-bugbear→B、flake8-simplify→SIM、pep8-naming→N、flake8-comprehensions→C4

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
ignore = ["E203", "W503"]
```

### isort からの移行

- `[tool.isort]` セクションを削除する
- `select = ["I"]` でインポートのソートを有効にする
- `[tool.ruff.lint.isort]` 下で `known-first-party` を設定する

```toml
[tool.ruff.lint]
select = ["I"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]
```

## トラブルシューティング

### 行長違反が残る

- formatter はベストエフォートの行長を適用するため `E501` を無視する

```toml
[tool.ruff.lint]
ignore = ["E501"]
```

### 設定が適用されない

- `ruff check --show-settings path/to/file.py` で設定を確認する
- `ruff clean` を使用してキャッシュをクリアする

### Jupyter Notebook サポート

- per-file-ignores を使用して notebook 固有のルールを無視する

```toml
[tool.ruff.lint.per-file-ignores]
"*.ipynb" = ["E402", "T20"]
```

## 設定の検証

- `ruff check --show-settings path/to/file.py` でアクティブな設定を確認する
- `ruff check --show-files` で設定ファイルをリストする
- `ruff clean` を使用してキャッシュをクリアする

## 段階的導入計画

- **第 1 週**: Ruff をインストールし、デフォルトで実行し、自動修正可能なエラーを修正する
- **第 2 週**: 追加のルールセットを有効にし、CI/CD を統合し、pre-commit を設定する
- **第 3 週**: formatter を有効にし、エディタ統合を設定する
- **第 4 週**: 設定を微調整し、古い Flake8/Black 設定を削除する

## セットアップチェックリスト

- プロジェクトに Ruff をインストールする
- `pyproject.toml` に基本設定を追加する
- CI/CD パイプラインに統合する
- エディタに Ruff 拡張機能をインストールする
- pre-commit フックを設定する
- チームメンバーに通知する

## 参考リンク

- [Ruff ドキュメント](https://docs.astral.sh/ruff/)
- [ルールリファレンス](https://docs.astral.sh/ruff/rules/)
