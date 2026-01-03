---
description: 'Google Cloud FunctionsでHTTPエンドポイントを実装する際のベストプラクティスとセキュリティガイドライン'
applyTo: '**/*.js, **/*.ts, **/main.py'
---

# Cloud Functions HTTP エンドポイント ベストプラクティス

Cloud Functions (2nd gen) で HTTPエンドポイントを実装する際の必須ガイドライン。コード生成、レビュー時には本指示に従うこと。

## プロジェクトコンテキスト

- **対象**: Cloud Functions (2nd gen) - Cloud Run インフラストラクチャ上で動作
- **実行環境**: Node.js 20+, Python 3.11+
- **フレームワーク**: Functions Framework for Node.js/Python
- **更新日**: 2026年1月3日

---

## 基本原則

### ✅ 必須実装事項

#### 1. HTTPレスポンスの確実な送信

**必ず実装**: すべてのコードパスでHTTPレスポンスを送信する

```javascript
// ✅ 良い例
functions.http('myFunction', (req, res) => {
  if (req.method !== 'POST') {
    return res.status(405).send('Method Not Allowed');
  }
  
  try {
    const result = processRequest(req.body);
    res.status(200).json(result);
  } catch (error) {
    console.error('Error:', error);
    res.status(500).send('Internal Server Error');
  }
});

// ❌ 悪い例 - レスポンスを送信していない
functions.http('badFunction', (req, res) => {
  console.log('Processing...');
  // res.send() を呼び出していない - タイムアウトまで実行され課金される
});
```

**理由**:
- レスポンスを送信しないとタイムアウトまで実行され続ける
- タイムアウト時間全体に対して課金される
- 予測不能な動作やコールドスタートの原因となる

#### 2. 冪等性の実装

**必ず実装**: 関数は複数回呼び出されても同じ結果を生成する

```javascript
// ✅ 良い例: 冪等性のある実装
functions.http('createUser', async (req, res) => {
  const userId = req.body.userId;
  
  if (!userId) {
    return res.status(400).json({ error: 'userId is required' });
  }
  
  // 既存チェックで冪等性を確保
  const existingUser = await db.getUser(userId);
  if (existingUser) {
    return res.status(200).json(existingUser);
  }
  
  // 新規作成
  const user = await db.createUser(userId, req.body);
  res.status(201).json(user);
});

// ❌ 悪い例 - 毎回新規作成してしまう
functions.http('badCreateUser', async (req, res) => {
  const user = await db.createUser(req.body.userId, req.body);
  res.status(201).json(user);
});
```

**理由**:
- 失敗した呼び出しの安全な再試行が可能
- ネットワーク障害時の再実行に対応
- イベント駆動型関数での重複イベント処理に必須

#### 3. レスポンス送信前の非同期処理完了

**必ず実装**: HTTPレスポンス送信前にすべての非同期処理を完了させる

```javascript
// ✅ 良い例
functions.http('processData', async (req, res) => {
  try {
    // すべての非同期処理を await で完了させる
    await saveToDatabase(req.body);
    await sendNotification(req.body.email);
    
    // すべて完了後にレスポンス送信
    res.status(200).send('Processing completed');
  } catch (error) {
    console.error('Error:', error);
    res.status(500).send('Error occurred');
  }
});

// ❌ 悪い例 - バックグラウンドタスクが未完了
functions.http('badProcessData', (req, res) => {
  saveToDatabase(req.body); // await していない
  res.status(200).send('Processing started'); // 即座にレスポンス
});
```

---

## HTTPエンドポイント設計

### ✅ Functions Framework の使用

**必ず実装**: Functions Framework の標準パターンに従う

```javascript
// Node.js
import { http } from '@google-cloud/functions-framework';

http('myHttpFunction', (req, res) => {
  // リクエスト処理
  res.send('OK');
});
```

```python
# Python
import functions_framework

@functions_framework.http
def my_http_function(request):
    return 'OK', 200
```

### ✅ HTTPメソッドの適切な処理

**必ず実装**: HTTPメソッドを検査し、適切に処理する

```javascript
// ✅ 良い例
functions.http('apiEndpoint', (req, res) => {
  res.set('Access-Control-Allow-Origin', '*');
  
  if (req.method === 'OPTIONS') {
    // CORS プリフライトリクエスト
    res.set('Access-Control-Allow-Methods', 'GET, POST');
    res.set('Access-Control-Allow-Headers', 'Content-Type');
    res.set('Access-Control-Max-Age', '3600');
    return res.status(204).send('');
  }
  
  switch (req.method) {
    case 'GET':
      return handleGet(req, res);
    case 'POST':
      return handlePost(req, res);
    default:
      return res.status(405).send('Method Not Allowed');
  }
});
```

### ✅ CORS の適切な設定

**必ず実装**: CORS ヘッダーを適切に設定する

```javascript
// ✅ 良い例: 本番環境向け - 特定オリジンのみ許可
functions.http('secureApi', (req, res) => {
  const allowedOrigins = [
    'https://example.com',
    'https://app.example.com'
  ];
  
  const origin = req.headers.origin;
  if (allowedOrigins.includes(origin)) {
    res.set('Access-Control-Allow-Origin', origin);
  }
  
  if (req.method === 'OPTIONS') {
    res.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.set('Access-Control-Max-Age', '3600');
    return res.status(204).send('');
  }
  
  // メインロジック
  res.json({ message: 'Success' });
});

// ❌ 悪い例: すべてのオリジンを許可（開発環境のみ使用可）
functions.http('insecureApi', (req, res) => {
  res.set('Access-Control-Allow-Origin', '*'); // 本番環境では避ける
  res.json({ message: 'Success' });
});
```

---

## セキュリティ実装

### ✅ 認証と認可

**必ず実装**: 公開すべきでないエンドポイントには認証を実装する

```javascript
// ✅ 良い例: ID トークン検証
const { OAuth2Client } = require('google-auth-library');
const client = new OAuth2Client();

functions.http('authenticatedEndpoint', async (req, res) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  const idToken = authHeader.split('Bearer ')[1];
  
  try {
    const ticket = await client.verifyIdToken({
      idToken,
      audience: process.env.CLIENT_ID,
    });
    const payload = ticket.getPayload();
    
    // ユーザー情報を使って処理
    const result = await processRequest(payload);
    res.json(result);
  } catch (error) {
    console.error('Authentication failed:', error);
    res.status(401).json({ error: 'Invalid token' });
  }
});
```

**IAM 権限設定例**:
```bash
# 特定のサービスアカウントのみアクセス許可
gcloud run services add-iam-policy-binding SERVICE_NAME \
  --member='serviceAccount:caller@project.iam.gserviceaccount.com' \
  --role='roles/run.invoker'
```

### ✅ 入力検証とサニタイゼーション

**必ず実装**: すべてのユーザー入力を検証する

```javascript
// ✅ 良い例
const escapeHtml = require('escape-html');

functions.http('userInput', (req, res) => {
  const name = req.body.name;
  const email = req.body.email;
  
  // 入力検証
  if (!name || typeof name !== 'string' || name.length > 100) {
    return res.status(400).json({ error: 'Invalid name' });
  }
  
  if (!email || !isValidEmail(email)) {
    return res.status(400).json({ error: 'Invalid email' });
  }
  
  // HTMLエスケープ
  const safeName = escapeHtml(name);
  
  res.send(`Hello ${safeName}!`);
});

function isValidEmail(email) {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

// ❌ 悪い例 - 検証なし
functions.http('unsafeInput', (req, res) => {
  res.send(`Hello ${req.body.name}!`); // XSS脆弱性
});
```

### ✅ Secret Management

**必ず実装**: 機密情報は Secret Manager を使用する

```javascript
// ✅ 良い例: Secret Manager の使用
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

async function getSecret(secretName) {
  const [version] = await client.accessSecretVersion({
    name: `projects/${process.env.PROJECT_ID}/secrets/${secretName}/versions/latest`,
  });
  return version.payload.data.toString();
}

functions.http('secureFunction', async (req, res) => {
  const apiKey = await getSecret('api-key');
  // API キーを使用
  res.send('OK');
});

// ❌ 悪い例 - コードに直接記述
const API_KEY = 'hardcoded-secret-key'; // 絶対に避ける

// ❌ 悪い例 - 環境変数（機密情報には使用しない）
const API_KEY = process.env.API_KEY; // 平文で保存される
```

---

## パフォーマンス最適化

### ✅ グローバル変数の活用

**推奨**: リクエスト間で再利用可能なオブジェクトはグローバルスコープで初期化

```javascript
// ✅ 良い例: グローバルスコープで初期化（インスタンス起動時に1回実行）
const { Firestore } = require('@google-cloud/firestore');
const db = new Firestore(); // インスタンス起動時に1回だけ実行

functions.http('optimizedFunction', async (req, res) => {
  // dbクライアントを再利用（毎回初期化しない）
  const doc = await db.collection('users').doc(req.body.userId).get();
  res.json(doc.data());
});

// ❌ 悪い例: 関数内で毎回初期化
functions.http('slowFunction', async (req, res) => {
  const db = new Firestore(); // 毎回初期化される - 遅い
  const doc = await db.collection('users').doc(req.body.userId).get();
  res.json(doc.data());
});
```

### ✅ 遅延初期化パターン

**推奨**: 必要になったときだけ初期化する

```javascript
// ✅ 良い例: 遅延初期化
let dbConnection = null;

async function getDbConnection() {
  if (!dbConnection) {
    dbConnection = await initializeDatabase();
  }
  return dbConnection;
}

functions.http('lazyInit', async (req, res) => {
  const db = await getDbConnection();
  const result = await db.query('SELECT * FROM users');
  res.json(result);
});
```

### ✅ 起動時間の最適化

**必ず確認**: 依存関係を最小限に抑える

```javascript
// ✅ 良い例: 必要な機能のみインポート
const { Firestore } = require('@google-cloud/firestore');

// ❌ 悪い例: 未使用のライブラリをインポート
const _ = require('lodash'); // 使用していない
const moment = require('moment'); // 使用していない
const axios = require('axios'); // 使用していない
```

**package.json の最適化**:
```json
{
  "dependencies": {
    "@google-cloud/functions-framework": "^3.0.0",
    "@google-cloud/firestore": "^7.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
```

### ✅ 一時ファイルの管理

**必ず実装**: 一時ファイルは使用後に削除する

```javascript
// ✅ 良い例
const fs = require('fs').promises;
const os = require('os');
const path = require('path');

functions.http('fileProcessing', async (req, res) => {
  const tmpDir = os.tmpdir();
  const tmpFile = path.join(tmpDir, `temp-${Date.now()}.txt`);
  
  try {
    await fs.writeFile(tmpFile, req.body.content);
    const processed = await processFile(tmpFile);
    res.json(processed);
  } finally {
    // 必ず削除する
    try {
      await fs.unlink(tmpFile);
    } catch (error) {
      console.error('Failed to delete temp file:', error);
    }
  }
});

// ❌ 悪い例 - 削除していない
functions.http('fileLeaking', async (req, res) => {
  const tmpFile = path.join(os.tmpdir(), `temp-${Date.now()}.txt`);
  await fs.writeFile(tmpFile, req.body.content);
  const processed = await processFile(tmpFile);
  // ファイルを削除していない - メモリリークの原因
  res.json(processed);
});
```

---

## エラーハンドリング

### ✅ すべての例外を処理

**必ず実装**: try-catch でエラーをキャッチし、適切なレスポンスを返す

```javascript
// ✅ 良い例
functions.http('robustFunction', async (req, res) => {
  try {
    const result = await processRequest(req.body);
    res.status(200).json(result);
  } catch (error) {
    // エラーログを記録
    console.error('Error processing request:', {
      error: error.message,
      stack: error.stack,
      requestBody: req.body,
    });
    
    // 適切なステータスコードとメッセージを返す
    if (error.name === 'ValidationError') {
      res.status(400).json({ error: 'Invalid request data' });
    } else if (error.name === 'NotFoundError') {
      res.status(404).json({ error: 'Resource not found' });
    } else {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
});

// ❌ 悪い例 - 例外を処理していない
functions.http('fragileFunction', async (req, res) => {
  const result = await processRequest(req.body); // エラーでクラッシュする
  res.json(result);
});
```

### ✅ カスタムエラークラス

**推奨**: 明確なエラー分類のためにカスタムエラーを定義

```javascript
// ✅ 良い例
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ValidationError';
    this.statusCode = 400;
  }
}

class NotFoundError extends Error {
  constructor(message) {
    super(message);
    this.name = 'NotFoundError';
    this.statusCode = 404;
  }
}

functions.http('typedErrors', async (req, res) => {
  try {
    if (!req.body.id) {
      throw new ValidationError('ID is required');
    }
    
    const user = await db.getUser(req.body.id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    
    res.json(user);
  } catch (error) {
    console.error('Error:', error);
    const statusCode = error.statusCode || 500;
    res.status(statusCode).json({ error: error.message });
  }
});
```

---

## ロギングとモニタリング

### ✅ 構造化ログの記録

**必ず実装**: JSON 形式の構造化ログを使用する

```javascript
// ✅ 良い例: 構造化ログ
functions.http('structuredLogging', (req, res) => {
  const globalLogFields = {};
  
  // トレースコンテキストの関連付け
  const traceHeader = req.header('X-Cloud-Trace-Context');
  if (traceHeader) {
    const [trace] = traceHeader.split('/');
    const projectId = process.env.GOOGLE_CLOUD_PROJECT;
    globalLogFields['logging.googleapis.com/trace'] = 
      `projects/${projectId}/traces/${trace}`;
  }
  
  // 構造化ログエントリ
  const logEntry = {
    severity: 'INFO',
    message: 'Processing request',
    userId: req.body.userId,
    action: 'create_user',
    ...globalLogFields,
  };
  
  console.log(JSON.stringify(logEntry));
  
  res.send('OK');
});

// ❌ 悪い例: 非構造化ログ
functions.http('unstructuredLogging', (req, res) => {
  console.log('Processing request for user ' + req.body.userId);
  res.send('OK');
});
```

### ✅ 重大度レベルの使用

**必ず実装**: 適切な重大度レベルを指定する

```javascript
// ✅ 良い例
function logInfo(message, data = {}) {
  console.log(JSON.stringify({
    severity: 'INFO',
    message,
    ...data,
  }));
}

function logError(message, error, data = {}) {
  console.error(JSON.stringify({
    severity: 'ERROR',
    message,
    error: error.message,
    stack: error.stack,
    ...data,
  }));
}

function logWarning(message, data = {}) {
  console.warn(JSON.stringify({
    severity: 'WARNING',
    message,
    ...data,
  }));
}

functions.http('properLogging', async (req, res) => {
  logInfo('Request received', { userId: req.body.userId });
  
  try {
    const result = await processRequest(req.body);
    logInfo('Request processed successfully', { result });
    res.json(result);
  } catch (error) {
    logError('Request processing failed', error, { 
      userId: req.body.userId 
    });
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### ✅ HTTPリクエスト情報のログ記録

**推奨**: httpRequest フィールドを使用する

```javascript
// ✅ 良い例
functions.http('httpRequestLogging', (req, res) => {
  const logEntry = {
    severity: 'INFO',
    message: 'HTTP request received',
    httpRequest: {
      requestMethod: req.method,
      requestUrl: req.url,
      userAgent: req.get('user-agent'),
      remoteIp: req.ip,
      referer: req.get('referer'),
    },
  };
  
  console.log(JSON.stringify(logEntry));
  res.send('OK');
});
```

---

## 開発とテスト

### ✅ ローカル開発環境

**推奨**: Functions Framework を使用したローカルテスト

```bash
# インストール
npm install @google-cloud/functions-framework

# package.json
{
  "scripts": {
    "start": "functions-framework --target=myHttpFunction --port=8080",
    "test": "jest"
  }
}

# ローカル起動
npm start

# テスト実行
curl http://localhost:8080 -d '{"userId": "123"}' -H "Content-Type: application/json"
```

### ✅ ユニットテスト

**必ず実装**: 関数のユニットテストを作成する

```javascript
// myFunction.js
exports.myHttpFunction = (req, res) => {
  if (!req.body.userId) {
    return res.status(400).json({ error: 'userId is required' });
  }
  res.json({ userId: req.body.userId, status: 'processed' });
};

// myFunction.test.js
const { myHttpFunction } = require('./myFunction');

describe('myHttpFunction', () => {
  let req, res;
  
  beforeEach(() => {
    req = {
      body: {},
    };
    res = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn(),
    };
  });
  
  test('should return 400 if userId is missing', () => {
    myHttpFunction(req, res);
    expect(res.status).toHaveBeenCalledWith(400);
    expect(res.json).toHaveBeenCalledWith({ error: 'userId is required' });
  });
  
  test('should process valid request', () => {
    req.body.userId = '123';
    myHttpFunction(req, res);
    expect(res.json).toHaveBeenCalledWith({ 
      userId: '123', 
      status: 'processed' 
    });
  });
});
```

---

## デプロイと運用

### ✅ デプロイ設定

**推奨**: 適切なリソース設定でデプロイする

```bash
# ✅ 良い例: 明示的なリソース設定
gcloud functions deploy myHttpFunction \
  --gen2 \
  --runtime=nodejs20 \
  --region=asia-northeast1 \
  --source=. \
  --entry-point=myHttpFunction \
  --trigger-http \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --timeout=60s \
  --max-instances=100 \
  --min-instances=1 \
  --concurrency=80

# セキュアなエンドポイント（認証必須）
gcloud functions deploy secureFunction \
  --gen2 \
  --runtime=nodejs20 \
  --region=asia-northeast1 \
  --source=. \
  --entry-point=secureFunction \
  --trigger-http \
  --no-allow-unauthenticated
```

### ✅ 環境変数の設定

**推奨**: デプロイ時に環境変数を設定する

```bash
# ✅ 良い例
gcloud functions deploy myFunction \
  --gen2 \
  --set-env-vars PROJECT_ID=my-project,REGION=asia-northeast1

# Secret Manager の使用
gcloud functions deploy myFunction \
  --gen2 \
  --set-secrets API_KEY=api-key:latest
```

---

## チェックリスト

### 基本実装

- [ ] すべてのコードパスでHTTPレスポンスを送信している
- [ ] 冪等性を考慮した実装になっている
- [ ] レスポンス送信前にすべての非同期処理を完了している
- [ ] すべての例外を try-catch で処理している

### セキュリティ

- [ ] 公開すべきでないエンドポイントに認証を実装している
- [ ] すべてのユーザー入力を検証している
- [ ] HTMLエスケープを実施している
- [ ] 機密情報は Secret Manager で管理している
- [ ] CORS ヘッダーを適切に設定している
- [ ] IAM 権限を最小権限の原則で設定している

### パフォーマンス

- [ ] DB クライアントなどの再利用可能なオブジェクトをグローバルスコープで初期化している
- [ ] 不要な依存関係を削除している
- [ ] 一時ファイルを使用後に削除している
- [ ] 最小インスタンス数を設定してコールドスタートを削減している

### ロギングとモニタリング

- [ ] JSON 形式の構造化ログを使用している
- [ ] 適切な重大度レベル（INFO, WARNING, ERROR）を指定している
- [ ] トレースコンテキストを関連付けている
- [ ] エラー時に十分な情報をログに記録している

### テストとデプロイ

- [ ] ユニットテストを作成している
- [ ] ローカル環境でテストしている
- [ ] 適切なリソース設定（メモリ、CPU、タイムアウト）でデプロイしている
- [ ] 環境変数や Secret を適切に設定している

---

## よくある間違いと修正方法

### ❌ レスポンスを送信せずに処理を終了

```javascript
// ❌ 悪い例
functions.http('bad', (req, res) => {
  if (req.method !== 'POST') {
    return; // レスポンスを送信していない
  }
  res.send('OK');
});

// ✅ 修正後
functions.http('good', (req, res) => {
  if (req.method !== 'POST') {
    return res.status(405).send('Method Not Allowed');
  }
  res.send('OK');
});
```

### ❌ 非同期処理を待たずにレスポンス送信

```javascript
// ❌ 悪い例
functions.http('bad', (req, res) => {
  saveToDatabase(req.body); // await していない
  res.send('OK');
});

// ✅ 修正後
functions.http('good', async (req, res) => {
  await saveToDatabase(req.body);
  res.send('OK');
});
```

### ❌ グローバル変数を毎回初期化

```javascript
// ❌ 悪い例
functions.http('bad', async (req, res) => {
  const db = new Firestore(); // 毎回初期化
  const result = await db.collection('users').get();
  res.json(result);
});

// ✅ 修正後
const db = new Firestore(); // グローバルスコープで1回だけ

functions.http('good', async (req, res) => {
  const result = await db.collection('users').get();
  res.json(result);
});
```

### ❌ 機密情報をコードに直接記述

```javascript
// ❌ 悪い例
const API_KEY = 'sk-1234567890abcdef';

// ✅ 修正後
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

async function getApiKey() {
  const [version] = await client.accessSecretVersion({
    name: `projects/${process.env.PROJECT_ID}/secrets/api-key/versions/latest`,
  });
  return version.payload.data.toString();
}
```

---

## 参考リソース

- [Cloud Run 公式ドキュメント](https://cloud.google.com/run/docs)
- [Functions Framework](https://github.com/GoogleCloudPlatform/functions-framework)
- [Cloud Run ベストプラクティス](https://cloud.google.com/run/docs/tips/general)
- [セキュリティ設計の概要](https://cloud.google.com/run/docs/securing/security)

---

**最終更新**: 2026年1月3日
**バージョン**: 1.0
