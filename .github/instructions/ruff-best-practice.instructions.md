---
description: 'Best practices and guidelines for Python code quality management using Ruff'
applyTo: '**/*.py, **/pyproject.toml, **/ruff.toml, **/.ruff.toml'
---

# Ruff Code Quality Management

Ruff is a high-performance Python linter and formatter implemented in Rust. This file explains best practices for code quality management using Ruff.

## Scope and Tool Overview

- Use Ruff for Python 3.10+ projects as unified linter and formatter
- Replace Black, Flake8, isort, pydocstyle, pyupgrade with Ruff
- Leverage Ruff's 10-100x speed improvement and 800+ rules

## Basic Configuration

### Installation

- Install Ruff using `uv tool install ruff@latest` and `uv add --dev ruff`
- Alternatively use `pip install ruff` if uv is not available

```bash
uv tool install ruff@latest
uv add --dev ruff
```

### Basic Execution Commands

- Run linter with auto-fix using `ruff check --fix .`
- Run formatter using `ruff format .`
- Execute both in order: `ruff check --fix . && ruff format .`

```bash
ruff check --fix . && ruff format .
```

## Linter Best Practices

### Rule Selection

- Start with minimal rules: `["E4", "E7", "E9", "F"]`
- Use recommended balanced set: `["E", "F", "UP", "B", "SIM", "I", "N", "ASYNC"]`
- Add rules incrementally to avoid overwhelming changes

```toml
# Recommended configuration
[tool.ruff.lint]
select = ["E", "F", "UP", "B", "SIM", "I", "N", "ASYNC"]
```

### Adding New Rules

- Use `extend-select` instead of `select` to add rules incrementally

```toml
[tool.ruff.lint]
extend-select = ["B", "UP", "I"]
```

### Per-File Exception Configuration

- Configure per-file ignores using `[tool.ruff.lint.per-file-ignores]`
- Allow unused imports in `__init__.py` with `F401`
- Allow `assert` in test files with `S101`

```toml
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
"tests/**/*.py" = ["S101"]
```

### Error Suppression

- Suppress line-level errors using `# noqa: RULE`
- Suppress multiple rules with comma-separated list: `# noqa: E741, F841`

```python
import os  # noqa: F401
x = 1  # noqa: E741, F841
```

### Detecting Unused noqa Comments

- Detect and remove unused `noqa` comments using `ruff check --extend-select RUF100 --fix`

### Fix Safety

- Apply only safe fixes by default with `ruff check --fix`
- Include unsafe fixes using `--unsafe-fixes` flag
- Control globally via `[tool.ruff] unsafe-fixes = true`

```toml
[tool.ruff]
unsafe-fixes = true
```

## Formatter Best Practices

### Black-Compatible Configuration

- Set `line-length = 88` and `quote-style = "double"` for Black compatibility
- Use `skip-magic-trailing-comma = false` to match Black behavior

```toml
[tool.ruff]
line-length = 88

[tool.ruff.format]
quote-style = "double"
```

### Docstring Formatting

- Enable docstring code formatting with `docstring-code-format = true`
- Use `docstring-code-line-length = "dynamic"` for automatic line length

```toml
[tool.ruff.format]
docstring-code-format = true
docstring-code-line-length = "dynamic"
```

### Format Suppression

- Use `# fmt: off` and `# fmt: on` for block-level suppression
- Use `# fmt: skip` for single-line suppression

### Avoiding Formatter and Linter Conflicts

- Ignore formatter-conflicting rules: `W191`, `E111`, `E114`, `E117`, `D206`, `D300`, `COM812`, `COM819`, `ISC002`

```toml
[tool.ruff.lint]
ignore = ["W191", "E111", "E114", "E117", "D206", "D300", "COM812", "COM819", "ISC002"]
```

## Configuration File Structure

### Recommended Minimal Configuration (pyproject.toml)

- Configure Ruff in `pyproject.toml` with minimal required settings
- Specify Python version, line length, and core rule sets
- Use `extend-exclude` for ignored directories

```toml
[project]
requires-python = ">=3.10"

[tool.ruff]
line-length = 88
target-version = "py310"
required-version = ">=0.14.0"

# File search configuration
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

### Configuration with ruff.toml

- Use `ruff.toml` or `.ruff.toml` for standalone configuration (omit `[tool.ruff]` header)

```toml
line-length = 88

[lint]
select = ["E", "F"]

[format]
quote-style = "double"
```

### Configuration Inheritance

- Extend parent configuration using `extend = "../pyproject.toml"`
- Override specific settings in child configuration

```toml
[tool.ruff]
extend = "../pyproject.toml"
line-length = 100  # Override parent setting
```

## Docstrings and Type Hints

### Enforcing Docstring Style

- Enable docstring rules with `select = ["D"]`
- Specify convention: `google`, `numpy`, or `pep257`

```toml
[tool.ruff.lint]
select = ["D"]

[tool.ruff.lint.pydocstyle]
convention = "google"
```

### Type Hint Related Rules

- Enable type hint rules with `select = ["ANN", "TCH"]`
- Use `future-annotations = true` to enable `from __future__ import annotations`

```toml
[tool.ruff.lint]
select = ["ANN", "TCH"]
future-annotations = true
```

## CI/CD Integration

### GitHub Actions

- Use `astral-sh/ruff-action@v3` for basic setup
- Use `--output-format=github` for proper annotations
- Run formatter check with `ruff format --check`

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

- Place `ruff-check` before `ruff-format` (order matters)
- Use `args: [--fix]` for auto-fixing

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.10
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
```

## Editor Integration

### VS Code

- Set `editor.defaultFormatter` to `charliermarsh.ruff`
- Enable `editor.formatOnSave` and `source.fixAll` on save
- Configure `ruff.lint.run` to `onSave`

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

- Enable Ruff in Settings → Tools → Ruff and set executable path

## Migration Guide

### Migrating from Black

- Remove `[tool.black]` section from pyproject.toml
- Add Ruff format configuration with `line-length = 88` and `quote-style = "double"`

```toml
[tool.ruff]
line-length = 88

[tool.ruff.format]
quote-style = "double"
```

### Migrating from Flake8

- Transfer `max-line-length` to `[tool.ruff] line-length`
- Convert ignored codes to `[tool.ruff.lint] ignore` array
- Map plugins: flake8-bugbear→B, flake8-simplify→SIM, pep8-naming→N, flake8-comprehensions→C4

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
ignore = ["E203", "W503"]
```

### Migrating from isort

- Remove `[tool.isort]` section
- Enable import sorting with `select = ["I"]`
- Configure `known-first-party` under `[tool.ruff.lint.isort]`

```toml
[tool.ruff.lint]
select = ["I"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]
```

## Troubleshooting

### Line Length Violations Remain

- Ignore `E501` since formatter applies best-effort line length

```toml
[tool.ruff.lint]
ignore = ["E501"]
```

### Configuration Not Applied

- Check settings with `ruff check --show-settings path/to/file.py`
- Clear cache using `ruff clean`

### Jupyter Notebook Support

- Ignore notebook-specific rules via per-file-ignores

```toml
[tool.ruff.lint.per-file-ignores]
"*.ipynb" = ["E402", "T20"]
```

## Configuration Verification

- Verify active settings with `ruff check --show-settings path/to/file.py`
- List configuration files with `ruff check --show-files`
- Clear cache using `ruff clean`

## Phased Adoption Plan

- **Week 1**: Install Ruff, run with defaults, fix auto-fixable errors
- **Week 2**: Enable additional rule sets, integrate CI/CD, configure pre-commit
- **Week 3**: Enable formatter, set up editor integration
- **Week 4**: Fine-tune configuration, remove old Flake8/Black settings

## Setup Checklist

- Install Ruff to project
- Add basic configuration to `pyproject.toml`
- Integrate into CI/CD pipeline
- Install Ruff extension in editor
- Configure pre-commit hooks
- Notify team members

## Reference Links

- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Rule Reference](https://docs.astral.sh/ruff/rules/)
