---
description: '安全で保守可能な GitHub Actions ワークフローとコンポジットアクションのベストプラクティス'
applyTo: '**/.github/workflows/**/*.yml, **/.github/workflows/**/*.yaml, **/.github/actions/**/action.yml, **/.github/actions/**/action.yaml, **/action.yml, **/action.yaml'
---

# GitHub Actions ベストプラクティス

安全で保守可能、かつ効率的な GitHub Actions ワークフローとコンポジットアクションを作成するためのガイドラインである。

## 目的と適用範囲

GitHub Actions ワークフローとコンポジットアクションにセキュリティファーストの原則を適用する。
すべての権限において最小権限の原則を優先する。
アクションをコミット SHA にピン留めすることでサプライチェーンの完全性を保護する。
環境変数を通じてシークレットを保護する。
キャッシングとコンカレンシー制御によりワークフローの実行を最適化する。

## 適用対象ファイル

ワークフロー: `.github/workflows/*.yml` および `.github/workflows/*.yaml`
コンポジットアクション: `action.yml` および `action.yaml`

## セキュリティ (優先度 1)

### 権限

すべてのワークフローファイルでワークフローレベルの明示的な `permissions` を定義する。
デフォルト権限として `contents: read` を設定する。
特定の操作には、ジョブレベルでのみ書き込み権限を付与する。
デフォルトの暗黙的な権限に依存してはならない。
**理由**: 明示的な権限により、権限昇格を防ぎ、侵害されたワークフローの影響範囲を制限する。

```yaml
# 推奨
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

# 非推奨: permissions が欠落している (暗黙的に広範な権限)
jobs:
  deploy:
    runs-on: ubuntu-latest
```

### サプライチェーンセキュリティ

すべての外部アクションを完全なコミット SHA (40 文字) にピン留めする。
再利用可能なワークフローをコミット SHA にピン留めする。
`@main`、`@v1`、または任意の可変タグ参照を避ける。
新しいバージョンをピン留めする前にアクションのソースコードをレビューする。
**理由**: SHA ピン留めにより、アクションメンテナーが可変参照に悪意のあるコードをプッシュするサプライチェーン攻撃を防止する。

```yaml
# 推奨
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

# 非推奨
- uses: some-owner/some-action@main
```

### スクリプトインジェクション防止

信頼できない入力を `run:` スクリプト内で直接連結してはならない。
すべての GitHub コンテキスト値を信頼できないものとして扱う: `github.event.pull_request.title`、`github.event.issue.body`、`github.head_ref`。
信頼できない値は `env:` ブロックのみを通じて渡す。
シェル内で環境変数を引用符で囲む: `$VAR` ではなく `"$VAR"`。
`echo "$VAR"` の代わりに `printf '%s\n' "$VAR"` を使用する。
**理由**: 直接補間により、攻撃者が PR タイトル、ブランチ名、または Issue 本文を制御する場合にコマンドインジェクションが可能になる。

```yaml
# 推奨
- name: Safe print
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"

# 非推奨
- name: Unsafe
  run: |
    echo "Title: ${{ github.event.pull_request.title }}"
```

## 認証

### OIDC (OpenID Connect)

静的な認証情報の代わりに、AWS、GCP、Azure の認証に OIDC を使用する。
OIDC 使用時には `permissions: id-token: write` を設定する。
リポジトリとブランチによるトークン使用を制限するようクラウドプロバイダーの信頼ポリシーを構成する。
クラウド認証情報を GitHub シークレットとして保存することを避ける。
**理由**: OIDC トークンは短命で自動的にローテーションされ、静的認証情報の漏洩リスクを排除する。

```yaml
permissions:
  contents: read
  id-token: write
```

## シークレット管理

ステップまたはジョブレベルの `env:` を通じてシークレットを渡す。
コマンドライン引数またはスクリプトパラメータとしてシークレットを渡してはならない。
`echo "::add-mask::$SECRET"` でシークレットを早期にマスクする。
シークレットが利用できない場合に正常にスキップまたは失敗するようワークフローを設計する。
`if:` 条件でシークレットを使用することを避ける (ワークフローログに表示される)。
フォーク PR でシークレットが利用できない場合にワークフローが正しく動作することをテストする。
**理由**: コマンドライン引数はログとプロセスリストに表示されるが、環境変数は保護されたままである。

```yaml
# 推奨
- name: Use secret
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: |
    ./tool --token "$API_TOKEN"

# 非推奨
- name: Unsafe secret
  run: |
    ./tool --token "${{ secrets.API_TOKEN }}"
```

## 環境

すべてのデプロイジョブに `environment` フィールドを定義する。
リポジトリ設定でプロダクション環境に手動承認を要求する。
環境保護ルールにブランチ制限を適用する。
環境アクセスを制限するためにデプロイメント保護ルールを設定する。
再利用可能なワークフローでは `environment` コンテキストが利用できないことを文書化する。
**理由**: 環境保護ルールは、重要な操作に対する人間の承認ゲートと監査証跡を追加する。

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - run: echo "deploy"
```

## ワークフロー構造

### 命名と整理

`name:` フィールドを使用して、明確で説明的なタイトルでワークフローに名前を付ける。
ワークフローファイル名にはケバブケースを使用する: `ci-tests.yml`、`deploy-production.yml`。
関連するジョブを単一のワークフローファイルにグループ化する。
無関係な懸念事項を別々のワークフローファイルに分割する。

### トリガー

デフォルトに依存するのではなく、明示的なイベントトリガーを指定する。
CI チェックには `pull_request` を使用し、`push` は使用しない (重複実行を避けるため)。
`push` トリガーを特定のブランチに制限する: `branches: [main, develop]`。
手動トリガーを有効にするために `workflow_dispatch` を使用する。
**理由**: 明示的なトリガーにより、予期しないワークフロー実行を防ぎ、リソース効率を向上させる。

```yaml
# 推奨
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

# 非推奨
on: [push]  # すべてのブランチでトリガーされる
```

### ジョブの依存関係

`needs:` を使用してジョブの依存関係を明示的に定義する。
`needs:` を省略することで独立したジョブを並列実行する。
論理的なステージでジョブを整理する: test → build → deploy。

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

## コンカレンシー制御

共有状態を変更するすべてのワークフローに `concurrency` グループを定義する。
PR とプッシュの両方をサポートするために `github.head_ref || github.ref` パターンを使用する。
CI とテストワークフローには `cancel-in-progress: true` を設定する。
デプロイメントワークフローには `cancel-in-progress: false` を設定する。
一意性のためにコンカレンシーグループにワークフロー名を含める。
**理由**: コンカレンシー制御により、デプロイメントでの競合状態を防ぎ、無駄な CI 実行を減らす。

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

## キャッシング

ワークフロー実行を高速化するために依存関係ディレクトリをキャッシュする。
ロックファイルのハッシュを主キャッシュキーとして使用する: `${{ hashFiles('**/package-lock.json') }}`。
段階的に特異性の低いパターンで `restore-keys` を定義する。
センシティブなパスを除外する: 認証ファイル、認証情報、API キー。
ネイティブ依存関係を持つリポジトリでは `node_modules` のキャッシュを避ける。
過度なストレージ使用を防ぐためにキャッシュサイズ制限を設定する。
**理由**: ロックファイルのハッシュにより再現可能なビルドを保証し、シークレットを除外することでキャッシュポイズニングによる認証情報漏洩を防止する。

```yaml
# 推奨
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# 非推奨: センシティブなパスのキャッシュ
- uses: actions/cache@v4
  with:
    path: ~/.npmrc  # 認証トークンを含む
```

## ジョブ構成

### ランナーの選択

特定のバージョンが必要でない限り、Linux ジョブには `ubuntu-latest` を使用する。
再現性のために正確なランナーバージョンを指定する: `ubuntu-22.04`。
複数の OS またはバージョンでテストするためにマトリックス戦略を使用する。
パブリックリポジトリでは、セルフホストランナーよりも GitHub ホストランナーを優先する。
**理由**: GitHub ホストランナーは、信頼できないコードに対して隔離とセキュリティを提供する。

### タイムアウト

暴走プロセスを防ぐために、すべてのジョブに `timeout-minutes` を設定する。
高速なジョブには短いタイムアウトを使用する: `timeout-minutes: 5`。
長時間ジョブには現実的なタイムアウトを設定する: `timeout-minutes: 60`。
**理由**: タイムアウトにより、無限ループやスタックしたプロセスによる請求問題を防止する。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
```

### 条件付き実行

ジョブまたはステップを条件付きでスキップするために `if:` を使用する。
イベントタイプをチェックする: `if: github.event_name == 'push'`。
ブランチをチェックする: `if: github.ref == 'refs/heads/main'`。
`if:` 条件でシークレットを使用することを避ける (ログに表示される)。

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
```

## 再利用可能なワークフロー

ジョブレベルで `uses:` を使用して再利用可能なワークフローを呼び出す。
再利用可能なワークフローをコミット SHA にピン留めする。
`with:` ブロックを通じて入力を渡す。
`secrets:` ブロックを通じてシークレットを明示的に渡す。
信頼できる組織のワークフローを除き、`secrets: inherit` を避ける。
ワークフローファイルで必要な入力とシークレットを文書化する。
**理由**: 明示的な入力とシークレットの宣言は、最小権限の原則に従い、保守性を向上させる。

```yaml
# 推奨
jobs:
  call:
    uses: org/repo/.github/workflows/reusable.yml@b4ffde65f46336ab88eb53be808477a3936bae11
    with:
      config-path: .github/labeler.yml
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}

# 非推奨
jobs:
  call:
    uses: org/repo/.github/workflows/reusable.yml@main  # 可変参照
    secrets: inherit  # すべてのシークレットを渡す
```

## セルフホストランナー

パブリックリポジトリではセルフホストランナーを避ける。
信頼できる貢献者のいるプライベートリポジトリにのみセルフホストランナーを使用する。
ランナーグループとリポジトリ権限でランナーアクセスを制限する。
エフェメラル環境 (コンテナ、VM) でセルフホストランナーを実行する。
ランナーソフトウェアとホスト OS を定期的に更新する。
内部ネットワークとセンシティブなリソースからランナーを隔離する。
**理由**: 信頼できないコードを実行するセルフホストランナーは、インフラストラクチャを侵害し、シークレットを流出させる可能性がある。

## マトリックス戦略

複数のバージョンまたは構成でテストするためにマトリックス戦略を使用する。
値の配列を使用して `strategy.matrix` でマトリックスを定義する。
ワークフローコストを削減するためにマトリックスの組み合わせを制限する。
特定の組み合わせを追加するために `include` を使用する。
不要な組み合わせを削除するために `exclude` を使用する。

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20]
    exclude:
      - os: macos-latest
        node: 18
```

## パフォーマンス最適化

### チェックアウトの最小化

シャロークローンのために `actions/checkout` を `fetch-depth: 1` で使用する。
必要なディレクトリのみをクローンするために `sparse-checkout` を使用する。

### 並列実行

`needs:` 依存関係なしで独立したジョブを並列実行する。
長いテストスイートを並列ジョブに分割する。
作業を分散するためにマトリックス戦略を使用する。

### アーティファクト管理

必要なアーティファクトのみをアップロードする。
アーティファクト保持期間を設定する: `retention-days: 7`。
アップロード前にアーティファクトを圧縮する。
不必要に大きなバイナリをアップロードすることを避ける。

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: results/
    retention-days: 7
```

### 出力とロギング

ログ出力をグループ化するために `echo "::group::Title"` を使用する。
グループ化で冗長な出力を折りたたむ。
重要なメッセージには `echo "::notice::Message"` を使用する。
警告には `echo "::warning::Message"` を使用する。
エラーには `echo "::error::Message"` を使用する。

## 一般的なアンチパターン

### 権限の欠落

**問題**: `permissions` が指定されておらず、広範なデフォルト権限が付与される。
**解決策**: ワークフローレベルで明示的な `permissions: contents: read` を追加する。

### 可変アクション参照

**問題**: アクションに `@main`、`@v1`、またはブランチ名を使用する。
**解決策**: コミット SHA にピン留めする: `actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11`。

### スクリプトインジェクション

**問題**: `run: echo "${{ github.event.pull_request.title }}"`。
**解決策**: env を通じて渡す: `env: TITLE: ${{ github.event.pull_request.title }}`、その後 `run: printf '%s\n' "$TITLE"`。

### シークレットの露出

**問題**: `run: deploy.sh ${{ secrets.API_KEY }}`。
**解決策**: `env: API_KEY: ${{ secrets.API_KEY }}`、その後 `run: deploy.sh "$API_KEY"`。

### キャッシュされた認証情報

**問題**: `~/.npmrc`、`~/.docker/config.json`、または認証情報ファイルのキャッシュ。
**解決策**: キャッシュから認証情報パスを除外し、代わりに OIDC を使用する。

### 非効率なチェックアウト

**問題**: 実行のたびに完全なリポジトリ履歴が取得される。
**解決策**: シャロークローンのために `fetch-depth: 1` を使用する。

### タイムアウトの欠落

**問題**: プロセスがハングした場合、ジョブが無期限に実行される。
**解決策**: すべてのジョブに `timeout-minutes: 15` を追加する。

### 過度に広範なトリガー

**問題**: `on: push` がすべてのブランチとコミットでトリガーされる。
**解決策**: ブランチを指定する: `on: push: branches: [main]`。

## 参考資料

- [GitHub Actions セキュリティベストプラクティス](https://docs.github.com/ja/actions/security-guides/security-hardening-for-github-actions)
- [GITHUB_TOKEN 権限](https://docs.github.com/ja/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)
- [Actions での OpenID Connect](https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
