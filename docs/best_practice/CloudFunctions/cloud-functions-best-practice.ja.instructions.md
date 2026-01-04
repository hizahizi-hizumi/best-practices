---
description: 'Cloud Functions (2nd gen) HTTPエンドポイント実装のベストプラクティス'
applyTo: '**/*.js, **/*.ts, **/main.py'
---

# Cloud Functions HTTPエンドポイント ベストプラクティス

## 目的とスコープ

Cloud Functions (2nd gen) HTTPエンドポイント実装のベストプラクティスを適用する。
対象ランタイム: Node.js 20+, Python 3.11+。

## 優先事項

1. 常にすべてのコードパスでHTTPレスポンスを送信する（最重要）
2. すべての操作に冪等性を実装する
3. レスポンス送信前に非同期操作を完了させる
4. セキュリティ: 認証、入力検証、Secret Managerの使用
5. パフォーマンス: 接続のグローバル再利用、依存関係の最小化
6. ログ: トレースコンテキスト付き構造化JSONログの使用

## コア要件

### 常にHTTPレスポンスを送信

すべてのコードパスでHTTPレスポンスを送信する。

```javascript
// Good: すべてのパスでレスポンスを返す
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

// Bad: レスポンス未送信はタイムアウトと課金の原因となる
functions.http('badFunction', (req, res) => {
  console.log('Processing...');
  // res.send()がない - タイムアウトまで実行される
});
```

**理由**: レスポンス未送信はタイムアウト課金と予測不能な動作を引き起こす。

### 冪等性の実装

関数が複数回呼び出されても同じ結果を生成することを保証する。

```javascript
// Good: 作成前に既存をチェック
functions.http('createUser', async (req, res) => {
  const userId = req.body.userId;
  
  if (!userId) {
    return res.status(400).json({ error: 'userId is required' });
  }
  
  const existingUser = await db.getUser(userId);
  if (existingUser) {
    return res.status(200).json(existingUser);
  }
  
  const user = await db.createUser(userId, req.body);
  res.status(201).json(user);
});

// Bad: リトライ時に重複を作成
functions.http('badCreateUser', async (req, res) => {
  const user = await db.createUser(req.body.userId, req.body);
  res.status(201).json(user);
});
```

**理由**: 失敗時の安全なリトライを可能にし、重複イベントを処理する。

### レスポンス前に非同期操作を完了

HTTPレスポンス送信前にすべての非同期操作を待機する。

```javascript
// Good: すべての操作を待機
functions.http('processData', async (req, res) => {
  try {
    await saveToDatabase(req.body);
    await sendNotification(req.body.email);
    res.status(200).send('Processing completed');
  } catch (error) {
    console.error('Error:', error);
    res.status(500).send('Error occurred');
  }
});

// Bad: 操作完了前にレスポンス
functions.http('badProcessData', (req, res) => {
  saveToDatabase(req.body); // awaitなし
  res.status(200).send('Processing started');
});
```

## HTTPエンドポイント設計

### Functions Framework標準パターンの使用

```javascript
// Node.js
import { http } from '@google-cloud/functions-framework';

http('myHttpFunction', (req, res) => {
  // リクエストを処理
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

### ✅ 適切なHTTPメソッド処理

**必須**: HTTPメソッドを検査し、適切に処理する

```javascript
// ✅ Good example
functions.http('apiEndpoint', (req, res) => {
  res.set('Access-Control-Allow-Origin', '*');
  
  if (req.method === 'OPTIONS') {
    // CORSプリフライトリクエスト
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

### ✅ 適切なCORS設定

**必須**: CORSヘッダーを適切に設定する

```javascript
// ✅ Good example: 本番環境 - 特定のオリジンのみ許可
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

// ❌ Bad example: すべてのオリジンを許可（開発環境のみで使用）
functions.http('insecureApi', (req, res) => {
  res.set('Access-Control-Allow-Origin', '*'); // 本番環境では避ける
  res.json({ message: 'Success' });
});
```

---

## セキュリティ実装

### 認証の実装

保護されたエンドポイントに対してIDトークンを検証する。

```javascript
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
    const result = await processRequest(payload);
    res.json(result);
  } catch (error) {
    console.error('Authentication failed:', error);
    res.status(401).json({ error: 'Invalid token' });
  }
});
```

最小権限でIAMを設定する:
```bash
gcloud run services add-iam-policy-binding SERVICE_NAME \
  --member='serviceAccount:caller@project.iam.gserviceaccount.com' \
  --role='roles/run.invoker'
```

### すべてのユーザー入力を検証

型、長さ、形式を検証し、HTMLをエスケープする。

```javascript
const escapeHtml = require('escape-html');

functions.http('userInput', (req, res) => {
  const name = req.body.name;
  const email = req.body.email;
  
  if (!name || typeof name !== 'string' || name.length > 100) {
    return res.status(400).json({ error: 'Invalid name' });
  }
  
  if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return res.status(400).json({ error: 'Invalid email' });
  }
  
  const safeName = escapeHtml(name);
  res.send(`Hello ${safeName}!`);
});

// Bad: 検証なしはXSSの原因となる
functions.http('unsafeInput', (req, res) => {
  res.send(`Hello ${req.body.name}!`);
});
```

### シークレットにはSecret Managerを使用

シークレットをハードコードまたは環境変数で管理しない。

```javascript
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
  res.send('OK');
});

// Bad: ハードコードや環境変数はシークレットを露出させる
const API_KEY = 'hardcoded-secret-key'; // 絶対にしない
const API_KEY = process.env.API_KEY; // プレーンテキストで保存される
```

## パフォーマンス最適化

### 再利用可能なオブジェクトをグローバルに初期化

DBクライアントと接続をグローバルスコープで一度だけ初期化する。

```javascript
const { Firestore } = require('@google-cloud/firestore');
const db = new Firestore(); // 一度だけ初期化

functions.http('optimizedFunction', async (req, res) => {
  const doc = await db.collection('users').doc(req.body.userId).get();
  res.json(doc.data());
});

// Bad: リクエストごとに再初期化
functions.http('slowFunction', async (req, res) => {
  const db = new Firestore(); // 遅い
  const doc = await db.collection('users').doc(req.body.userId).get();
  res.json(doc.data());
});
```

**理由**: リクエスト間で接続を再利用し、コールドスタートのオーバーヘッドを回避する。

### 必要に応じて遅延初期化を使用

```javascript
let dbConnection = null;

async function getDbConnection() {
  if (!dbConnection) {
    dbConnection = await initializeDatabase();
  }
  return dbConnection;
}
```

### 依存関係を最小化

コールドスタート時間を短縮するため、必要なモジュールのみをインポートする。

```javascript
// Good: 必要なインポートのみ
const { Firestore } = require('@google-cloud/firestore');

// Bad: 未使用のインポートは起動を遅くする
const _ = require('lodash');
const moment = require('moment');
```

### 一時ファイルをクリーンアップ

finallyブロックで一時ファイルを削除し、ディスクリークを防ぐ。

```javascript
const fs = require('fs').promises;
const os = require('os');
const path = require('path');

functions.http('fileProcessing', async (req, res) => {
  const tmpFile = path.join(os.tmpdir(), `temp-${Date.now()}.txt`);
  
  try {
    await fs.writeFile(tmpFile, req.body.content);
    const processed = await processFile(tmpFile);
    res.json(processed);
  } finally {
    try {
      await fs.unlink(tmpFile);
    } catch (error) {
      console.error('Failed to delete temp file:', error);
    }
  }
});
```

## エラーハンドリング

### すべての例外をキャッチ

try-catchを使用してエラーを処理し、適切なHTTPステータスを返す。

```javascript
functions.http('robustFunction', async (req, res) => {
  try {
    const result = await processRequest(req.body);
    res.status(200).json(result);
  } catch (error) {
    console.error('Error:', {
      error: error.message,
      stack: error.stack,
      requestBody: req.body,
    });
    
    if (error.name === 'ValidationError') {
      res.status(400).json({ error: 'Invalid request data' });
    } else if (error.name === 'NotFoundError') {
      res.status(404).json({ error: 'Resource not found' });
    } else {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
});
```

### カスタムエラークラスを定義

```javascript
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

## ログ

### 構造化JSONログを使用

重要度とトレースコンテキストを含むJSONをログに記録する。

```javascript
functions.http('structuredLogging', (req, res) => {
  const traceHeader = req.header('X-Cloud-Trace-Context');
  const trace = traceHeader ? traceHeader.split('/')[0] : null;
  
  console.log(JSON.stringify({
    severity: 'INFO',
    message: 'Processing request',
    userId: req.body.userId,
    'logging.googleapis.com/trace': trace ? 
      `projects/${process.env.GOOGLE_CLOUD_PROJECT}/traces/${trace}` : undefined,
  }));
  
  res.send('OK');
});
```

### 重要度レベルを使用

console.log/warn/errorでINFO、WARNING、ERRORを使用する。

```javascript
function logError(message, error, data = {}) {
  console.error(JSON.stringify({
    severity: 'ERROR',
    message,
    error: error.message,
    stack: error.stack,
    ...data,
  }));
}
```

## 開発とテスト

### Functions Frameworkでローカルテスト

```bash
npm install @google-cloud/functions-framework

# package.json
{
  "scripts": {
    "start": "functions-framework --target=myHttpFunction --port=8080",
    "test": "jest"
  }
}

npm start
curl http://localhost:8080 -d '{"userId": "123"}' -H "Content-Type: application/json"
```

### ユニットテストを書く

```javascript
const { myHttpFunction } = require('./myFunction');

describe('myHttpFunction', () => {
  let req, res;
  
  beforeEach(() => {
    req = { body: {} };
    res = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn(),
    };
  });
  
  test('should return 400 if userId is missing', () => {
    myHttpFunction(req, res);
    expect(res.status).toHaveBeenCalledWith(400);
  });
  
  test('should process valid request', () => {
    req.body.userId = '123';
    myHttpFunction(req, res);
    expect(res.json).toHaveBeenCalledWith({ userId: '123', status: 'processed' });
  });
});
```

## デプロイ

### 適切なリソース設定でデプロイ

```bash
gcloud functions deploy myHttpFunction \
  --gen2 \
  --runtime=nodejs20 \
  --region=asia-northeast1 \
  --entry-point=myHttpFunction \
  --trigger-http \
  --memory=512Mi \
  --cpu=1 \
  --timeout=60s \
  --max-instances=100 \
  --min-instances=1 \
  --concurrency=80 \
  --set-env-vars PROJECT_ID=my-project \
  --set-secrets API_KEY=api-key:latest

# セキュアなエンドポイント（認証必須）
gcloud functions deploy secureFunction \
  --gen2 \
  --runtime=nodejs20 \
  --trigger-http \
  --no-allow-unauthenticated
```

## チェックリスト

### コア
- [ ] すべてのコードパスでHTTPレスポンスを送信
- [ ] 冪等性を実装
- [ ] レスポンス前に非同期操作を完了
- [ ] try-catchですべての例外をキャッチ

### セキュリティ
- [ ] 保護されたエンドポイントに認証を実装
- [ ] すべてのユーザー入力を検証
- [ ] HTML出力をエスケープ
- [ ] シークレットにSecret Managerを使用
- [ ] CORSを適切に設定
- [ ] 最小権限でIAMを設定

### パフォーマンス
- [ ] DBクライアントをグローバルに初期化
- [ ] 未使用の依存関係を削除
- [ ] 一時ファイルをクリーンアップ
- [ ] コールドスタート削減のためmin-instancesを設定

### ログ
- [ ] 構造化JSONログを使用
- [ ] 重要度レベルを指定（INFO、WARNING、ERROR）
- [ ] トレースコンテキストを関連付け

### テスト
- [ ] ユニットテストを書く
- [ ] Functions Frameworkでローカルテスト
- [ ] デプロイに適切なリソースを設定

## よくある間違い

### レスポンスの未送信
```javascript
// Bad: レスポンスが送信されない
if (req.method !== 'POST') return;

// Good
if (req.method !== 'POST') return res.status(405).send('Method Not Allowed');
```

### awaitなしの非同期処理
```javascript
// Bad
saveToDatabase(req.body); // awaitなし
res.send('OK');

// Good
await saveToDatabase(req.body);
res.send('OK');
```

### 接続の再初期化
```javascript
// Bad: リクエストごとに再初期化
const db = new Firestore();

// Good: 一度だけグローバルに初期化
const db = new Firestore(); // 関数外
```

## 参考資料

- [Cloud Runドキュメント](https://cloud.google.com/run/docs)
- [Functions Framework](https://github.com/GoogleCloudPlatform/functions-framework)
- [Cloud Runベストプラクティス](https://cloud.google.com/run/docs/tips/general)

