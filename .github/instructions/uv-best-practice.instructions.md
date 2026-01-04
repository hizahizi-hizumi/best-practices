---
description: 'Best practices for Python project management with uv'
applyTo: 'pyproject.toml, uv.lock, **/*.py'
---

# uv Project Management Best Practices

## Purpose and Scope

Use uv (v0.9.21+) as the standard Python package manager for all Python projects, replacing pip, pip-tools, and poetry.

## Tools and Versions

- **uv**: 0.9.21 or later (Astral's Rust-based package manager)
- **Python**: 3.10+
- **Platform**: Cross-platform (Linux, macOS, Windows)

## Core Principles

- Use uv commands exclusively for dependency management
- Include `uv.lock` in version control
  **Rationale**: Ensures reproducible builds across all environments
- Add `.venv/` to `.gitignore`
  **Rationale**: Virtual environments are environment-specific and should not be shared
- Pin Python version with `.python-version` file
  **Rationale**: Guarantees consistent Python version across team members
- Use `--locked` flag in CI/CD
  **Rationale**: Prevents dependency resolution differences between environments

## Project Structure

```
my-project/
├── .python-version          # Pin Python version (required)
├── .gitignore              # Exclude .venv/
├── pyproject.toml          # Project metadata and dependencies
├── uv.lock                 # Lock file (required in VCS)
├── src/
│   └── my_project/
│       ├── __init__.py
│       └── main.py
└── tests/
    └── test_main.py
```

Version Control Rules:
- Add `.venv/` to `.gitignore`
- Include `uv.lock` in version control
- Include `.python-version` in version control

## Dependency Management

### Adding Dependencies

```bash
# Production dependency
uv add "httpx>=0.24.0"

# Development dependency
uv add --group dev pytest ruff mypy

# Optional dependency
uv add --optional ml numpy pandas

# Group dependency (recommended)
uv add --group lint ruff mypy
uv add --group test pytest pytest-cov
```

### pyproject.toml Configuration

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "Clear description"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31.0",      # Specify lower bound
    "pandas>=2.0.0,<3.0.0",  # Major version constraint
]

[dependency-groups]
lint = ["ruff>=0.1.0", "mypy>=1.0.0"]
test = ["pytest>=7.0.0", "pytest-cov>=4.0.0"]

[tool.uv]
python-preference = "managed"  # Prefer managed Python
default-groups = ["lint", "test"]
```

### Version Specification Rules

- Specify lower bound for all direct dependencies
  **Rationale**: Ensures minimum feature availability
- Avoid upper bounds unless compatibility is explicitly limited
  **Rationale**: Allows receiving bug fixes and security patches
- Avoid `==` pinning (let lock file handle exact versions)
  **Rationale**: Lock file provides reproducibility while allowing flexibility in `pyproject.toml`

Good:
```toml
dependencies = [
    "requests>=2.31.0",
    "pandas>=2.0.0,<3.0.0",
]
```

Bad:
```toml
dependencies = [
    "requests",           # No version
    "pandas==2.1.3",      # Over-constrained
]
```

## Python Version Management

```bash
# Install specific version
uv python install 3.12

# Pin project Python version (required)
uv python pin 3.12

# Test multiple versions
uv python install 3.10 3.11 3.12
```

Configure in pyproject.toml:
```toml
[project]
requires-python = ">=3.10"  # Avoid overly broad range

[tool.uv]
python-preference = "managed"  # Use managed Python
```

## Environment Sync and Lock Files

### Basic Workflow

```bash
# Add dependency (automatically updates lock)
uv add package-name

# Sync environment
uv sync

# CI/CD (required)
uv sync --locked

# Run commands
uv run python script.py
uv run pytest
```

### Sync Options

```bash
# Development: all dependencies
uv sync --all-extras

# Production: exclude dev dependencies
uv sync --no-dev --locked

# CI/CD: verify lock integrity
uv sync --locked --frozen

# Specific groups only
uv sync --group lint --group test
```

Good (CI/CD):
```yaml
- name: Install dependencies
  run: uv sync --locked --no-dev
- name: Run tests
  run: uv run pytest
```

Bad (CI/CD):
```yaml
# Ignoring lock file
- run: uv sync

# Including dev dependencies in production
- run: uv sync --all-extras
```

## CI/CD Configuration

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

### Matrix Testing

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

### Caching Strategy

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

## Docker Configuration

### Production Dockerfile

```dockerfile
FROM python:3.12-slim-bookworm

# Install uv (pin version)
COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

WORKDIR /app

# Install dependencies (layer cache optimization)
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

### Optimized Multi-stage Build

```dockerfile
FROM python:3.12-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:0.9.21 /uv /uvx /bin/

WORKDIR /app

# Enable bytecode compilation
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

Docker Best Practices:
- Pin uv version explicitly
- Use layer caching (separate dependencies from code)
  **Rationale**: Reduces rebuild time and image size
- Use `--no-dev` in production
- Enable bytecode compilation: `UV_COMPILE_BYTECODE=1`
- Use non-editable installs: `--no-editable`
- Use multi-stage builds to minimize image size

## Workspaces and Monorepos

Configure in root `pyproject.toml`:
```toml
[tool.uv.workspace]
members = ["packages/*", "apps/*"]

[tool.uv.sources]
common-lib = { workspace = true }
```

Use for: tightly-coupled packages, monorepos, shared dependencies
Avoid for: conflicting dependencies, different Python versions

## Common Workflows

### New Project
```bash
uv init my-project && cd my-project
uv python pin 3.12
uv add requests pandas
uv add --group dev pytest ruff mypy
git init && echo ".venv/" >> .gitignore
git add . && git commit -m "Initial commit"
```

### Migration from pip/poetry
```bash
uv add --requirements requirements.txt  # From requirements.txt
uv sync                                  # From poetry
uv lock
git add pyproject.toml uv.lock .python-version
```

### Dependency Upgrade
```bash
uv lock --upgrade                # All dependencies
uv lock --upgrade-package pkg    # Specific package
uv sync && uv run pytest         # Test
```

### Multi-version Testing
```bash
uv sync --resolution lowest && uv run pytest
uv sync --resolution highest && uv run pytest
uv sync  # Restore
```

## Anti-Patterns

### Version Control Errors
Bad: `git add .venv/` or `echo "uv.lock" >> .gitignore`
Good: `echo ".venv/" >> .gitignore && git add uv.lock .python-version`

### CI/CD Errors
Bad: `uv sync`
Good: `uv sync --locked`

### Version Specification Errors
Bad: `dependencies = ["requests", "pandas"]`
Good: `dependencies = ["requests>=2.31.0", "pandas>=2.0.0,<3.0.0"]`

### Production Configuration Errors
Bad: `RUN uv sync --all-extras`
Good: `RUN uv sync --locked --no-dev`

### Mixing Package Managers
Bad: `uv add requests && pip install pandas`
Good: `uv add requests pandas`

## Troubleshooting

```bash
# Lock file out of date
uv lock && uv sync

# Dependency conflicts
uv sync --verbose
uv sync --resolution lowest
uv lock --upgrade-package problematic-package

# Cache issues
uv cache clean
uv sync --no-cache
uv sync --reinstall-package numpy

# Build failures (add to pyproject.toml)
[tool.uv]
no-build-isolation-package = ["problem-package"]
```

## Validation

```bash
test -f pyproject.toml && test -f uv.lock && test -f .python-version && \
grep -q ".venv/" .gitignore && \
uv lock --check && \
uv sync --locked && \
uv run pytest && \
echo "✓ All checks passed"
```

## Security

```toml
[tool.uv]
exclude-newer = "1 week"  # Cooldown for new packages

[tool.uv.exclude-newer-package]
requests = "30 days"
```

Best practices:
- Set cooldown period for new versions (allows time to discover vulnerabilities)
- Use `--locked` in CI/CD
- Scan vulnerabilities regularly with `pip-audit`

## Essential Checklist

- [ ] `.venv/` in `.gitignore`
- [ ] `uv.lock` in version control
- [ ] `.python-version` pinned
- [ ] Lower bounds for all dependencies
- [ ] `--locked` in CI/CD
- [ ] `--no-dev` in production
- [ ] `python-preference = "managed"` in pyproject.toml

## References

- [Official Docs](https://docs.astral.sh/uv/)
- [GitHub](https://github.com/astral-sh/uv)
- [setup-uv Action](https://github.com/astral-sh/setup-uv)
