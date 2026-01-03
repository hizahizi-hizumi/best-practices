---
description: 'Google Cloud Workflows YAML ファイル作成時のベストプラクティスとセキュリティガイドライン'
applyTo: '**/*.yaml, **/*.yml'
---

# Cloud Workflows ベストプラクティス

Cloud Workflows で HTTP エンドポイントを作成する際の実践的なガイドラインです。セキュリティ、エラーハンドリング、パフォーマンス最適化を重視した実装を行ってください。

## 全般的な指針

- すべてのワークフローに適切なエラーハンドリングとリトライポリシーを実装する
- 機密データは Secret Manager を使用し、ハードコーディングしない
- 最小権限の原則に従い、専用サービスアカウントを使用する
- ロギングを有効化し、構造化ログを活用する
- 並列実行可能なステップは parallel ブロックで実装する

## HTTPリクエストの実装

### URL の指定

URL をハードコーディングせず、環境変数または Secret Manager を使用してください。

**良い例**:
```yaml
- call_api:
    call: http.get
    args:
      url: ${sys.get_env("API_BASE_URL") + "/data"}
      timeout: 60
```

**悪い例**:
```yaml
- call_api:
    call: http.get
    args:
      url: https://prod-api.example.com/data
```

### タイムアウトの設定

すべての HTTP リクエストに明示的なタイムアウトを設定してください（最大1,800秒）。

```yaml
- api_call_with_timeout:
    call: http.get
    args:
      url: ${api_url}
      timeout: 300
```

### 認証の実装

#### Google Cloud API（OAuth2）

```yaml
- call_gcp_api:
    call: http.post
    args:
      url: https://compute.googleapis.com/compute/v1/projects/myproject/zones/us-central1-a/instances/myvm/stop
      auth:
        type: OAuth2
        scopes: https://www.googleapis.com/auth/cloud-platform
```

#### Cloud Run / Cloud Functions（OIDC）

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

#### 外部 API（Bearer Token）

Secret Manager からトークンを取得して使用してください。

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

### try/except 構文

すべての外部呼び出しには try/except を実装してください。

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

### エラーログの記録

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

### 冪等操作（GET リクエストなど）

```yaml
- idempotent_call:
    try:
      call: http.get
      args:
        url: ${api_url}
    retry: ${http.default_retry}
```

### 非冪等操作（POST/PUT リクエストなど）

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

特定のエラーコードに対してのみリトライを行う場合:

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

## ロギング

### 構造化ログの活用

重要なイベントには構造化ログを使用してください。

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

- `DEBUG`: デバッグ情報
- `INFO`: 一般的な情報（推奨）
- `WARNING`: 警告
- `ERROR`: エラー
- `CRITICAL`: 重大なエラー

## パフォーマンス最適化

### 並列実行

独立したステップは並列実行してください。

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

### メモリ使用量の最適化

不要な大きなオブジェクトはクリアしてください。

```yaml
- get_filtered_data:
    call: http.get
    args:
      url: ${api_url}
    result: response

- extract_needed_data:
    assign:
      - needed_items: ${response.body.items[0:10]}
      - response: null  # 不要になった大きなオブジェクトをクリア
```

### サブワークフローの活用

再利用可能なロジックはサブワークフローに分離してください。

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

### Secret Manager の使用

機密データは必ず Secret Manager に保存してください。

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

ワークフローの入力は必ず検証してください。

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

- デフォルトサービスアカウントを使用しない
- 最小権限の原則に従う
- 専用のサービスアカウントを作成する

## 長時間実行オペレーション

### ポーリングパターン

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

### 内部ステップとして課金されるドメインを使用

以下のドメインは内部ステップとして課金されます（コスト削減）:
- `*.appspot.com`
- `*.cloud.goog`
- `*.cloudfunctions.net`
- `*.run.app`

```yaml
# 良い例 - 内部ステップ
- call_cloud_run:
    call: http.get
    args:
      url: https://myservice-xyz.run.app/data
```

### 過度なログ出力を避ける

ループ内での sys.log 呼び出しは避け、サマリーを記録してください。

```yaml
# 悪い例
- process_items:
    for:
      value: item
      in: ${items}
      steps:
        - log_each_item:
            call: sys.log  # コスト高
            args:
              json: ${item}

# 良い例
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

## ドキュメンテーション

ワークフロー内にコメントを含めて、保守性を向上させてください。

```yaml
# データ処理ワークフロー
# 目的: 外部APIからデータを取得し、処理して保存する
# 所有者: Backend Team
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

## デプロイメント

### ロギングレベルの設定

```bash
gcloud workflows deploy my-workflow \
    --source=workflow.yaml \
    --call-log-level=log-all-calls \
    --service-account=workflow-sa@PROJECT_ID.iam.gserviceaccount.com
```

### 環境変数の設定

```bash
gcloud workflows deploy my-workflow \
    --source=workflow.yaml \
    --set-env-vars=ENVIRONMENT=production,API_BASE_URL=https://api.prod.example.com
```

## チェックリスト

デプロイ前に以下を確認してください:

- [ ] すべての HTTP リクエストにタイムアウトが設定されている
- [ ] 適切なエラーハンドリングとリトライポリシーが実装されている
- [ ] 機密データは Secret Manager に保存されている
- [ ] URL がハードコーディングされていない
- [ ] 専用サービスアカウントを使用している
- [ ] 構造化ログを実装している
- [ ] 並列実行可能なステップは並列化されている
- [ ] 入力検証が実装されている
- [ ] ワークフローにコメントが含まれている

## 参考リンク

- [Cloud Workflows 公式ドキュメント](https://docs.cloud.google.com/workflows/docs)
- [セキュリティベストプラクティス](https://docs.cloud.google.com/workflows/docs/security-best-practices)
