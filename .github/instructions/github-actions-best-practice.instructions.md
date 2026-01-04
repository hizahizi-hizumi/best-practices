````instructions
---
description: 'Best practices for secure and maintainable GitHub Actions workflows and composite actions'
applyTo: '**/.github/workflows/**/*.yml, **/.github/workflows/**/*.yaml, **/.github/actions/**/action.yml, **/.github/actions/**/action.yaml, **/action.yml, **/action.yaml'
---

# GitHub Actions Best Practices

Guidelines for creating secure, maintainable, and efficient GitHub Actions workflows and composite actions.

## Purpose and Scope

Apply security-first principles to GitHub Actions workflows and composite actions.
Prioritize least privilege for all permissions.
Protect supply chain integrity by pinning actions to commit SHAs.
Safeguard secrets through environment variables.
Optimize workflow execution with caching and concurrency control.

## Applicable Files

Workflows: `.github/workflows/*.yml` and `.github/workflows/*.yaml`
Composite actions: `action.yml` and `action.yaml`

## Security (Priority 1)

### Permissions

Define explicit `permissions` at workflow level for every workflow file.
Set `contents: read` as the default permission.
Grant write permissions only at job level for specific operations.
Never rely on default implicit permissions.
**Rationale**: Explicit permissions prevent privilege escalation and limit the blast radius of compromised workflows.

```yaml
# Recommended
permissions:
  contents: read

jobs:
  triage:
    permissions:
      contents: read
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      - run: echo "triage"

# Not Recommended: missing permissions (implicit wide permissions)
jobs:
  deploy:
    runs-on: ubuntu-latest
```

### Supply Chain Security

Pin all external actions to full commit SHA (40 characters).
Pin reusable workflows to commit SHA.
Avoid `@main`, `@v1`, or any mutable tag references.
Review action source code before pinning new versions.
**Rationale**: SHA pinning prevents supply chain attacks where action maintainers push malicious code to mutable references.

```yaml
# Recommended
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

# Not Recommended
- uses: some-owner/some-action@main
```

### Script Injection Prevention

Never concatenate untrusted input directly in `run:` scripts.
Treat all GitHub context values as untrusted: `github.event.pull_request.title`, `github.event.issue.body`, `github.head_ref`.
Pass untrusted values through `env:` block only.
Quote environment variables in shell: `"$VAR"` not `$VAR`.
Use `printf '%s\n' "$VAR"` instead of `echo "$VAR"`.
**Rationale**: Direct interpolation enables command injection when attacker controls PR titles, branch names, or issue bodies.

```yaml
# Recommended
- name: Safe print
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"

# Not Recommended
- name: Unsafe
  run: |
    echo "Title: ${{ github.event.pull_request.title }}"
```

## Authentication

### OIDC (OpenID Connect)

Use OIDC for AWS, GCP, and Azure authentication instead of static credentials.
Set `permissions: id-token: write` when using OIDC.
Configure cloud provider trust policies to restrict token usage by repository and branch.
Avoid storing cloud credentials as GitHub secrets.
**Rationale**: OIDC tokens are short-lived and automatically rotated, eliminating static credential leakage risk.

```yaml
permissions:
  contents: read
  id-token: write
```

## Secrets Management

Pass secrets through `env:` at step or job level.
Never pass secrets as command-line arguments or script parameters.
Mask secrets early with `echo "::add-mask::$SECRET"`.
Design workflows to skip or fail gracefully when secrets are unavailable.
Avoid secrets in `if:` conditions (they are visible in workflow logs).
Test workflows work correctly in fork PRs where secrets are unavailable.
**Rationale**: Command-line arguments appear in logs and process lists; environment variables remain protected.

```yaml
# Recommended
- name: Use secret
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: |
    ./tool --token "$API_TOKEN"

# Not Recommended
- name: Unsafe secret
  run: |
    ./tool --token "${{ secrets.API_TOKEN }}"
```

## Environments

Define `environment` field for all deployment jobs.
Require manual approval for production environments in repository settings.
Apply branch restrictions to environment protection rules.
Set deployment protection rules to limit environment access.
Document that `environment` context is unavailable in reusable workflows.
**Rationale**: Environment protection rules add human approval gates and audit trails for critical operations.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - run: echo "deploy"
```

## Workflow Structure

### Naming and Organization

Name workflows with clear, descriptive titles using `name:` field.
Use kebab-case for workflow file names: `ci-tests.yml`, `deploy-production.yml`.
Group related jobs under a single workflow file.
Split unrelated concerns into separate workflow files.

### Triggers

Specify explicit event triggers rather than relying on defaults.
Use `pull_request` for CI checks, not `push` (to avoid duplicate runs).
Limit `push` triggers to specific branches: `branches: [main, develop]`.
Use `workflow_dispatch` to enable manual triggering.
**Rationale**: Explicit triggers prevent unexpected workflow executions and improve resource efficiency.

```yaml
# Recommended
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

# Not Recommended
on: [push]  # Triggers on all branches
```

### Job Dependencies

Use `needs:` to define job dependencies explicitly.
Run independent jobs in parallel by omitting `needs:`.
Organize jobs in logical stages: test → build → deploy.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
  build:
    needs: test
    runs-on: ubuntu-latest
  deploy:
    needs: build
    runs-on: ubuntu-latest
```

## Concurrency Control

Define `concurrency` group for all workflows that modify shared state.
Use `github.head_ref || github.ref` pattern to support both PRs and pushes.
Set `cancel-in-progress: true` for CI and test workflows.
Set `cancel-in-progress: false` for deployment workflows.
Include workflow name in concurrency group for uniqueness.
**Rationale**: Concurrency control prevents race conditions in deployments and reduces wasted CI runs.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

## Caching

Cache dependency directories to speed up workflow execution.
Use lock file hash as primary cache key: `${{ hashFiles('**/package-lock.json') }}`.
Define `restore-keys` with progressively less specific patterns.
Exclude sensitive paths: auth files, credentials, API keys.
Avoid caching `node_modules` in repositories with native dependencies.
Set cache size limits to prevent excessive storage usage.
**Rationale**: Lock file hashing ensures reproducible builds; excluding secrets prevents credential leakage through cache poisoning.

```yaml
# Recommended
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# Not Recommended: caching sensitive paths
- uses: actions/cache@v4
  with:
    path: ~/.npmrc  # Contains auth tokens
```

## Job Configuration

### Runner Selection

Use `ubuntu-latest` for Linux jobs unless specific version required.
Specify exact runner version for reproducibility: `ubuntu-22.04`.
Use matrix strategy to test across multiple OS or versions.
Prefer GitHub-hosted runners over self-hosted for public repositories.
**Rationale**: GitHub-hosted runners provide isolation and security for untrusted code.

### Timeouts

Set `timeout-minutes` for all jobs to prevent runaway processes.
Use shorter timeouts for fast jobs: `timeout-minutes: 5`.
Set realistic timeouts for long jobs: `timeout-minutes: 60`.
**Rationale**: Timeouts prevent billing issues from infinite loops or stuck processes.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
```

### Conditional Execution

Use `if:` to skip jobs or steps conditionally.
Check event type: `if: github.event_name == 'push'`.
Check branch: `if: github.ref == 'refs/heads/main'`.
Avoid secrets in `if:` conditions (they appear in logs).

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
```

## Reusable Workflows

Call reusable workflows with `uses:` at job level.
Pin reusable workflows to commit SHA.
Pass inputs through `with:` block.
Pass secrets explicitly through `secrets:` block.
Avoid `secrets: inherit` except for trusted organization workflows.
Document required inputs and secrets in workflow file.
**Rationale**: Explicit input and secret declarations follow least privilege principle and improve maintainability.

```yaml
# Recommended
jobs:
  call:
    uses: org/repo/.github/workflows/reusable.yml@b4ffde65f46336ab88eb53be808477a3936bae11
    with:
      config-path: .github/labeler.yml
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}

# Not Recommended
jobs:
  call:
    uses: org/repo/.github/workflows/reusable.yml@main  # Mutable reference
    secrets: inherit  # Passes all secrets
```

## Self-Hosted Runners

Avoid self-hosted runners for public repositories.
Use self-hosted runners only for private repositories with trusted contributors.
Restrict runner access with runner groups and repository permissions.
Run self-hosted runners in ephemeral environments (containers, VMs).
Regularly update runner software and host OS.
Isolate runners from internal networks and sensitive resources.
**Rationale**: Self-hosted runners executing untrusted code can compromise infrastructure and exfiltrate secrets.

## Matrix Strategy

Use matrix strategy to test across multiple versions or configurations.
Define matrix in `strategy.matrix` with arrays of values.
Limit matrix combinations to reduce workflow cost.
Use `include` to add specific combinations.
Use `exclude` to remove unwanted combinations.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20]
    exclude:
      - os: macos-latest
        node: 18
```

## Performance Optimization

### Minimize Checkout

Use `actions/checkout` with `fetch-depth: 1` for shallow clone.
Use `sparse-checkout` to clone only needed directories.

### Parallel Execution

Run independent jobs in parallel without `needs:` dependencies.
Split long test suites into parallel jobs.
Use matrix strategy to distribute work.

### Artifact Management

Upload only necessary artifacts.
Set artifact retention period: `retention-days: 7`.
Compress artifacts before upload.
Avoid uploading large binaries unnecessarily.

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: results/
    retention-days: 7
```

### Output and Logging

Use `echo "::group::Title"` to group log output.
Collapse verbose output with grouping.
Use `echo "::notice::Message"` for important messages.
Use `echo "::warning::Message"` for warnings.
Use `echo "::error::Message"` for errors.

## Common Anti-Patterns

### Missing Permissions

**Problem**: No `permissions` specified, granting broad default permissions.
**Solution**: Add explicit `permissions: contents: read` at workflow level.

### Mutable Action References

**Problem**: Using `@main`, `@v1`, or branch names for actions.
**Solution**: Pin to commit SHA: `actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11`.

### Script Injection

**Problem**: `run: echo "${{ github.event.pull_request.title }}"`.
**Solution**: Pass through env: `env: TITLE: ${{ github.event.pull_request.title }}`, then `run: printf '%s\n' "$TITLE"`.

### Exposed Secrets

**Problem**: `run: deploy.sh ${{ secrets.API_KEY }}`.
**Solution**: `env: API_KEY: ${{ secrets.API_KEY }}`, then `run: deploy.sh "$API_KEY"`.

### Cached Credentials

**Problem**: Caching `~/.npmrc`, `~/.docker/config.json`, or credential files.
**Solution**: Exclude credential paths from cache; use OIDC instead.

### Inefficient Checkouts

**Problem**: Full repository history fetched on every run.
**Solution**: Use `fetch-depth: 1` for shallow clone.

### Missing Timeouts

**Problem**: Job runs indefinitely if process hangs.
**Solution**: Add `timeout-minutes: 15` to all jobs.

### Overly Broad Triggers

**Problem**: `on: push` triggers on all branches and commits.
**Solution**: Specify branches: `on: push: branches: [main]`.

## References

- [GitHub Actions Security Best Practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [GITHUB_TOKEN Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)
- [OpenID Connect in Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
````
