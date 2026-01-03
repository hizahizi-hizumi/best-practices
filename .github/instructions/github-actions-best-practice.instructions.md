---
description: 'GitHub Actionsで安全かつ保守可能なワークフローを書くためのベストプラクティス規則'
applyTo: '**/.github/workflows/**/*.yml, **/.github/workflows/**/*.yaml, **/.github/actions/**/action.yml, **/.github/actions/**/action.yaml, **/action.yml, **/action.yaml'
---

# GitHub Actions ベストプラクティス

GitHub Actions のワークフローや composite action（`action.yml`）を作成・修正するときに従うガイドラインです。
セキュリティ（最小権限・サプライチェーン対策・Secrets保護）を最優先し、次に運用性（可観測性・並行制御）とパフォーマンス（キャッシュ）を最適化します。

## 前提

- ワークフローは `.github/workflows/*.yml` / `.yaml`
- composite action は `action.yml` / `action.yaml`
- 公式リファレンスに沿って、YAML 構文キー（`permissions`, `concurrency`, `strategy`, `workflow_call` 等）を正しく使う

## セキュリティ（最優先）

### 最小権限（必須）

- 可能な限り `permissions` を明示し、`contents: read` をデフォルトとする
- 例外的に書き込み権限が必要な job のみ、job 単位で権限を上げる

```yaml
# ✅ 推奨: read-only をデフォルトに
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
```

### サプライチェーン対策

- 外部リポジトリの action / reusable workflow は、可能な限り **コミットSHAで固定**する
- `@main` や可変ブランチ参照は避ける

```yaml
# ✅ 推奨
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

# ❌ 非推奨
- uses: some-owner/some-action@main
```

### スクリプトインジェクション対策

- PRタイトル、ブランチ名、issue本文など「攻撃者が制御し得る値」を `run:` に直結しない
- 値は `env:` に入れ、シェルで安全にクオートして扱う（`printf` を優先）

```yaml
# ✅ 推奨
- name: Safe print
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

## 認証（OIDC / トークン）

- クラウド認証や外部連携は、長期Secretsより **OIDC** を優先する
- OIDC を使う場合は `permissions: id-token: write` を付与する

```yaml
permissions:
  contents: read
  id-token: write
```

## Secrets と変数

- Secrets は job の `env:` として渡し、ログに出さない（出る可能性がある場合は `::add-mask::` も併用）
- コマンドライン引数に Secrets を渡さない（ログやプロセス一覧に残るリスクがあるため）
- fork PR / Dependabot / reusable workflow の制約を前提に、Secrets が無い状況でも安全に動作する設計にする

```yaml
# ✅ 推奨: env で渡す
- name: Use secret
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: |
    ./tool --token "$API_TOKEN"
```

## 環境（Environments）

- 本番デプロイ job は `environment` を指定し、承認・待機・ブランチ制限などの保護ルールを適用する

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - run: echo "deploy"
```

- 再利用ワークフローでは呼び出し側から `environment` を渡せない前提で設計する（環境シークレットの扱いに注意）

## 並行制御（concurrency）

- デプロイや同一ブランチのCIは `concurrency` で二重実行・競合を防ぐ
- `github.head_ref || github.ref` のようにフォールバックを入れる

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

## キャッシュ

- 機密情報が混入し得るパスをキャッシュ対象にしない
- ロックファイルのハッシュをキーにして再現性を担保し、`restore-keys` でヒット率を上げる

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

## 再利用（Reusable workflows）

- 再利用ワークフローは job 直下で `uses:` を使う（step ではない）
- Secrets は必要なものだけ明示して渡す（`secrets: inherit` の濫用は避ける）

```yaml
jobs:
  call:
    uses: org/repo/.github/workflows/reusable.yml@<SHA>
    with:
      config-path: .github/labeler.yml
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

## self-hosted runner

- self-hosted runner は信頼境界が変わるため、公開リポジトリや不特定のPRを実行する用途では慎重に扱う
- ランナーグループ等でアクセスを絞り、意図しないワークフローから実行されないようにする

## アンチパターン

- `permissions` 未指定（暗黙の広い権限に依存）
- サードパーティ action を `@main` 参照
- `run:` で untrusted input を文字列連結して実行
- Secrets をログに出す／CLI引数で渡す
- キャッシュ対象パスに Secrets が混ざる（例: 認証情報ファイル、`.npmrc`、鍵ファイル等）
