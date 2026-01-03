# Cloud Functions HTTPエンドポイント ベストプラクティス

Cloud FunctionsでHTTPエンドポイントを作成する際の推奨事項と実装ガイドです。

> **Note:** Cloud Functions (2nd gen) は Cloud Run インフラストラクチャ上で動作します。
> 本ドキュメントは2025年時点のベストプラクティスをまとめたものです。

## 📚 目次

- [基本原則](#基本原則)
- [HTTPエンドポイント設計](#httpエンドポイント設計)
- [セキュリティ](#セキュリティ)
- [パフォーマンス最適化](#パフォーマンス最適化)
- [エラーハンドリング](#エラーハンドリング)
- [ロギングとモニタリング](#ロギングとモニタリング)
- [開発とテスト](#開発とテスト)
- [デプロイと運用](#デプロイと運用)
- [コストの最適化](#コストの最適化)

---

## 基本原則

### 冪等性（Idempotency）

**重要:** 関数は複数回呼び出されても同じ結果を生成する必要があります。

**理由:**
- 失敗した呼び出しの再試行を可能にする
- ネットワーク障害時の安全な再実行
- イベント駆動型関数での重複イベント処理

**実装例:**
```javascript
// 良い例: 冪等性のある実装
functions.http('createUser', async (req, res) => {
  const userId = req.body.userId;

  // 既存チェック
  const existingUser = await db.getUser(userId);
  if (existingUser) {
    return res.status(200).json(existingUser);
  }

  // 新規作成
  const user = await db.createUser(userId, req.body);
  res.status(201).json(user);
});
```

### HTTPレスポンスの確実な送信

**必須:** HTTP関数は必ずHTTPレスポンスを送信する必要があります。

**問題点:**
- レスポンスを送信しないとタイムアウトまで実行される
- タイムアウト時間全体に対して課金される
- 予測不能な動作やコールドスタートの原因となる

**実装例:**
```javascript
// 良い例
functions.http('helloHttp', (req, res) => {
  const name = req.query.name || req.body.name || 'World';
  res.send(`Hello ${escapeHtml(name)}!`);
});

// 悪い例 - レスポンスを送信していない
functions.http('badExample', (req, res) => {
  console.log('Processing...');
  // res.send()を呼び出していない - タイムアウトまで実行される
});
```

---

## HTTPエンドポイント設計

### Functions Frameworkの利用

**推奨事項:**
- Functions Frameworkを使用して関数を記述する
- HTTPハンドラ関数は明確な署名タイプを指定する
- エントリポイントは一貫性のある命名規則に従う

**Node.js実装例:**
```javascript
import { http } from '@google-cloud/functions-framework';

http('myHttpFunction', (req, res) => {
  // リクエスト処理
  // レスポンス送信前にすべての非同期処理を完了させる
  res.send('OK');
});
```

**重要ポイント:**
- HTTPレスポンス送信前にすべてのバックグラウンドタスクを完了させる
- 未完了のタスクは未定義の動作につながる可能性がある

### HTTPリクエスト/レスポンス処理

**ベストプラクティス:**
- リクエストメソッドを適切に検査し、メソッドごとに異なるアクションを実行
- レスポンスストリーミングが必要な場合は `Transfer-Encoding: chunked` ヘッダーを使用
- HTTPからHTTPSへの自動リダイレクトを考慮した実装

**注意点:**
- TLSはウェブサービス到達前に終了される
- 混合コンテンツ警告を避けるため、すべてのURIに `https` プロトコルを使用
- `X-Forwarded-Proto` HTTPヘッダーを活用

### CORS対応

**実装例:**
```javascript
functions.http('corsEnabledFunction', (req, res) => {
  // プリフライトリクエスト用のCORSヘッダー設定
  res.set('Access-Control-Allow-Origin', '*');

  if (req.method === 'OPTIONS') {
    // OPTIONSリクエストへのレスポンス
    res.set('Access-Control-Allow-Methods', 'GET');
    res.set('Access-Control-Allow-Headers', 'Content-Type');
    res.set('Access-Control-Max-Age', '3600');
    res.status(204).send('');
  } else {
    res.send('Hello World!');
  }
});
```

**CORS制限への対処:**
- 公開アクセスを許可する
- Identity-Aware Proxyを構成する
- Firebase Hostingとの統合を検討

---

## セキュリティ

### アーキテクチャとセキュリティ設計

**セキュリティレイヤー:**
- Google Front End (GFE): TLS終端とDoS攻撃保護
- HTTP プロキシ: リクエストのロードバランシング
- 実行環境: gVisor (第1世代) または Linux microVM (第2世代) によるサンドボックス化

**推奨事項:**
- 第2世代実行環境の使用を検討（互換性とパフォーマンス向上）
- seccompシステムコールフィルタリングによる保護を活用

### 認証と認可

**サービス間認証:**
```javascript
const {GoogleAuth} = require('google-auth-library');
const auth = new GoogleAuth();

async function request() {
  const client = await auth.getIdTokenClient(targetAudience);
  const res = await client.fetch(url);
  console.info(res.data);
}
```

**IAM権限設定:**
- サービスごとにユーザー管理サービスアカウントを使用
- 最小権限の原則に従って権限を付与
- `run.invoker` ロールを適切に設定

**認証方法:**
1. 認証ライブラリの使用（推奨）
2. メタデータサーバーの使用
3. Workload Identity連携の活用
4. サービスアカウントキー（最終手段）

### ネットワークセキュリティ

**上り（内向き）トラフィック:**
- デフォルトの `run.app` URLはHTTPS必須
- Cloud Armorで追加のセキュリティ機能を適用可能
- IAPとSSLポリシーの活用

**下り（外向き）トラフィック:**
- VPCファイアウォールルールの適用
- VPC Flow Logsでトラフィック検査
- 静的アウトバウンドIPアドレスの設定（必要に応じて）

### データ保護

**推奨事項:**
- ステートフルデータは外部ストレージサービスに保存（Cloud SQL、Memorystore等）
- 機密データは環境変数やSecret Managerで管理
- Binary Authorizationで信頼できるイメージのみデプロイ

---

## パフォーマンス最適化

### 起動時間の最適化

**コンテナ起動の高速化:**
- 最小コンテナイメージをビルド
- 依存関係の数とサイズを最小限に抑える
- 起動時のCPUブーストを有効化

**グローバル変数の活用:**
```javascript
// グローバルスコープ（インスタンス起動時に1回実行）
const instanceVar = heavyComputation();

functions.http('scopeDemo', (req, res) => {
  // 関数スコープ（毎回実行）
  const functionVar = lightComputation();
  res.send(`Per instance: ${instanceVar}, per function: ${functionVar}`);
});
```

**遅延初期化:**
```javascript
let lazyGlobal;

functions.http('lazyGlobals', (req, res) => {
  // 必要になったときだけ初期化
  lazyGlobal = lazyGlobal || functionSpecificComputation();
  res.send(`Lazy global: ${lazyGlobal}`);
});
```

### リソース設定

**メモリとCPU:**
- デフォルト: 512 MiB（関数は256 MiB）
- 最大: 32 GiB
- CPU値に応じた適切なメモリ設定が必要

| CPU    | 必要なメモリ |
| ------ | ------------ |
| 1 vCPU | 最大 4 GiB   |
| 2 vCPU | 最大 8 GiB   |
| 4 vCPU | 2～16 GiB    |
| 8 vCPU | 4～32 GiB    |

**計算式:**
```
ピーク時メモリ = 標準メモリ + (リクエストあたりのメモリ × 同時実行数)
```

### 同時実行の最適化

**推奨手順:**
1. コードをプロファイリングして同時実行可能なレベルを把握
2. コードレベルの同時実行設定を構成
3. Cloud Runの同時実行設定をコードレベル以下に設定
4. 負荷テストで動作を確認
5. パフォーマンスに応じて調整を繰り返す

**注意点:**
- 変更可能なグローバル状態を回避
- 必要に応じてロックやミューテックスを使用
- デフォルトの最大同時実行数80は多くのケースに適している

### スケーリング戦略

**最小インスタンスの設定:**
- コールドスタートを削減
- 常時起動インスタンスによる即座のレスポンス
- コストとパフォーマンスのトレードオフを考慮

**インスタンス自動スケーリング:**
- 60% CPU/同時実行しきい値でスケーリング
- アイドルインスタンスは最大15分保持
- リクエストキュー時間: インスタンス起動時間の3.5倍または10秒

---

## エラーハンドリング

### Error Reporting統合

**自動検出されるエラー:**
- サポート言語のスタックトレースを含む例外
- メモリ上限超過エラー
- 利用できるインスタンスがない状況

**ベストプラクティス:**
```javascript
try {
  // 処理
} catch (error) {
  // エラーを適切に処理し、サービスをクラッシュさせない
  console.error('Error occurred:', error);
  res.status(500).send('Internal Server Error');
}
```

**推奨事項:**
- すべての例外を処理してクラッシュを防止
- エラー発生時にコンテナの起動遅延を回避
- Error Reportingで集計されたエラーを定期的に確認

### 一時ファイルの管理

**注意点:**
- メモリ内ファイルシステムがディスクストレージになる
- 削除しないファイルはメモリを継続的に使用
- 最終的にメモリ不足エラーにつながる可能性

**推奨事項:**
- 一時ファイルは使用後に必ず削除
- ファイルシステムへの書き込みを最小限に抑える

---

## ロギングとモニタリング

### 構造化ログの記録

**実装例:**
```javascript
const globalLogFields = {};

// リクエストログとの関連付け
if (typeof req !== 'undefined') {
  const traceHeader = req.header('X-Cloud-Trace-Context');
  if (traceHeader && project) {
    const [trace] = traceHeader.split('/');
    globalLogFields['logging.googleapis.com/trace'] =
      `projects/${project}/traces/${trace}`;
  }
}

// 構造化ログエントリの作成
const entry = Object.assign(
  {
    severity: 'NOTICE',
    message: 'This is the default display field.',
    component: 'arbitrary-property',
  },
  globalLogFields
);

console.log(JSON.stringify(entry));
```

### 特別なJSONフィールド

**自動処理されるフィールド:**
- `severity`: ログの重大度レベル
- `message`: メイン表示テキスト
- `logging.googleapis.com/trace`: リクエストログとの関連付け
- `httpRequest`: HTTPリクエスト情報

### ログの種類

**リクエストログ（サービスのみ）:**
- 自動作成
- リクエストの詳細情報を含む

**コンテナログ:**
- `stdout`、`stderr` から自動キャプチャ
- 構造化JSONまたはテキストメッセージ

### モニタリング指標

**組み込み指標:**
- リクエスト数
- レイテンシ
- エラー率
- CPU/メモリ使用率
- インスタンス数

**推奨事項:**
- Cloud Monitoringで指標を定期的に確認
- アラートポリシーを設定
- Cloud Traceで分散トレーシングを有効化

---

## 開発とテスト

### ローカル開発

**Functions Frameworkの活用:**
```bash
# インストール
npm install @google-cloud/functions-framework

# ローカル起動
npx functions-framework --target=myHttpFunction --port=8080
```

**推奨事項:**
- ローカル環境でテスト実行
- Functions Frameworkを使用した開発サーバー起動
- 本番環境と同じ条件でテスト

### 負荷テスト

**ベストプラクティス:**
- 同時実行を構成できる負荷テストツールを使用
- 予想される負荷で動作が安定していることを確認
- 様々な同時実行設定でテスト
- `container/billable_instance_time` 指標を監視

**テスト項目:**
- 同時リクエスト処理能力
- レスポンスタイム
- エラー率
- リソース使用状況

---

## デプロイと運用

### デプロイ戦略

**段階的ロールアウト:**
- トラフィック分割による段階的デプロイ
- タグを使用したテスト環境
- ロールバック戦略の準備

**デプロイ方法:**
```bash
# gcloud CLIでのデプロイ
gcloud run deploy SERVICE_NAME \
  --source . \
  --region REGION \
  --memory 512Mi \
  --cpu 1
```

### バックグラウンドアクティビティ

**インスタンスベース課金を使用:**
- リクエスト外でのバックグラウンドアクティビティをサポート
- CPU常時アクセス可能

**リクエストベース課金を使用:**
- リクエスト処理終了後はCPUアクセスが制限される
- バックグラウンドスレッドやルーティンを開始しない
- レスポンス送信前にすべての非同期処理を完了

**確認方法:**
- HTTPリクエストエントリの後にログ記録がないか確認

### サービスURLの管理

**決定論的URL:**
```
https://[TAG---]SERVICE_NAME-PROJECT_NUMBER.REGION.run.app
```

**特徴:**
- サービス作成前に予測可能
- サービス間通信に有用
- 63文字以下のDNSセグメントでのみ利用可能

**非決定論的URL:**
```
https://[TAG---]SERVICE_IDENTIFIER.run.app
```

**特徴:**
- ランダムなハッシュベースの識別子
- デプロイ後は永続的
- 完全なURLは事前予測不可

### URL無効化

**推奨事項:**
- ロードバランサ経由のアクセスのみに制限する場合、デフォルトの `run.app` URLを無効化
- 内部サービスの場合、上り（内向き）設定で公開インターネットからのアクセスをブロック

---

## コストの最適化

### 料金モデルの理解

**Cloud Run Functions (2025) の料金体系:**
```
料金 = vCPU-seconds + GiB-seconds + リクエスト数 + ネットワーク（egress）
```

**無料枠（us-central1の例、request-based billing）:**
- 180,000 vCPU-seconds
- 360,000 GiB-seconds
- 2,000,000 リクエスト/月

**課金計算例:**
```
条件:
- 1 vCPU、2 GiB
- 300ms/リクエスト
- 5M リクエスト/月

計算:
- vCPU-seconds: 0.3 × 1 × 5,000,000 = 1,500,000 vCPU-s
- GiB-seconds: 0.3 × 2 × 5,000,000 = 3,000,000 GiB-s
```

### コスト削減のベストプラクティス

**1. 実行時間の最適化:**
- 不要な依存関係を削除してコールドスタートを削減
- 効率的なコードで実行時間を短縮
- グローバル変数を活用してリクエスト間で再利用

**2. メモリとCPU設定:**
- 過剰なリソース割り当てを避ける
- 負荷テストで最適な設定を見つける
- `container/billable_instance_time` 指標を監視

**3. 同時実行の調整:**
```
費用への影響 = インスタンス時間 × 同時実行設定

最適化のポイント:
- 同時実行数を上げる → インスタンス数が減る → コスト削減の可能性
- ただしレイテンシとのトレードオフを考慮
```

**4. リクエストベースvs.インスタンスベース課金:**

| 課金モード         | 適している用途             | 特徴                             |
| ------------------ | -------------------------- | -------------------------------- |
| リクエストベース   | スパイクのあるトラフィック | リクエスト処理中のみ課金         |
| インスタンスベース | 安定したトラフィック       | インスタンスの存続期間全体に課金 |

**5. Serverless VPC Accessコネクタ:**
- 別途課金される
- 必要な場合のみ使用
- コストを事前に確認

---

## まとめ

### 重要なチェックリスト

**基本原則:**
- [ ] 冪等性のある関数を実装
- [ ] 必ずHTTPレスポンスを送信
- [ ] コールドスタート対策を実施

**セキュリティ:**
- [ ] IAM権限を最小権限の原則で設定
- [ ] 認証が必要なエンドポイントで適切な認証を実装
- [ ] 機密データはSecret Managerで管理（環境変数は避ける）
- [ ] CORS設定を適切に構成
- [ ] Identity-Aware Proxy（IAP）の活用を検討

**パフォーマンス:**
- [ ] 起動時間を最適化（最小イメージ、依存関係削減）
- [ ] グローバル変数を適切に活用
- [ ] メモリとCPUを適切に設定
- [ ] 同時実行設定を最適化（デフォルト80、最大1000）
- [ ] 最小インスタンス数を設定してコールドスタート削減

**運用:**
- [ ] 構造化ログを記録
- [ ] Error Reportingで監視
- [ ] 負荷テストを実施
- [ ] デプロイ戦略を計画（段階的ロールアウト）
- [ ] クォータと制限を理解

**コスト:**
- [ ] 実行時間を最適化
- [ ] 適切な課金モード（リクエストベース vs インスタンスベース）を選択
- [ ] 無料枠を活用
- [ ] モニタリング指標で課金対象時間を追跡

**エラーハンドリング:**
- [ ] すべての例外を処理
- [ ] 一時ファイルを適切に削除
- [ ] エラー時にサービスをクラッシュさせない

### Cloud Run Functions vs Cloud Run Services

**Cloud Run Functionsが適している場合:**
- イベント駆動処理（Cloud Storage、Pub/Sub、Audit Logs等）
- WebhooksやライトウェイトなAPI
- GCPサービス間のサービスグルー（接着コード）
- 軽量なHTTPエンドポイント

**Cloud Run Servicesを検討すべき場合:**
- カスタムベースイメージが必要
- 非標準ランタイムの使用
- 長時間実行の処理やバックグラウンドワーカー
- 関数で公開される以上の同時実行/起動動作の詳細制御

### タイムアウト設定

| 種類                         | 最大タイムアウト |
| ---------------------------- | ---------------- |
| HTTPリクエスト               | 最大60分         |
| Eventarc（イベント駆動）関数 | 最大9分          |

### 参考リソース

**公式ドキュメント:**
- [Cloud Run 公式ドキュメント](https://docs.cloud.google.com/run/docs)
- [Functions Framework](https://github.com/GoogleCloudPlatform/functions-framework)
- [Cloud Run Functions 比較](https://docs.cloud.google.com/run/docs/functions/comparison)
- [Cloud Run ベストプラクティス](https://docs.cloud.google.com/run/docs/tips/general)
- [セキュリティ設計の概要](https://docs.cloud.google.com/run/docs/securing/security)

**コミュニティリソース:**
- [Cloud Run functions ガイド（2025年版）](https://cloudchipr.com/blog/google-cloud-functions)
- [クラウドセキュリティベストプラクティス 2024](https://200oksolutions.com/blog/cloud-security-best-practices-2024/)

---

最終更新: 2026年1月3日
バージョン: 1.1
