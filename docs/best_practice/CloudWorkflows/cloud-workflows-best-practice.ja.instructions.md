---
description: 'セキュリティとエラーハンドリングを備えたGoogle Cloud Workflows YAMLファイルのベストプラクティス'
applyTo: '**/*.yaml, **/*.yml'
---

# Cloud Workflowsベストプラクティス

Cloud Workflowsで安全で信頼性の高いHTTPエンドポイントを作成するためのガイドライン。

## 目的と適用範囲

Cloud Workflows YAMLファイルにベストプラクティスを適用し、セキュリティ、エラーハンドリング、パフォーマンス最適化を確保する。

## 核となる原則

- すべての外部呼び出しにtry/exceptとリトライポリシーを実装する
- 機密データはSecret Managerに保存し、決してハードコードしない
- 最小限の必要な権限を持つ専用サービスアカウントを使用する
- すべての重要なイベントに対して構造化ログを有効にする
- `parallel`ブロックで独立したステップを並列化する

**根拠**: これらの原則は、ランタイム障害を防ぎ、認証情報を保護し、実行時間を最適化する

## HTTPリクエスト

### URL設定

URLには環境変数またはSecret Managerを使用し、決してハードコードしない。

**推奨**:
```yaml
- call_api:
    call: http.get
    args:
      url: ${sys.get_env("API_BASE_URL") + "/data"}
      timeout: 60
```

**非推奨**:
```yaml
- call_api:
    call: http.get
    args:
      url: https://prod-api.example.com/data
```

**根拠**: ハードコードされたURLは環境間でのデプロイを妨げ、内部インフラストラクチャを露出させる。

### タイムアウト設定

すべてのHTTPリクエストに明示的なタイムアウトを設定する（最大1,800秒）。

**根拠**: 応答のないエンドポイントでワークフローが無期限にハングするのを防ぐ。

```yaml
- api_call_with_timeout:
    call: http.get
    args:
      url: ${api_url}
      timeout: 300
```

### 認証

#### Google Cloud API (OAuth2)

Google Cloud API呼び出しにはOAuth2認証を使用する。

```yaml
- call_gcp_api:
    call: http.post
    args:
      url: https://compute.googleapis.com/compute/v1/projects/myproject/zones/us-central1-a/instances/myvm/stop
      auth:
        type: OAuth2
        scopes: https://www.googleapis.com/auth/cloud-platform
```

#### Cloud Run / Cloud Functions (OIDC)

Cloud RunとCloud FunctionsにはOIDC認証を使用する。

```yaml
- call_cloud_function:
    call: http.post
    args:
      url: https://us-central1-myproject.cloudfunctions.net/myfunction
      auth:
        type: OIDC
      body:
        message: "Hello from Workflows"
```

#### 外部API (Bearerトークン)

外部API認証にはSecret Managerからトークンを取得する。

**根拠**: バージョン管理とログでのトークン露出を防ぐ。

```yaml
- get_api_key:
    call: googleapis.secretmanager.v1.projects.secrets.versions.accessString
    args:
      secret_id: api-key
    result: api_key

- call_external_api:
    call: http.get
    args:
      url: https://api.example.com/data
      headers:
        Authorization: Bearer ${api_key}
```

## エラーハンドリング

### Try/Exceptブロック

すべての外部呼び出しをtry/exceptブロックでラップする。

**根拠**: 適切なハンドリングなしでワークフローの失敗が伝播するのを防ぐ。

```yaml
- safe_api_call:
    try:
      call: http.get
      args:
        url: ${api_url}
      result: api_response
    except:
      as: e
      steps:
        - handle_error:
            switch:
              - condition: ${not("HttpError" in e.tags)}
                next: connection_problem
              - condition: ${e.code == 404}
                next: not_found
              - condition: ${e.code == 403}
                next: forbidden
        - unhandled:
            raise: ${e}

- success:
    return: ${api_response.body}

- connection_problem:
    return: "Connection error occurred"

- not_found:
    return: "Resource not found"

- forbidden:
    return: "Access forbidden"
```

### エラーログ

タイムスタンプとコンテキストを含む構造化JSON形式ですべてのエラーをログに記録する。

```yaml
- log_error:
    call: sys.log
    args:
      severity: ERROR
      json:
        error_message: ${e.message}
        error_code: ${e.code}
        step: "api_call_failed"
        timestamp: ${sys.now()}
```

## リトライポリシー

### べき等操作（GETリクエスト）

べき等操作には`http.default_retry`を使用する。

**根拠**: GETリクエストは副作用なしにリトライしても安全である。

```yaml
- idempotent_call:
    try:
      call: http.get
      args:
        url: ${api_url}
    retry: ${http.default_retry}
```

### 非べき等操作（POST/PUTリクエスト）

副作用のある操作には`http.default_retry_non_idempotent`を使用する。

**根拠**: リソースの重複作成やデータ破損を防ぐ。

```yaml
- non_idempotent_call:
    try:
      call: http.post
      args:
        url: ${api_url}
        body: ${data}
    retry: ${http.default_retry_non_idempotent}
```

### カスタムリトライポリシー

特定のエラーコードのみをリトライするカスタム述語を定義する。

**根拠**: 永続的な失敗（4xxエラー）のリトライを避けながら、一時的な問題（5xxエラー）を処理する。

```yaml
main:
  steps:
    - api_call_with_custom_retry:
        try:
          call: http.get
          args:
            url: ${api_url}
          result: response
        retry:
          predicate: ${custom_retry_predicate}
          max_retries: 5
          backoff:
            initial_delay: 2
            max_delay: 60
            multiplier: 2

custom_retry_predicate:
  params: [e]
  steps:
    - check_retry_condition:
        switch:
          - condition: ${e.code == 500}
            return: true
          - condition: ${e.code == 503}
            return: true
    - no_retry:
        return: false
```

## ログ記録

### 構造化ログ

すべての重要なイベントに対して構造化JSONログを使用する。

**根拠**: Cloud Loggingでの効率的なクエリとモニタリングを可能にする。

```yaml
- log_structured_data:
    call: sys.log
    args:
      severity: INFO
      json:
        event: "order_created"
        order:
          id: ${order_id}
          amount: ${amount}
        metadata:
          workflow_execution_id: ${sys.get_env("GOOGLE_CLOUD_WORKFLOW_EXECUTION_ID")}
          timestamp: ${sys.now()}
```

### ログレベル

適切な重要度レベルを使用する:
- `DEBUG`: デバッグ情報
- `INFO`: 一般情報（通常フローに推奨）
- `WARNING`: 警告状態
- `ERROR`: エラー状態
- `CRITICAL`: 即座の対応が必要な重大なエラー

## パフォーマンス最適化

### 並列実行

`parallel`ブロックを使用して独立したステップを並列実行する。

**根拠**: ステップ間に依存関係がない場合、全体のワークフロー実行時間を短縮する。

```yaml
- parallel_api_calls:
    parallel:
      shared: [results]
      branches:
        - branch1:
            steps:
              - call_api1:
                  call: http.get
                  args:
                    url: ${api1_url}
                  result: response1
              - store_result1:
                  assign:
                    - results[0]: ${response1.body}

        - branch2:
            steps:
              - call_api2:
                  call: http.get
                  args:
                    url: ${api2_url}
                  result: response2
              - store_result2:
                  assign:
                    - results[1]: ${response2.body}
```

### メモリ最適化

必要なデータを抽出した後、`null`を割り当てて大きなオブジェクトをクリアする。

**根拠**: 長時間実行されるワークフローでのメモリ枯渇を防ぐ。

```yaml
- get_filtered_data:
    call: http.get
    args:
      url: ${api_url}
    result: response

- extract_needed_data:
    assign:
      - needed_items: ${response.body.items[0:10]}
      - response: null  # もう必要ない大きなオブジェクトをクリア
```

### サブワークフロー

再利用可能なロジックをサブワークフローに抽出する。

**根拠**: コードの再利用性とテスト可能性を向上させる。

```yaml
main:
  steps:
    - process_user:
        call: validate_and_process_user
        args:
          user: ${user}
        result: processed_user

validate_and_process_user:
  params: [user]
  steps:
    - validate:
        switch:
          - condition: ${not("@" in user.email)}
            raise: "Invalid email format"
    - return_valid:
        return: ${user}
```

## セキュリティ

### Secret Manager

すべての機密データをSecret Managerに保存し、決してワークフローファイルに含めない。

**根拠**: バージョン管理とログでの認証情報の露出を防ぐ。

```yaml
- get_credentials:
    call: googleapis.secretmanager.v1.projects.secrets.versions.accessString
    args:
      secret_id: database-password
    result: db_password

- connect_database:
    call: http.post
    args:
      url: ${database_url}
      headers:
        Authorization: ${db_password}
```

### 入力検証

処理前にすべてのワークフロー入力を検証する。

**根拠**: 不正な形式の入力によるランタイムエラーとセキュリティ脆弱性を防ぐ。

```yaml
main:
  params: [input]
  steps:
    - validate_input:
        switch:
          - condition: ${not("user_id" in input)}
            raise: "Missing required field: user_id"
          - condition: ${not("data_type" in input)}
            raise: "Missing required field: data_type"
```

### サービスアカウントの権限

最小権限の原則に従う:
- 各ワークフローに専用のサービスアカウントを作成する
- デフォルトのCompute Engineサービスアカウントは決して使用しない
- 必要な権限のみを付与する

**根拠**: 認証情報が侵害された場合の影響範囲を制限する。

## 長時間実行される操作

### ポーリングパターン

長時間実行される操作には指数バックオフを使用したポーリングを使用する。

**根拠**: 応答性とリソース効率のバランスを取る。

```yaml
- start_long_operation:
    call: http.post
    args:
      url: ${api_url}/start-job
    result: job_info

- poll_job_status:
    steps:
      - initialize:
          assign:
            - job_id: ${job_info.body.job_id}
            - max_attempts: 60
            - completed: false

      - poll_loop:
          for:
            value: attempt
            range: ${[0, max_attempts]}
            steps:
              - check_status:
                  call: http.get
                  args:
                    url: ${api_url + "/jobs/" + job_id}
                  result: status_response

              - evaluate_status:
                  switch:
                    - condition: ${status_response.body.status == "COMPLETED"}
                      steps:
                        - mark_complete:
                            assign:
                              - completed: true
                        - exit_loop:
                            next: success

                    - condition: ${status_response.body.status == "FAILED"}
                      raise: "Job failed"

              - wait:
                  call: sys.sleep
                  args:
                    seconds: 30

- success:
    return: ${status_response.body}
```

### コールバックパターン

イベント駆動の長時間実行操作にはコールバックエンドポイントを使用する。

**根拠**: 完了時に通知できる操作に対して、ポーリングよりも効率的である。

```yaml
- create_callback:
    call: events.create_callback_endpoint
    args:
      http_callback_method: POST
    result: callback_details

- start_async_operation:
    call: http.post
    args:
      url: ${api_url}/async-operation
      body:
        callback_url: ${callback_details.url}

- await_callback:
    call: events.await_callback
    args:
      callback: ${callback_details}
      timeout: 3600
    result: callback_request

- process_result:
    return: ${callback_request.http_request.body}
```

## コスト最適化

### 内部ステップドメイン

コストを削減するためにGoogle Cloudドメインを使用する（内部ステップとして請求される）:
- `*.appspot.com`
- `*.cloud.goog`
- `*.cloudfunctions.net`
- `*.run.app`

**推奨**:
```yaml
- call_cloud_run:
    call: http.get
    args:
      url: https://myservice-xyz.run.app/data
```

**根拠**: 内部ステップは外部HTTP呼び出しよりも低い料金で請求される。

### ログのオーバーヘッドを最小化する

ループ内でのログ記録を避け、代わりに要約をログに記録する。

**非推奨**:
```yaml
- process_items:
    for:
      value: item
      in: ${items}
      steps:
        - log_each_item:
            call: sys.log  # 高コスト
            args:
              json: ${item}
```

**推奨**:
```yaml
- process_items:
    for:
      value: item
      in: ${items}
      steps:
        - process_item:
            call: process_item_subworkflow
            args:
              item: ${item}

- log_summary:
    call: sys.log
    args:
      json:
        total_processed: ${len(items)}
        status: "completed"
```

## ドキュメント

目的、所有者、ステップの説明をワークフローにコメントとして含める。

**根拠**: チームメンバーと将来の変更に対する保守性を向上させる。

```yaml
# データ処理ワークフロー
# 目的: 外部APIからデータを取得、処理、保存する
# 所有者: バックエンドチーム
# 最終更新: 2026-01-03

main:
  params: [input]
  steps:
    # ステップ1: 入力検証
    # 必須フィールド: user_id, data_type
    - validate_input:
        switch:
          - condition: ${not("user_id" in input)}
            raise: "Missing required field: user_id"
```

## デプロイ

### 呼び出しログレベル

開発とトラブルシューティングには`--call-log-level=log-all-calls`を設定する。

```bash
gcloud workflows deploy my-workflow \
    --source=workflow.yaml \
    --call-log-level=log-all-calls \
    --service-account=workflow-sa@PROJECT_ID.iam.gserviceaccount.com
```

### 環境変数

`--set-env-vars`を使用してデプロイ時に環境変数を設定する。

```bash
gcloud workflows deploy my-workflow \
    --source=workflow.yaml \
    --set-env-vars=ENVIRONMENT=production,API_BASE_URL=https://api.prod.example.com
```

## デプロイ前チェックリスト

デプロイ前に以下を確認する:

- [ ] すべてのHTTPリクエストに明示的なタイムアウトがある
- [ ] エラーハンドリングとリトライポリシーが実装されている
- [ ] 機密データがSecret Managerに保存されている
- [ ] ハードコードされたURLや認証情報がない
- [ ] 専用のサービスアカウントが設定されている
- [ ] 構造化ログが実装されている
- [ ] 独立したステップが並列化されている
- [ ] 入力検証が実装されている
- [ ] ワークフローにドキュメントコメントが含まれている

## 参考資料

- [Cloud Workflowsドキュメント](https://docs.cloud.google.com/workflows/docs)
- [セキュリティベストプラクティス](https://docs.cloud.google.com/workflows/docs/security-best-practices)
- [HTTPコールバックの例](https://docs.cloud.google.com/workflows/docs/creating-callback-endpoints)
