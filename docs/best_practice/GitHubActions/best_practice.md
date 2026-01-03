# GitHub Actions ベストプラクティス

## 目次

1. [概要](#概要)
2. [セキュリティ（最優先）](#セキュリティ最優先)
3. [認証・権限（GITHUB_TOKEN / OIDC）](#認証権限github_token--oidc)
4. [Secrets と変数](#secrets-と変数)
5. [環境（Environments）とデプロイ保護](#環境environmentsとデプロイ保護)
6. [ワークフロー設計](#ワークフロー設計)
7. [キャッシュと成果物](#キャッシュと成果物)
8. [再利用（Reusable workflows / Composite actions）](#再利用reusable-workflows--composite-actions)
9. [ランナー（GitHub-hosted / self-hosted）](#ランナーgithub-hosted--self-hosted)
10. [リポジトリ/組織のガバナンス設定](#リポジトリ組織のガバナンス設定)
11. [よくある落とし穴と回避策](#よくある落とし穴と回避策)
12. [最小テンプレ例](#最小テンプレ例)

---

## 概要

GitHub Actions は「CI/CD」「自動化」「デプロイ」を簡単にしますが、同時に **サプライチェーン・シークレット漏えい・権限過大** のリスクも増やします。
このドキュメントは、公式ドキュメント（Actionsの構文・セキュリティ・OIDC・Secrets・キャッシュ・並行制御・再利用・環境・設定）をベースに、一般的な実務の観点（外部のセキュリティチェックリスト等でよく挙がる論点）も統合して、実装しやすいガイドラインとしてまとめます。

---

## セキュリティ（最優先）

### 1) 第三者アクション/再利用ワークフローは「固定」する

- サードパーティ製 `uses:` は、可能な限り **コミットSHAで固定** します。
- これはサプライチェーン攻撃（タグの差し替え・リポジトリ侵害）への耐性を上げる基本です。

```yaml
# ✅ 推奨: コミットSHAで固定
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

# ⚠️ 許容（ただし更新で挙動が変わる）
- uses: actions/checkout@v5

# ❌ 非推奨: main 等の移動ブランチ
- uses: some-owner/some-action@main
```

### 2) “信頼できない入力” を `run:` に直結しない

- `github.*`（例: PRタイトル、ブランチ名、issue本文など）は、**攻撃者が制御できる可能性**があります。
- 文字列連結でシェルに渡すと **スクリプトインジェクション** の温床になります。

```yaml
# ❌ 非推奨: PRタイトルをそのままシェルに埋め込む
- run: echo "${{ github.event.pull_request.title }}"

# ✅ 推奨: まず環境変数に入れて、クオートして扱う
- name: Print PR title safely
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

### 3) Secrets を「使う」より先に「不要にする」

外部の一般的なセキュリティガイドでも共通して強調されますが、Secrets を増やすほど漏えい面積が増えます。

- 可能なら **OIDC** に置き換え、短命でスコープが狭いトークンを使う
- 長期トークン（PAT等）を使う場合は **用途別・最小権限・定期更新**

### 4) 変更管理（ワークフロー/アクションの変更は“コード変更”として扱う）

- `.github/workflows/**` は実質的に **実行権限を持つコード** です。
- CODEOWNERS 等でレビューを必須にし、PRでの変更は依存関係レビュー（Dependabot等）も併用します。

---

## 認証・権限（GITHUB_TOKEN / OIDC）

### 1) `permissions` は必ず最小権限で明示する

- ワークフロー全体、または job 単位で `permissions` を明示します。
- 公式仕様上、`permissions` を一部でも指定すると、指定しなかった権限が `none` になる（暗黙に広がらない）ため、**必要権限を列挙**する運用が安全です。

```yaml
name: CI
on:
  pull_request:

# ✅ 推奨: まず read-only をデフォルトに
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: npm ci
      - run: npm test
```

PRにコメントしたり、ラベルを付けたりする job だけ権限を上げます。

```yaml
jobs:
  triage:
    permissions:
      contents: read
      pull-requests: write
    uses: org/repo/.github/workflows/triage.yml@<SHA-or-tag>
```

### 2) クラウド認証は OIDC を優先する

- クラウドへのデプロイや外部システム連携で、長期シークレットを渡す代わりに OIDC を使うことで、短命で絞った認証が可能になります。
- OIDC を使う job では `permissions: id-token: write` が必要です。

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      # 以降: 各クラウドのOIDC連携アクション/設定に従う
```

---

## Secrets と変数

### 1) Secrets の利用制約を前提に設計する

- fork 由来の `pull_request` 実行では、Secrets が渡らない（`GITHUB_TOKEN` 以外）
- Dependabot 由来の実行では Secrets が使えないケースがある
- 再利用ワークフロー（`workflow_call`）へは Secrets が自動では渡らない（明示または `inherit` が必要）

### 2) ログに Secrets を出さない

- Secrets はマスクされますが、**確実ではありません**（加工・分割・エンコード等で漏れる）。
- どうしても出力に含まれる可能性がある値は、`::add-mask::` で追加マスクします。

```yaml
- name: Mask value
  run: |
    echo "::add-mask::${SECRET_VALUE}"
  env:
    SECRET_VALUE: ${{ secrets.SOME_SECRET }}
```

### 3) コマンドライン引数に Secrets を渡さない

- 引数はプロセス一覧やログに残りやすいので、可能なら環境変数または stdin を使います。

---

## 環境（Environments）とデプロイ保護

### 1) 本番系は Environments でゲートする

- `jobs.<job_id>.environment` を指定すると、その環境に設定した保護ルール（承認、待機、ブランチ制限等）が適用されます。

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: echo "deploy"
```

### 2) 環境シークレット/変数は「承認後にのみ」使えるようにする

- 環境シークレットは、その環境を参照する job のみが利用できます。
- さらに、必須レビュー等の保護ルールが通った後にのみアクセス可能にできるため、
  “承認前に漏れる” リスクを下げられます。

### 3) 再利用ワークフローと環境シークレットの注意

- `workflow_call` では `environment` キーワードがサポートされないため、呼び出し側から環境シークレットを「受け渡す」設計はできません。
- もし再利用ワークフロー側の job で `environment` を指定すると、呼び出し側から渡した Secrets ではなく、その環境の Secrets が使われる点に注意します。

---

## ワークフロー設計

### 1) 並行制御（concurrency）で二重実行や競合を防ぐ

- デプロイ、マイグレーション、同一ブランチの重複CIなどは `concurrency` で保護します。
- `concurrency.group` 名は **大文字小文字を区別しない**ため、生成規則を統一します。
- `pull_request` などで `github.head_ref` が空になるケースがあるため、フォールバックを入れます。

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

### 2) 可観測性: `GITHUB_STEP_SUMMARY` を活用する

- 重要な情報（テスト結果の要約、リンク、生成物）を summary に集約すると運用が楽になります。

### 3) マトリクス（strategy.matrix）は粒度を調整する

- 便利ですが、過剰に増やすとキュー待ち/コスト増になります。
- `fail-fast` や `max-parallel` を使い、CIの意味（早期フィードバック vs 網羅性）に合わせます。

---

## キャッシュと成果物

### 1) キャッシュに機密情報を入れない

- 公式の注意点として、キャッシュは意図しない参照経路を持ち得ます。
- 「Secrets が入る可能性のあるパス」を `actions/cache` の対象にしない。

### 2) キャッシュキーは「再現性」と「ヒット率」を両立する

- 依存関係ロックファイル（例: `package-lock.json` / `poetry.lock`）のハッシュをキーにします。
- `restore-keys` でフォールバックするとヒット率が上がります。

```yaml
- name: Cache
  uses: actions/cache@v4
  with:
    path: |
      ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

### 3) キャッシュと成果物（artifacts）を混同しない

- キャッシュ: 速度最適化（再利用が前提、破棄され得る）
- 成果物: 証跡/受け渡し（保持期間の管理対象）

---

## 再利用（Reusable workflows / Composite actions）

### 1) 再利用ワークフローは job 直下で `uses:`

- 再利用ワークフローは step ではなく **job** として呼び出します。
- 他リポジトリ参照では `@{ref}`（SHA/タグ/ブランチ）が必須で、SHA固定が最も安全です。

```yaml
jobs:
  call:
    uses: octo-org/example/.github/workflows/reusable.yml@<SHA>
    with:
      config-path: .github/labeler.yml
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

### 2) Secrets は明示して渡す（または `inherit`）

- `secrets: inherit` は便利ですが、「渡るSecretsの範囲」が広がりやすいので、
  可能なら **必要なものだけ列挙** します。

### 3) Composite action は“チーム内標準化”に効く

- よく使う一連の steps を `.github/actions/**/action.yml` に閉じ込めると、
  YAMLの複製を減らせます。
- ただし composite action も実行コードなので、入力の扱いは安全に（インジェクション耐性）します。

---

## ランナー（GitHub-hosted / self-hosted）

### 1) self-hosted は「信頼境界」が変わる

- self-hosted runner は強力ですが、侵害時の影響も大きいので、
  公開リポジトリ/不特定のPRを実行する用途には原則として慎重に判断します。

### 2) ランナーグループ等でアクセスを絞る

- 組織/リポジトリ単位で runner の利用範囲を制限し、意図しないワークフローから実行されないようにします。
- 可能なら“一時的/エフェメラル” な実行環境（都度クリーン）に寄せます。

---

## リポジトリ/組織のガバナンス設定

### 1) 使える actions / reusable workflows を制限する

- リポジトリ設定で GitHub Actions を無効化/制限できます。
- 可能なら「GitHub作成」「検証済み作成者」「許可リスト」などの方針を採用します。

### 2) 成果物・ログ・キャッシュの保持/上限を運用に合わせる

- artifact/log の保持期間
- cache の保持期間・サイズ

は、必要十分に（長すぎる保持はコスト/漏えい面積が増えます）。

---

## よくある落とし穴と回避策

- `secrets` を fork PR に期待して動かない → fork向けには secrets不要な検証だけに分離する
- `secrets: inherit` を多用して過剰に渡る → “必要なSecretsだけ列挙” へ
- キャッシュに `.npmrc` や資格情報が混ざる → キャッシュ対象パスを最小化し、機密パスを除外
- `concurrency.group` が衝突して別ブランチが巻き込まれる → group命名規則を固定し、必要に応じて ref を含める
- 再利用ワークフローに環境シークレットを渡せると思い込む → 呼び出し設計を見直す（環境は呼び出し先で参照）

---

## 最小テンプレ例

```yaml
name: CI

on:
  pull_request:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: npm ci
      - run: npm test
```
