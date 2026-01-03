# Cloud Workflows HTTPエンドポイント作成のベストプラクティス

最終更新日: 2026年1月3日

## 目次

1. [概要](#概要)
2. [HTTPエンドポイント実装](#httpエンドポイント実装)
3. [セキュリティ](#セキュリティ)
4. [エラーハンドリング](#エラーハンドリング)
5. [リトライポリシー](#リトライポリシー)
6. [タイムアウト設定](#タイムアウト設定)
7. [ロギングとモニタリング](#ロギングとモニタリング)
8. [パフォーマンス最適化](#パフォーマンス最適化)
9. [開発・運用のベストプラクティス](#開発運用のベストプラクティス)

---

## 概要

このドキュメントは、Google Cloud Workflows を使用して HTTP エンドポイントを作成する際のベストプラクティスをまとめたものです。Workflows は、サービスのオーケストレーションやマイクロサービス間の調整に最適なサーバーレスプラットフォームです。

### Workflows の主な特徴

- **サーバーレス実行**: インフラ管理不要で、必要なときだけ実行される
- **統合性**: Google Cloud サービスとの緊密な統合
- **スケーラビリティ**: 自動的にスケールアップ・ダウン
- **コスト効率**: 実行ステップ数に基づく課金

---

## HTTPエンドポイント実装

### 基本的なHTTPリクエストの構文

Cloud Workflows では、HTTP リクエストを行う際に `http.get`、`http.post`、`http.put`、`http.delete`、`http.patch` などのショートカットメソッド、または汎用的な `http.request` を使用できます。

```yaml
- call_external_api:
    call: http.get
    args:
      url: https://api.example.com/data
      headers:
        Content-Type: application/json
      query:
        param1: value1
        param2: value2
      timeout: 60
    result: api_response
```

### HTTPリクエストの重要なパラメータ

#### 1. URL指定

**ベストプラクティス**: URL のハードコーディングを避ける

```yaml
# 悪い例
- call_api:
    call: http.get
    args:
      url: https://prod-api.example.com/data

# 良い例 - 環境変数を使用
- call_api:
    call: http.get
    args:
      url: ${sys.get_env("API_BASE_URL") + "/data"}
```

**推奨アプローチ**:
- **環境変数**: ワークフローの環境変数を使用して、デプロイ先の環境に応じて動的に設定
- **ランタイム引数**: 実行時にパラメータとして URL を渡す
- **Secret Manager**: URL を Secret Manager に安全に保存し、取得する

#### 2. ヘッダー設定

```yaml
- call_with_headers:
    call: http.post
    args:
      url: https://api.example.com/data
      headers:
        Content-Type: application/json
        Authorization: Bearer ${token}
        User-Agent: MyWorkflow/1.0
      body:
        key: value
```

**重要な注意点**:
- `Content-Type` ヘッダーがサポートするメディアタイプ:
  - `application/json` または `application/*+json` - マッピング形式
  - `application/x-www-form-urlencoded` - エンコードされていない文字列
  - `text/*` - 文字列形式
- `User-Agent` ヘッダーのデフォルト値は `GoogleCloudWorkflows; (+https://cloud.google.com/workflows/docs)` で、カスタム値を指定すると追加されます

#### 3. リクエストボディ

```yaml
- post_data:
    call: http.post
    args:
      url: https://api.example.com/items
      body:
        name: "サンプル商品"
        price: 1000
        tags:
          - tag1
          - tag2
    result: create_response
```

**注意**: `Content-Type` ヘッダーが指定されていない場合、本文は自動的に JSON エンコードされ、ヘッダーは `Content-Type: application/json; charset=utf-8` に設定されます。

### レスポンスデータへのアクセス

```yaml
- process_response:
    assign:
      - status_code: ${api_response.code}
      - response_body: ${api_response.body}
      - header_value: ${api_response.headers["Content-Type"]}
      - specific_field: ${api_response.body.data.items[0].name}
```

**レスポンスの構造**:
- `code`: HTTP ステータスコード
- `body`: レスポンスの本文（JSON の場合は自動的にマップに変換）
- `headers`: レスポンスヘッダー

### Workflows コネクタの活用

Google Cloud サービスを呼び出す場合は、HTTP リクエストではなく Workflows コネクタを使用することを強く推奨します。

**コネクタの利点**:
- リクエストのフォーマット処理が簡素化される
- 再試行と長時間実行オペレーションを自動的に処理
- Google Cloud API の詳細を認識する必要がない

```yaml
# コネクタを使用した例
- create_vm:
    call: googleapis.compute.v1.instances.insert
    args:
      project: ${project_id}
      zone: ${zone}
      body:
        name: ${instance_name}
        machineType: ${machine_type}
```

---

## セキュリティ

### 認証と認可

#### 1. サービスアカウントの設定

**ベストプラクティス**:
- **最小権限の原則**: ワークフローに必要な最小限の権限のみを付与
- **デフォルトサービスアカウントを使用しない**: 編集者の基本ロールが付与されているため、カスタムサービスアカウントを作成

```bash
# カスタムサービスアカウントの作成
gcloud iam service-accounts create workflow-sa \
    --display-name="Workflow Service Account"

# 必要な権限のみを付与
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:workflow-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/cloudscheduler.admin"
```

#### 2. OAuth2 認証（Google Cloud API向け）

Google Cloud API（`.googleapis.com` で終わるホスト名）への認証には OAuth2 を使用します。

```yaml
- call_gcp_api:
    call: http.post
    args:
      url: https://compute.googleapis.com/compute/v1/projects/myproject/zones/us-central1-a/instances/myvm/stop
      auth:
        type: OAuth2
        scopes: https://www.googleapis.com/auth/cloud-platform
```

**スコープの指定**:
- 文字列: `"https://www.googleapis.com/auth/cloud-platform"`
- リスト: `["scope1", "scope2", "scope3"]`
- スペース区切り: `"scope1 scope2 scope3"`
- カンマ区切り: `"scope1,scope2,scope3"`

#### 3. OIDC 認証（Cloud Run / Cloud Functions向け）

Cloud Run や Cloud Functions を呼び出す場合は OIDC を使用します。

```yaml
- call_cloud_function:
    call: http.post
    args:
      url: https://us-central1-myproject.cloudfunctions.net/myfunction
      auth:
        type: OIDC
        audience: https://us-central1-myproject.cloudfunctions.net/myfunction
      body:
        message: "Hello from Workflows"
```

**注意**:
- `audience` はオプションで、デフォルトでは `url` と同じ値
- サービスのルート URL に設定する必要がある

#### 4. 外部 API への認証

サードパーティ API の場合は、`Authorization` ヘッダーを使用します。

```yaml
- call_external_api:
    call: http.get
    args:
      url: https://api.external.com/data
      headers:
        Authorization: Bearer ${secret_token}
```

### 機密データの保護

#### Secret Manager の活用

API キー、パスワード、証明書などの機密データは Secret Manager に保存します。

```yaml
# Secret Manager から値を取得
- get_api_key:
    call: googleapis.secretmanager.v1.projects.secrets.versions.accessString
    args:
      secret_id: api-key
    result: api_key

- call_api_with_secret:
    call: http.get
    args:
      url: https://api.example.com/data
      headers:
        X-API-Key: ${api_key}
```

**ベストプラクティス**:
- 機密データをワークフロー定義に直接記述しない
- Secret Manager のバージョニング機能を活用
- 定期的にシークレットをローテーション

#### 顧客管理の暗号鍵（CMEK）

独自の暗号鍵でワークフローデータを保護する場合は CMEK を使用します。

```bash
gcloud workflows deploy my-workflow \
    --source=workflow.yaml \
    --kms-key=projects/PROJECT_ID/locations/LOCATION/keyRings/KEY_RING/cryptoKeys/KEY
```

#### VPC Service Controls

データ漏洩リスクを軽減するために、サービス境界を設定します。

```yaml
# サービス境界内のリソースへのアクセス
- call_protected_service:
    call: http.get
    args:
      url: https://protected-service.example.com/data
      private_service_name: projects/PROJECT_ID/locations/LOCATION/namespaces/NAMESPACE/services/SERVICE
```

---

## エラーハンドリング

### try/except 構文の基本

```yaml
- safe_api_call:
    try:
      call: http.get
      args:
        url: https://api.example.com/data
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
    return: "Resource not found (404)"

- forbidden:
    return: "Access forbidden (403)"
```

### エラーマップの構造

HTTP エラーの場合、エラーマップには以下の属性が含まれます:

- `tags`: エラータグ（例: `["HttpError"]`）
- `message`: 人間が読めるエラーメッセージ
- `code`: HTTP ステータスコード
- `headers`: レスポンスヘッダー
- `body`: レスポンスボディ

### カスタムエラーの発生

```yaml
- validate_input:
    switch:
      - condition: ${input.value < 0}
        raise:
          code: 400
          message: "Value must be positive"
          tags: ["ValidationError"]
```

### 複数ステップでの共通エラーハンドリング

```yaml
- multi_step_operation:
    try:
      steps:
        - step_a:
            call: http.get
            args:
              url: https://api1.example.com/data
            result: response_a
        - step_b:
            call: http.post
            args:
              url: https://api2.example.com/process
              body: ${response_a.body}
            result: response_b
    except:
      as: e
      steps:
        - log_error:
            call: sys.log
            args:
              severity: ERROR
              text: ${e.message}
        - handle_error:
            # エラー処理ロジック
            return: "Operation failed"
```

---

## リトライポリシー

### デフォルトリトライポリシー

Workflows は冪等（idempotent）および非冪等（non-idempotent）ステップ用のデフォルトリトライポリシーを提供しています。

#### 冪等ステップ用（安全に繰り返せるステップ）

```yaml
- idempotent_call:
    try:
      call: http.get
      args:
        url: https://api.example.com/data
    retry: ${http.default_retry}
```

**デフォルト設定**:
```yaml
predicate: ${http.default_retry_predicate}
max_retries: 5
backoff:
  initial_delay: 1
  max_delay: 60
  multiplier: 1.25
```

**リトライ対象**: HTTP ステータスコード `429`, `502`, `503`, `504`、接続エラー、タイムアウトエラー

#### 非冪等ステップ用（繰り返すと問題が生じる可能性があるステップ）

```yaml
- non_idempotent_call:
    try:
      call: http.post
      args:
        url: https://api.example.com/create
        body:
          item: "new item"
    retry: ${http.default_retry_non_idempotent}
```

**リトライ対象**: HTTP ステータスコード `429`, `503`、接続失敗

### カスタムリトライポリシー

#### 基本的なカスタムリトライ

```yaml
- custom_retry_call:
    try:
      call: http.get
      args:
        url: https://api.example.com/data
      result: api_response
    retry:
      predicate: ${http.default_retry_predicate}
      max_retries: 10
      backoff:
        initial_delay: 2
        max_delay: 120
        multiplier: 2
```

**バックオフパラメータ**:
- `initial_delay`: 最初のリトライまでの遅延時間（秒）
- `max_delay`: 最大遅延時間（秒）
- `multiplier`: 遅延時間の倍率

**遅延時間の計算例**（multiplier=2の場合）:
- 1回目: 2秒
- 2回目: 4秒
- 3回目: 8秒
- 4回目: 16秒
- 5回目: 32秒
- 6回目: 64秒
- 7回目: 120秒（max_delay）
- 8回目以降: 120秒

#### カスタムプレディケート（条件付きリトライ）

特定のHTTPステータスコードに対してのみリトライを実行する場合:

```yaml
main:
  steps:
    - api_call_with_custom_retry:
        try:
          call: http.get
          args:
            url: https://api.example.com/data
          result: api_response
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
    - log_error:
        call: sys.log
        args:
          data: ${e}
          severity: INFO
    - check_retry_condition:
        switch:
          - condition: ${e.code == 500}
            return: true
          - condition: ${e.code == 503}
            return: true
    - no_retry:
        return: false
```

### Saga パターンでの分散トランザクション管理

複数のサービスにまたがるトランザクションでは、Saga パターンを使用してデータ整合性を管理します。

```yaml
main:
  steps:
    - reserve_inventory:
        try:
          call: http.post
          args:
            url: ${inventory_service}/reserve
            body:
              item_id: ${item_id}
              quantity: ${quantity}
          result: reservation
        retry: ${http.default_retry}
        except:
          as: e
          steps:
            - log_inventory_failure:
                call: sys.log
                args:
                  severity: ERROR
                  text: "Inventory reservation failed"
            - fail:
                raise: ${e}

    - charge_payment:
        try:
          call: http.post
          args:
            url: ${payment_service}/charge
            body:
              amount: ${amount}
              payment_method: ${payment_method}
          result: payment
        retry: ${http.default_retry}
        except:
          as: e
          steps:
            - compensate_inventory:
                call: http.post
                args:
                  url: ${inventory_service}/release
                  body:
                    reservation_id: ${reservation.body.id}
            - fail:
                raise: ${e}

    - confirm_order:
        return: "Order confirmed"
```

---

## タイムアウト設定

### HTTPリクエストのタイムアウト

```yaml
- api_call_with_timeout:
    call: http.get
    args:
      url: https://api.example.com/long-operation
      timeout: 300  # 最大1,800秒（30分）
    result: response
```

**ベストプラクティス**:
- API の性質に応じて適切なタイムアウトを設定
- 長時間実行される操作の場合、コールバックやポーリングの使用を検討
- タイムアウトエラーに対するリトライポリシーを設定

### ポーリングによる長時間実行オペレーションの待機

```yaml
- start_long_operation:
    call: http.post
    args:
      url: https://api.example.com/start-job
    result: job_info

- poll_job_status:
    steps:
      - initialize:
          assign:
            - job_id: ${job_info.body.job_id}
            - max_attempts: 60
            - attempt: 0
            - completed: false

      - poll_loop:
          for:
            value: attempt
            range: ${[0, max_attempts]}
            steps:
              - check_status:
                  call: http.get
                  args:
                    url: ${"https://api.example.com/jobs/" + job_id}
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

### コールバックの使用

ポーリングの代わりにコールバックを使用すると、より効率的に待機できます。

```yaml
- create_callback:
    call: events.create_callback_endpoint
    args:
      http_callback_method: POST
    result: callback_details

- start_async_operation:
    call: http.post
    args:
      url: https://api.example.com/async-operation
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

---

## ロギングとモニタリング

### Cloud Logging への実行ログ送信

#### 基本的なロギング設定

ワークフローのデプロイ時にロギングレベルを設定します。

```bash
gcloud workflows deploy my-workflow \
    --source=workflow.yaml \
    --call-log-level=log-all-calls
```

**ログレベルオプション**:
- `log-all-calls`: すべての HTTP 呼び出しとレスポンスをログに記録（推奨）
- `log-errors-only`: エラーのみをログに記録
- `none`: 呼び出しログを記録しない

#### カスタムログの出力

```yaml
- log_custom_message:
    call: sys.log
    args:
      text: "Processing user request"
      severity: INFO
      json:
        user_id: ${user_id}
        action: "data_processing"
        timestamp: ${sys.now()}
```

**重要度レベル**:
- `DEBUG`: デバッグ情報
- `INFO`: 一般的な情報
- `NOTICE`: 重要な正常イベント
- `WARNING`: 警告
- `ERROR`: エラー
- `CRITICAL`: 重大なエラー
- `ALERT`: 即座の対応が必要
- `EMERGENCY`: システム使用不可

#### 構造化ログの活用

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
          items: ${items}
        metadata:
          workflow_execution_id: ${sys.get_env("GOOGLE_CLOUD_WORKFLOW_EXECUTION_ID")}
          timestamp: ${sys.now()}
```

### Cloud Monitoring との統合

#### メトリクスの確認

Cloud Monitoring で以下のメトリクスを監視できます:

- **実行数**: `workflows.googleapis.com/finished_execution_count`
- **実行時間**: `workflows.googleapis.com/execution_duration`
- **失敗率**: `workflows.googleapis.com/failed_execution_count`

#### アラートの設定

```yaml
# Terraform を使用したアラートポリシーの設定例
resource "google_monitoring_alert_policy" "workflow_failures" {
  display_name = "Workflow Execution Failures"
  combiner     = "OR"

  conditions {
    display_name = "Failure rate > 10%"

    condition_threshold {
      filter          = "metric.type=\"workflows.googleapis.com/failed_execution_count\" resource.type=\"workflows.googleapis.com/Workflow\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 0.1
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

### 監査ログの活用

#### ワークフローの変更履歴の追跡

```bash
# 監査ログの確認
gcloud logging read \
    'protoPayload.serviceName="workflows.googleapis.com" AND
     protoPayload.methodName:"workflows.update"' \
    --limit 10 \
    --format json
```

#### 実行履歴の確認

```yaml
# ワークフロー内から実行IDを取得
- get_execution_info:
    assign:
      - execution_id: ${sys.get_env("GOOGLE_CLOUD_WORKFLOW_EXECUTION_ID")}
      - workflow_id: ${sys.get_env("GOOGLE_CLOUD_WORKFLOW_ID")}
      - revision_id: ${sys.get_env("GOOGLE_CLOUD_WORKFLOW_REVISION_ID")}

- log_execution_info:
    call: sys.log
    args:
      json:
        execution_id: ${execution_id}
        workflow_id: ${workflow_id}
        revision_id: ${revision_id}
```

---

## パフォーマンス最適化

### 並列実行

独立したステップは並列実行することで、ワークフローの実行時間を大幅に短縮できます。

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
                    url: https://api1.example.com/data
                  result: api1_response
              - store_result1:
                  assign:
                    - results[0]: ${api1_response.body}

        - branch2:
            steps:
              - call_api2:
                  call: http.get
                  args:
                    url: https://api2.example.com/data
                  result: api2_response
              - store_result2:
                  assign:
                    - results[1]: ${api2_response.body}

        - branch3:
            steps:
              - call_api3:
                  call: http.get
                  args:
                    url: https://api3.example.com/data
                  result: api3_response
              - store_result3:
                  assign:
                    - results[2]: ${api3_response.body}

- process_results:
    return: ${results}
```

**注意点**:
- 並列ブランチ間でデータを共有する場合は `shared` フィールドを使用
- すべてのブランチが完了するまで次のステップには進まない
- 並列度の上限はプロジェクトのクォータに依存

### メモリ使用量の最適化

#### 必要なデータのみを保存

```yaml
# 悪い例 - すべてのレスポンスを保存
- get_large_data:
    call: http.get
    args:
      url: https://api.example.com/large-dataset
    result: all_data  # 大きなデータセット全体を保存

# 良い例 - 必要な部分のみを抽出
- get_filtered_data:
    call: http.get
    args:
      url: https://api.example.com/large-dataset
    result: response

- extract_needed_data:
    assign:
      - needed_items: ${response.body.items[0:10]}  # 最初の10件のみ
      - response: null  # 不要になった大きなオブジェクトをクリア
```

#### 変数のクリア

```yaml
- process_batch:
    for:
      value: batch
      in: ${large_dataset}
      steps:
        - process_item:
            call: http.post
            args:
              url: https://api.example.com/process
              body: ${batch}

        # 処理が終わったら変数をクリア
        - clear_batch:
            assign:
              - batch: null
```

### サブワークフローの活用

コードの再利用と保守性を向上させるために、サブワークフローを使用します。

```yaml
main:
  steps:
    - process_users:
        for:
          value: user
          in: ${users}
          steps:
            - validate_and_process:
                call: validate_user
                args:
                  user: ${user}
                result: validated_user
            - store_user:
                call: store_validated_user
                args:
                  user: ${validated_user}

validate_user:
  params: [user]
  steps:
    - check_email:
        switch:
          - condition: ${not("@" in user.email)}
            raise: "Invalid email format"

    - check_age:
        switch:
          - condition: ${user.age < 18}
            raise: "User must be 18 or older"

    - return_valid:
        return: ${user}

store_validated_user:
  params: [user]
  steps:
    - save_to_database:
        call: http.post
        args:
          url: https://api.example.com/users
          body: ${user}
    - return_success:
        return: "User stored successfully"
```

### コストの最適化

#### 内部ステップとして課金されるドメインを使用

```yaml
# 良い例 - 内部ステップとして課金（コスト削減）
- call_cloud_run:
    call: http.get
    args:
      url: https://myservice-xyz.run.app/data

# 悪い例 - 外部ステップとして課金（コスト高）
- call_custom_domain:
    call: http.get
    args:
      url: https://api.mycustomdomain.com/data
```

**内部ステップとして課金されるドメイン**:
- `*.appspot.com`
- `*.cloud.goog`
- `*.cloudfunctions.net`
- `*.run.app`

#### カスタムポーリングポリシー

長時間実行オペレーションを待機する場合、効率的なポーリング間隔を設定します。

```yaml
- call_long_running_operation:
    call: googleapis.compute.v1.globalOperations.wait
    args:
      project: ${project_id}
      operation: ${operation_id}
      connector_params:
        polling_policy:
          initial_delay: 60
          multiplier: 1.5
          max_delay: 900
```

#### sys.log の適切な使用

```yaml
# 過度なログ出力は避ける
- process_items:
    for:
      value: item
      in: ${items}
      steps:
        - process:
            # 各アイテムでログを出力すると高コスト
            # call: sys.log  # 避ける
            call: process_item_subworkflow
            args:
              item: ${item}

# 代わりに、重要なイベントのみログに記録
- log_summary:
    call: sys.log
    args:
      json:
        total_processed: ${len(items)}
        status: "completed"
```

---

## 開発・運用のベストプラクティス

### Infrastructure as Code（IaC）の活用

#### Terraform を使用したワークフローのデプロイ

```hcl
# main.tf
resource "google_workflows_workflow" "api_orchestrator" {
  name            = "api-orchestrator"
  region          = "us-central1"
  description     = "Orchestrates multiple API calls"
  service_account = google_service_account.workflow_sa.email

  source_contents = templatefile("${path.module}/workflow.yaml", {
    api_base_url    = var.api_base_url
    timeout_seconds = var.timeout_seconds
  })

  labels = {
    environment = var.environment
    team        = "backend"
  }
}

resource "google_service_account" "workflow_sa" {
  account_id   = "workflow-sa"
  display_name = "Workflow Service Account"
}

resource "google_project_iam_member" "workflow_invoker" {
  project = var.project_id
  role    = "roles/workflows.invoker"
  member  = "serviceAccount:${google_service_account.workflow_sa.email}"
}
```

```yaml
# workflow.yaml (テンプレート)
main:
  steps:
    - call_api:
        call: http.get
        args:
          url: ${api_base_url}/data
          timeout: ${timeout_seconds}
```

#### 環境変数の使用

```bash
# 環境ごとに異なる設定を使用
gcloud workflows deploy api-orchestrator \
    --source=workflow.yaml \
    --service-account=workflow-sa@PROJECT_ID.iam.gserviceaccount.com \
    --set-env-vars=ENVIRONMENT=production,API_BASE_URL=https://api.prod.example.com
```

### CI/CD パイプラインの構築

#### Cloud Build を使用した自動デプロイ

```yaml
# cloudbuild.yaml
steps:
  # ワークフロー構文の検証
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'workflows'
      - 'deploy'
      - '--source=workflow.yaml'
      - '--dry-run'
      - 'my-workflow'

  # テスト環境へのデプロイ
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'workflows'
      - 'deploy'
      - '--source=workflow.yaml'
      - '--service-account=workflow-sa@${PROJECT_ID}.iam.gserviceaccount.com'
      - '--set-env-vars=ENVIRONMENT=staging'
      - 'my-workflow-staging'

  # 統合テストの実行
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'workflows'
      - 'run'
      - 'my-workflow-staging'
      - '--data={"test_mode": true}'

  # 本番環境へのデプロイ（手動承認後）
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'workflows'
      - 'deploy'
      - '--source=workflow.yaml'
      - '--service-account=workflow-sa@${PROJECT_ID}.iam.gserviceaccount.com'
      - '--set-env-vars=ENVIRONMENT=production'
      - 'my-workflow'
    waitFor: ['-']  # 手動承認を待つ

options:
  logging: CLOUD_LOGGING_ONLY
```

#### Git リポジトリからのデプロイ

```bash
# Cloud Build トリガーの作成
gcloud builds triggers create github \
    --repo-name=my-workflows \
    --repo-owner=myorg \
    --branch-pattern="^main$" \
    --build-config=cloudbuild.yaml
```

### バージョン管理とロールバック

```bash
# 特定のリビジョンへのロールバック
gcloud workflows revisions list my-workflow

gcloud workflows deploy my-workflow \
    --source=workflow-v1.yaml
```

### デバッグとトラブルシューティング

#### ステップ実行履歴の確認

```bash
# 実行の詳細を確認
gcloud workflows executions describe EXECUTION_ID \
    --workflow=my-workflow \
    --location=us-central1

# ステップごとの実行履歴を表示
gcloud workflows executions describe EXECUTION_ID \
    --workflow=my-workflow \
    --location=us-central1 \
    --format="value(steps)"
```

#### ログを使用したデバッグ

```yaml
- debug_variables:
    call: sys.log
    args:
      severity: DEBUG
      json:
        step: "debug_checkpoint_1"
        variables:
          user_id: ${user_id}
          request_data: ${request_data}
          current_time: ${sys.now()}
```

### テストストラテジー

#### ユニットテストの実装

サブワークフローを個別にテストします。

```yaml
# test_workflow.yaml
main:
  params: [test_case]
  steps:
    - run_test:
        switch:
          - condition: ${test_case == "validate_user"}
            call: test_validate_user
          - condition: ${test_case == "process_payment"}
            call: test_process_payment

test_validate_user:
  steps:
    - test_valid_email:
        call: validate_user
        args:
          user:
            email: "test@example.com"
            age: 25
        result: valid_result

    - assert_valid:
        switch:
          - condition: ${valid_result.email == "test@example.com"}
            next: test_invalid_email
        raise: "Validation test failed"

    - test_invalid_email:
        try:
          call: validate_user
          args:
            user:
              email: "invalid-email"
              age: 25
        except:
          as: e
          steps:
            - assert_error:
                switch:
                  - condition: ${e.message == "Invalid email format"}
                    return: "Test passed"
                raise: "Expected validation error"

validate_user:
  # サブワークフローの実装
  ...
```

#### 統合テストの実行

```bash
# テストデータを使用してワークフローを実行
gcloud workflows run my-workflow \
    --data='{"test_mode": true, "user_id": "test-user-123"}' \
    --location=us-central1
```

### スケジューリング

```yaml
# Cloud Scheduler を使用した定期実行
resource "google_cloud_scheduler_job" "workflow_schedule" {
  name        = "daily-data-sync"
  description = "Run data sync workflow daily at 2am"
  schedule    = "0 2 * * *"
  time_zone   = "Asia/Tokyo"

  http_target {
    http_method = "POST"
    uri         = "https://workflowexecutions.googleapis.com/v1/projects/${var.project_id}/locations/us-central1/workflows/data-sync/executions"

    oauth_token {
      service_account_email = google_service_account.scheduler_sa.email
    }

    body = base64encode(jsonencode({
      argument = jsonencode({
        sync_date = "$${timestamp}"
      })
    }))
  }
}
```

### ドキュメンテーション

ワークフロー内にコメントを含めて、保守性を向上させます。

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
          - condition: ${not("data_type" in input)}
            raise: "Missing required field: data_type"

    # ステップ2: データ取得
    # タイムアウト: 60秒
    # リトライ: 最大3回（指数バックオフ）
    - fetch_data:
        try:
          call: http.get
          args:
            url: ${sys.get_env("API_BASE_URL") + "/data"}
            query:
              user_id: ${input.user_id}
              type: ${input.data_type}
            timeout: 60
          result: raw_data
        retry:
          max_retries: 3
          backoff:
            initial_delay: 2
            max_delay: 30
            multiplier: 2

    # ステップ3: データ変換と保存
    - process_and_store:
        call: process_data_subworkflow
        args:
          data: ${raw_data.body}
        result: processed_result

    - return_result:
        return: ${processed_result}
```

---

## セキュリティチェックリスト

HTTPエンドポイント実装時の包括的なセキュリティチェックリストです。

### デプロイ前チェック

- [ ] 専用サービスアカウントを作成し、最小権限のみを付与
- [ ] デフォルトサービスアカウントを使用していないことを確認
- [ ] すべての機密データ（API キー、パスワード、トークン）を Secret Manager に保存
- [ ] ワークフロー定義にハードコードされた認証情報がないことを確認
- [ ] 環境変数または Secret Manager を使用して URL を設定
- [ ] 適切な認証方式（OAuth2 または OIDC）を設定
- [ ] タイムアウト値を適切に設定（デフォルトは使用しない）
- [ ] エラーハンドリングが実装されていることを確認
- [ ] リトライポリシーが適切に設定されていることを確認
- [ ] ロギングレベルを設定（推奨: `log-all-calls`）

### 運用中の継続的チェック

- [ ] 定期的にアクセスキーとシークレットをローテーション
- [ ] サービスアカウントの権限を定期的に監査
- [ ] 実行ログを定期的に確認し、異常なパターンを検出
- [ ] セキュリティパッチとアップデートを適用
- [ ] Cloud Monitoring でアラートを設定
- [ ] インシデント対応計画を策定・更新
- [ ] 定期的なペネトレーションテストの実施
- [ ] コンプライアンス要件の遵守を確認

### セキュリティインシデント対応

万が一セキュリティインシデントが発生した場合の対応手順:

1. **検知と初動対応**
   - Cloud Logging でインシデントの範囲を特定
   - 影響を受けたワークフローを一時停止
   - 関係者に通知

2. **封じ込め**
   - 侵害されたサービスアカウントのアクセス取り消し
   - 影響を受けたシークレットをローテーション
   - ネットワークアクセスを制限

3. **根本原因分析**
   - 監査ログを詳細に分析
   - 侵入経路を特定
   - 再発防止策を策定

4. **回復と改善**
   - セキュアな構成でワークフローを再デプロイ
   - セキュリティ対策を強化
   - インシデント報告書を作成

---

## まとめ

このドキュメントでは、Cloud Workflows で HTTP エンドポイントを作成する際のベストプラクティスを包括的にカバーしました。

### 重要なポイント

1. **セキュリティ第一**: 最小権限の原則、Secret Manager の活用、適切な認証方式の選択
2. **エラーハンドリングとリトライ**: 適切な例外処理とリトライポリシーで信頼性を向上
3. **パフォーマンス最適化**: 並列実行、メモリ管理、コスト効率的な実装
4. **可観測性**: ログとモニタリングによる運用の透明性確保
5. **IaC とCI/CD**: 自動化による一貫性のあるデプロイメント
6. **継続的な改善**: 定期的なセキュリティ監査とパフォーマンスレビュー

### 参考リンク

- [Cloud Workflows 公式ドキュメント](https://docs.cloud.google.com/workflows/docs)
- [Workflows ベストプラクティス](https://docs.cloud.google.com/workflows/docs/best-practice)
- [セキュリティベストプラクティス](https://docs.cloud.google.com/workflows/docs/security-best-practices)
- [GitHub Workflows Demos](https://github.com/GoogleCloudPlatform/workflows-demos)

---

**最終更新**: 2026年1月3日
**バージョン**: 1.0
