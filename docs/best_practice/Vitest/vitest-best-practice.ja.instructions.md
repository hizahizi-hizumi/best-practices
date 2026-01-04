```instructions
---
description: 'Vitestを使用したテストコードを書く際のベストプラクティスと推奨パターン'
applyTo: '**/*.test.ts, **/*.spec.ts, **/*.test.tsx, **/*.spec.tsx, **/*.test.js, **/*.spec.js, **/*.test.jsx, **/*.spec.jsx, **/tests/**/*.ts, **/tests/**/*.tsx, **/tests/**/*.js, **/tests/**/*.jsx, **/test/**/*.ts, **/test/**/*.tsx, **/test/**/*.js, **/test/**/*.jsx'
---

# Vitest ベストプラクティス

Vitestを使用したテストコードを書く際の推奨パターンと規約を定義する。

## スコープとコンテキスト

- フレームワーク: Vitest 1.0以降
- 環境: Node.js / jsdom / happy-dom
- Jest APIとの互換性を活用する

## ファイル構成

### 命名規則

- `.test.{ts,tsx,js,jsx}` または `.spec.{ts,tsx,js,jsx}` を使用する
- テスト対象と同じディレクトリか `tests/` 配下に配置する

### テストのグループ化

- 関連するテストは `describe` でグループ化する
- 階層構造で機能やメソッド単位を明確にする

```typescript
// 推奨: 階層的なグループ化
describe('Calculator', () => {
  describe('add', () => {
    test('adds positive numbers', () => {
      expect(add(1, 2)).toBe(3)
    })
  })
})

// 非推奨: フラットなテスト
test('add positive numbers', () => {
  expect(add(1, 2)).toBe(3)
})
```

### パラメータ化テスト

- 同じロジックを異なるデータでテストする場合は `test.each` を使用する

```typescript
// 推奨
test.each([
  [1, 1, 2],
  [1, 2, 3],
  [-1, 1, 0],
])('add(%i, %i) -> %i', (a, b, expected) => {
  expect(a + b).toBe(expected)
})

// 非推奨: 重複するテスト
test('add(1, 1)', () => expect(1 + 1).toBe(2))
test('add(1, 2)', () => expect(1 + 2).toBe(3))
```

## テストの書き方

### テスト名

- テスト対象と期待される動作を明確に記述する
- 条件と結果を含める (例: `ユーザーが存在しない場合にnullを返す`)

```typescript
// 推奨: 具体的な動作を記述
test('returns null when user does not exist', () => {
  expect(getUser('unknown')).toBeNull()
})

// 非推奨: 曖昧な名前
test('getUser', () => {
  expect(getUser('unknown')).toBeNull()
})
```

### AAAパターン

- Arrange-Act-Assertの3セクションでテストを構造化する
- 各セクションを空行で区切る

```typescript
test('can create user', () => {
  // Arrange
  const userData = { name: 'John', age: 30 }
  // Act
  const user = createUser(userData)
  // Assert
  expect(user.name).toBe('John')
})
```

### 単一責任原則

- 1つのテストで1つの概念のみをテストする
- 複数のアサーションは関連する場合のみ許容される

```typescript
// 推奨: 焦点を絞ったテスト
test('sets user name correctly', () => {
  expect(createUser({ name: 'John' }).name).toBe('John')
})

// 非推奨: 複数の独立した検証
test('creates user correctly', () => {
  const user = createUser({ name: 'John', age: 30 })
  expect(user.name).toBe('John')
  expect(user.age).toBe(30)
  expect(user.isActive).toBe(true)
  expect(user.createdAt).toBeDefined()
})
```

### test.skipとtest.only

- 未実装機能のテストを一時的にスキップする場合は `test.skip` を使用する
- デバッグ中にのみ `test.only` を使用する
- `test.only` は決してコミットしない

```typescript
test.skip('unimplemented feature', () => { /* TODO */ })
test.only('debugging', () => { /* local only */ })
```

## アサーション

### マッチャーの選択

- 最も明確なエラーメッセージを提供するマッチャーを使用する
- 目的に特化したマッチャーを優先する

```typescript
// 推奨: 適切なマッチャー
expect(value).toBe(true)          // プリミティブ
expect(obj).toEqual(expected)     // オブジェクト/配列
expect(arr).toContain(item)       // 配列要素
expect(fn).toThrow(Error)         // 例外
expect(num).toBeGreaterThan(0)    // 数値比較

// 非推奨: 冗長または不明瞭
expect(value === true).toBe(true)
expect(JSON.stringify(obj)).toBe(JSON.stringify(expected))
```

### カスタムメッセージとソフトアサーション

- 複雑なテストにはカスタムエラーメッセージを追加する
- 複数の関連するアサーションの一括検証には `expect.soft` を使用する

```typescript
// カスタムメッセージ
expect(
  calculateDiscount(price, percentage),
  `割引計算エラー: price=${price}, percentage=${percentage}`
).toBe(expectedDiscount)

// ソフトアサーション
test('validate multiple properties', () => {
  expect.soft(user.name).toBe('John')
  expect.soft(user.age).toBe(30)
})
```

## モック

### モック関数

- `vi.fn()` でモック関数を作成する
- 既存のメソッドを監視する場合は `vi.spyOn()` を使用する
- モジュール全体をモックする場合は `vi.mock()` を使用する

```typescript
// モック関数
const callback = vi.fn()
expect(callback).toHaveBeenCalledTimes(3)

// スパイ
const spy = vi.spyOn(utils, 'formatDate')
spy.mockRestore()

// モジュールモック
vi.mock('./api', () => ({
  fetchUser: vi.fn(() => Promise.resolve({ id: '1' }))
}))
```

### モックのクリーンアップ

- 設定ファイルで `clearMocks` と `restoreMocks` を有効にする
- テスト間での状態の漏洩を防ぐ

```typescript
// vitest.config.ts (推奨)
export default defineConfig({
  test: {
    clearMocks: true,
    restoreMocks: true,
  },
})

// または各テストファイルで
beforeEach(() => vi.clearAllMocks())
afterEach(() => vi.restoreAllMocks())
```

## 非同期テスト

### async/await

- 非同期テストには常に `async/await` を使用する
- Promiseチェーンは使用しない
- `resolves` / `rejects` マッチャーには常に `await` を付ける

```typescript
// 推奨
test('fetch async data', async () => {
  const data = await fetchData()
  expect(data).toBeDefined()
})

test('Promise resolves', async () => {
  await expect(fetchUser('123')).resolves.toEqual({ id: '123' })
})

// 非推奨: awaitなし
test('async', () => {
  fetchData().then(data => expect(data).toBeDefined())
})
```

### アサーション数の保証

- コールバック内のアサーションには `expect.assertions()` を使用する

```typescript
test('validate inside callback', async () => {
  expect.assertions(2)
  await processAsync(data => {
    expect(data).toBeTruthy()
    expect(data.id).toBeDefined()
  })
})
```

## スナップショットテスト

### 基本的なスナップショット

```typescript
// ✅ 良い例 - 小さく焦点を絞ったスナップショット
test('formats user info correctly', () => {
  const formatted = formatUser({ name: 'John', age: 30 })
  expect(formatted).toMatchSnapshot()
})
```

### インラインスナップショット

小さなスナップショットにはインラインスナップショットを使用する。

```typescript
// ✅ 良い例
test('formats data correctly', () => {
  expect(formatData({ name: 'John' })).toMatchInlineSnapshot(`
    {
      "name": "John",
    }
  `)
})
```

### スナップショットのベストプラクティス

- スナップショットは小さく焦点を絞る
- 頻繁に変更される部分はスナップショットにしない
- 動的な値(日付、IDなど)は除外またはモックする

```typescript
// ✅ 良い例 - 動的な値を除外
test('create user', () => {
  const user = createUser({ name: 'John' })
  
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  })
})

// ❌ 避けるべき - 動的な値を含む
test('create user', () => {
  const user = createUser({ name: 'John' })
  expect(user).toMatchSnapshot() // idとcreatedAtが毎回変わる
})
```

## セットアップとクリーンアップ

### ライフサイクルフック

- `beforeEach` / `afterEach`: 各テストの前後で実行される
- `beforeAll` / `afterAll`: すべてのテストの前後で1回実行される
- `onTestFinished`: テスト完了時に自動クリーンアップ

```typescript
// テストごとのセットアップ
beforeEach(async () => {
  db = await createTestDatabase()
})
afterEach(async () => {
  await db.close()
})

// グローバルなセットアップ
beforeAll(async () => {
  server = await startTestServer()
})
afterAll(async () => {
  await server.close()
})

// 自動クリーンアップ
test('test', async () => {
  const db = connectDatabase()
  onTestFinished(() => db.close())
})
```

## UIコンポーネントテスト

### Testing Libraryとの統合

- React Testing Library / Vue Testing Libraryと併用する
- アクセシブルな要素選択には `screen.getByRole()` を優先する
- ユーザー操作には `userEvent` を使用する

```typescript
// React
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

test('button click', async () => {
  const handleClick = vi.fn()
  render(<Button onClick={handleClick} />)
  await userEvent.setup().click(screen.getByRole('button'))
  expect(handleClick).toHaveBeenCalledTimes(1)
})

// Vue
import { render } from '@testing-library/vue'

test('component', () => {
  const { getByText } = render(MyComponent, { props: { message: 'Hello' } })
  expect(getByText('Hello')).toBeTruthy()
})
```

## アンチパターン

### 避けるべきパターン

- **awaitの省略**: `resolves` / `rejects` には常に `await` を付ける
- **test.onlyのコミット**: 決してコミットしない
- **グローバル状態**: `beforeEach` で状態をリセットする
- **テスト順序への依存**: 各テストを独立させる

```typescript
// 非推奨: 共有状態
let counter = 0
test('test 1', () => counter++)
test('test 2', () => expect(counter).toBe(1)) // 失敗する

// 推奨: 独立したテスト
describe('tests', () => {
  let counter: number
  beforeEach(() => { counter = 0 })
  test('test 1', () => { counter++; expect(counter).toBe(1) })
  test('test 2', () => { counter++; expect(counter).toBe(1) })
})
```

## パフォーマンス

### 並列実行とタイムアウト

- 独立したテストは `test.concurrent` で並列実行する
- 順序依存のテストには `test.sequential` を使用する
- 長時間実行されるテストには適切なタイムアウトを設定する

```typescript
// 並列実行
test.concurrent('parallel 1', async () => await asyncOp())
test.concurrent('parallel 2', async () => await asyncOp())

// 順次実行
test.sequential('sequential 1', async () => await sequentialOp())

// タイムアウト
test('long-running', async () => await longOp(), 30000)
```

## 設定

### 推奨される設定

- `globals: true` でインポートを不要にする
- `clearMocks` と `restoreMocks` を有効にする
- カバレッジ閾値を設定する(80%推奨)
- 環境を指定する(`node` / `jsdom` / `happy-dom`)

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    clearMocks: true,
    restoreMocks: true,
    testTimeout: 10000,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      thresholds: { branches: 80, functions: 80, lines: 80, statements: 80 },
    },
  },
})

// tsconfig.json
{ "compilerOptions": { "types": ["vitest/globals"] } }
```

## チェックリスト

### コミット前の確認事項

- [ ] テスト名が明確で具体的である
- [ ] 各テストが独立して実行できる
- [ ] 非同期操作に `await` を使用している
- [ ] `test.only` / `test.skip` が削除されている
- [ ] モックが適切にクリーンアップされている
- [ ] スナップショットに動的な値が含まれていない
- [ ] 適切なマッチャーが使用されている
- [ ] エラーケースがテストされている

## 優先順位

1. ユーザーの明示的な指示を最優先する
2. テストの独立性とクリーンアップ
3. 非同期操作の適切な処理
4. 明確なテスト名と適切なマッチャー

```
