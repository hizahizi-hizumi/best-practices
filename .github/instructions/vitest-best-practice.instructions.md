---
description: 'Vitestを使用したテストコードの作成におけるベストプラクティスと推奨パターン'
applyTo: '**/*.test.ts, **/*.spec.ts, **/*.test.tsx, **/*.spec.tsx, **/*.test.js, **/*.spec.js, **/*.test.jsx, **/*.spec.jsx, **/tests/**/*.ts, **/tests/**/*.tsx, **/tests/**/*.js, **/tests/**/*.jsx, **/test/**/*.ts, **/test/**/*.tsx, **/test/**/*.js, **/test/**/*.jsx'
---

# Vitest ベストプラクティス

Vitestを使用した効果的で保守可能なテストコードを書くためのガイドラインです。

## プロジェクトコンテキスト

- テストフレームワーク: Vitest
- 対象環境: Node.js、ブラウザ (jsdom, happy-dom)
- 推奨バージョン: Vitest 1.0以上
- 互換性: Jest APIとの高い互換性あり

## テストファイルの構成

### ファイル命名規則

- テストファイル名には `.test.` または `.spec.` を含める
- テスト対象ファイルと同じディレクトリまたは `tests/` ディレクトリに配置
- 例: `sum.test.ts`, `calculator.spec.ts`

### describeを使ったテストの整理

関連するテストは `describe` でグループ化し、階層構造を明確にする。

```typescript
// ✅ 良い例
import { describe, expect, test } from 'vitest'

describe('Calculator', () => {
  describe('add', () => {
    test('正の数を加算する', () => {
      expect(add(1, 2)).toBe(3)
    })

    test('負の数を加算する', () => {
      expect(add(-1, -2)).toBe(-3)
    })
  })

  describe('subtract', () => {
    test('正の数を減算する', () => {
      expect(subtract(5, 3)).toBe(2)
    })
  })
})

// ❌ 避けるべき例
test('add正の数', () => {
  expect(add(1, 2)).toBe(3)
})

test('add負の数', () => {
  expect(add(-1, -2)).toBe(-3)
})

test('subtract正の数', () => {
  expect(subtract(5, 3)).toBe(2)
})
```

### test.eachでパラメータ化テスト

同じロジックを異なるデータでテストする場合は `test.each` を使用する。

```typescript
// ✅ 良い例
import { expect, test } from 'vitest'

test.each([
  [1, 1, 2],
  [1, 2, 3],
  [2, 1, 3],
  [-1, 1, 0],
])('add(%i, %i) -> %i', (a, b, expected) => {
  expect(a + b).toBe(expected)
})

// ❌ 避けるべき例 - 重複したテストケース
test('add(1, 1)', () => {
  expect(1 + 1).toBe(2)
})

test('add(1, 2)', () => {
  expect(1 + 2).toBe(3)
})

test('add(2, 1)', () => {
  expect(2 + 1).toBe(3)
})
```

## テストの書き方

### テスト名は明確で具体的に

テスト名は何をテストしているのか、期待される動作は何かを明確に示す。

```typescript
// ✅ 良い例
test('ユーザーが存在しない場合、nullを返す', () => {
  expect(getUser('unknown')).toBeNull()
})

test('無効なメールアドレスの場合、ValidationErrorをスローする', () => {
  expect(() => validateEmail('invalid')).toThrow(ValidationError)
})

// ❌ 避けるべき例
test('getUser', () => {
  expect(getUser('unknown')).toBeNull()
})

test('validation', () => {
  expect(() => validateEmail('invalid')).toThrow()
})
```

### AAAパターン (Arrange-Act-Assert)

テストを3つのセクションに分けて構造化し、可読性を高める。

```typescript
// ✅ 良い例
test('ユーザーを作成できる', () => {
  // Arrange: テストの準備
  const userData = { name: 'John', age: 30 }

  // Act: テスト対象の実行
  const user = createUser(userData)

  // Assert: 結果の検証
  expect(user.name).toBe('John')
  expect(user.age).toBe(30)
})
```

### 1つのテストで1つの概念をテスト

テストは焦点を絞り、1つの振る舞いや概念のみをテストする。

```typescript
// ✅ 良い例
test('ユーザー名が正しく設定される', () => {
  const user = createUser({ name: 'John' })
  expect(user.name).toBe('John')
})

test('ユーザー年齢が正しく設定される', () => {
  const user = createUser({ age: 30 })
  expect(user.age).toBe(30)
})

// ❌ 避けるべき例 - 複数の概念を1つのテストで検証
test('ユーザーが正しく作成される', () => {
  const user = createUser({ name: 'John', age: 30 })
  expect(user.name).toBe('John')
  expect(user.age).toBe(30)
  expect(user.isActive).toBe(true)
  expect(user.createdAt).toBeDefined()
  expect(user.permissions).toEqual(['read'])
})
```

### test.skip と test.only の適切な使用

- `test.skip`: 一時的にテストをスキップ（コードは削除しない）
- `test.only`: デバッグ時に特定のテストのみ実行

```typescript
// 一時的にテストをスキップ
test.skip('未実装の機能テスト', () => {
  // TODO: 機能実装後にスキップを解除
})

// デバッグ中は特定のテストのみ実行
test.only('デバッグ中のテスト', () => {
  expect(debugFunction()).toBe(expectedValue)
})
```

⚠️ **重要**: `test.only` はコミット前に必ず削除すること。

## アサーション

### 適切なマッチャーの選択

明確なエラーメッセージを得るため、最も適切なマッチャーを使用する。

```typescript
// ✅ 良い例
expect(value).toBe(true)           // プリミティブの厳密な等価性
expect(obj).toEqual(expected)      // オブジェクトの深い等価性
expect(arr).toContain(item)        // 配列に要素が含まれるか
expect(str).toMatch(/pattern/)     // 文字列が正規表現にマッチ
expect(fn).toThrow(Error)          // 関数がエラーをスロー
expect(num).toBeGreaterThan(0)     // 数値比較
expect(value).toBeDefined()        // undefinedでない
expect(value).toBeTruthy()         // 真と評価される
expect(value).toBeNull()           // nullである

// ❌ 避けるべき例
expect(value === true).toBe(true)  // 冗長で不明瞭
expect(JSON.stringify(obj)).toBe(JSON.stringify(expected))  // わかりにくい
expect(arr.includes(item)).toBe(true)  // 不適切なマッチャー
```

### カスタムエラーメッセージ

複雑なテストでは、失敗時の理解を助けるカスタムメッセージを追加する。

```typescript
// ✅ 良い例
expect(
  calculateDiscount(price, percentage),
  `割引計算が正しくない: price=${price}, percentage=${percentage}`
).toBe(expectedDiscount)

expect(
  result,
  `Expected result to be positive for input: ${JSON.stringify(input)}`
).toBeGreaterThan(0)
```

### expect.softで複数のアサーション実行

最初のアサーション失敗後もテストを継続し、すべてのエラーを収集する場合に使用。

```typescript
// ✅ 適切な使用例
test('複数のプロパティを検証', () => {
  const user = createUser()
  expect.soft(user.name).toBe('John')
  expect.soft(user.age).toBe(30)
  expect.soft(user.email).toBe('john@example.com')
  // すべての失敗を一度に確認できる
})
```

## モック機能

### vi.fnでモック関数を作成

```typescript
import { expect, test, vi } from 'vitest'

// ✅ 良い例
test('コールバックが適切に呼ばれる', () => {
  const callback = vi.fn()
  
  processData([1, 2, 3], callback)
  
  expect(callback).toHaveBeenCalledTimes(3)
  expect(callback).toHaveBeenNthCalledWith(1, 1)
  expect(callback).toHaveBeenNthCalledWith(2, 2)
  expect(callback).toHaveBeenNthCalledWith(3, 3)
})
```

### vi.spyOnでメソッドをスパイ

```typescript
import { expect, test, vi } from 'vitest'
import * as utils from './utils'

// ✅ 良い例
test('ユーティリティ関数が正しく呼ばれる', () => {
  const spy = vi.spyOn(utils, 'formatDate')
  
  const result = processDate(new Date('2024-01-01'))
  
  expect(spy).toHaveBeenCalledTimes(1)
  expect(spy).toHaveBeenCalledWith(expect.any(Date))
  
  spy.mockRestore() // テスト後に元に戻す
})
```

### vi.mockでモジュール全体をモック

```typescript
import { test, vi } from 'vitest'

// ✅ 良い例
vi.mock('./api', () => ({
  fetchUser: vi.fn(() => Promise.resolve({ id: '1', name: 'John' })),
  fetchPosts: vi.fn(() => Promise.resolve([]))
}))

test('APIをモックしてテスト', async () => {
  const user = await fetchUser('123')
  expect(user.name).toBe('John')
})
```

### モックのクリーンアップ

テスト間でモックの状態が漏れないよう、適切にクリーンアップする。

```typescript
import { afterEach, beforeEach, vi } from 'vitest'

// ✅ 推奨設定
beforeEach(() => {
  vi.clearAllMocks() // モックの呼び出し履歴をクリア
})

afterEach(() => {
  vi.restoreAllMocks() // すべてのモックを元に戻す
})
```

または設定ファイルで自動化:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    clearMocks: true,
    restoreMocks: true,
  },
})
```

## 非同期テスト

### async/awaitの使用

非同期テストには `async/await` を使用する。

```typescript
// ✅ 良い例
test('非同期データを取得する', async () => {
  const data = await fetchData()
  expect(data).toBeDefined()
  expect(data.items).toHaveLength(3)
})

// ❌ 避けるべき例
test('非同期データを取得する', () => {
  fetchData().then(data => {
    expect(data).toBeDefined() // このアサーションは実行されない可能性あり
  })
})
```

### Promiseのアサーション

```typescript
// ✅ 良い例
test('Promiseが解決される', async () => {
  await expect(fetchUser('123')).resolves.toEqual({ id: '123', name: 'John' })
})

test('Promiseが拒否される', async () => {
  await expect(fetchUser('invalid')).rejects.toThrow('User not found')
})

// ❌ 避けるべき例 - awaitがない
test('Promiseが解決される', () => {
  expect(fetchUser('123')).resolves.toEqual({ id: '123', name: 'John' })
  // awaitがないため、アサーションが実行されない
})
```

⚠️ **重要**: `resolves` と `rejects` を使用する場合は、必ず `await` すること。

### expect.assertionsで非同期アサーション保証

非同期コールバック内でアサーションが確実に実行されることを保証する。

```typescript
// ✅ 良い例
test('コールバック内のアサーション', async () => {
  expect.assertions(2)
  
  await processAsync((data) => {
    expect(data).toBeTruthy()
    expect(data.id).toBeDefined()
  })
})
```

## スナップショットテスト

### 基本的なスナップショット

```typescript
// ✅ 良い例 - 小さく焦点を絞ったスナップショット
test('ユーザー情報を正しくフォーマット', () => {
  const formatted = formatUser({ name: 'John', age: 30 })
  expect(formatted).toMatchSnapshot()
})
```

### インラインスナップショット

小さなスナップショットにはインラインスナップショットを使用する。

```typescript
// ✅ 良い例
test('データを正しくフォーマット', () => {
  expect(formatData({ name: 'John' })).toMatchInlineSnapshot(`
    {
      "name": "John",
    }
  `)
})
```

### スナップショットのベストプラクティス

- スナップショットは小さく、焦点を絞る
- 頻繁に変更される部分はスナップショットしない
- 動的な値（日付、IDなど）は除外またはモックする

```typescript
// ✅ 良い例 - 動的な値を除外
test('ユーザー作成', () => {
  const user = createUser({ name: 'John' })
  
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  })
})

// ❌ 避けるべき例 - 動的な値を含む
test('ユーザー作成', () => {
  const user = createUser({ name: 'John' })
  expect(user).toMatchSnapshot() // idとcreatedAtが毎回変わる
})
```

## セットアップとクリーンアップ

### beforeEach と afterEach

各テストの前後で実行される処理を定義する。

```typescript
import { afterEach, beforeEach, describe, expect, test } from 'vitest'

describe('Database tests', () => {
  let db: Database

  beforeEach(async () => {
    // 各テスト前にデータベースをセットアップ
    db = await createTestDatabase()
    await db.seed()
  })

  afterEach(async () => {
    // 各テスト後にクリーンアップ
    await db.clear()
    await db.close()
  })

  test('データを取得できる', async () => {
    const data = await db.query('SELECT * FROM users')
    expect(data).toHaveLength(3)
  })
})
```

### beforeAll と afterAll

すべてのテストの実行前後で1回だけ実行される処理を定義する。

```typescript
import { afterAll, beforeAll, describe, test } from 'vitest'

describe('API tests', () => {
  let server: Server

  beforeAll(async () => {
    // すべてのテスト前に1回だけサーバーを起動
    server = await startTestServer()
  })

  afterAll(async () => {
    // すべてのテスト後に1回だけサーバーを停止
    await server.close()
  })

  test('エンドポイントにアクセスできる', async () => {
    const response = await fetch(`${server.url}/api/users`)
    expect(response.ok).toBe(true)
  })
})
```

### onTestFinishedフック

テスト終了時に自動的にクリーンアップ処理を実行する。

```typescript
import { onTestFinished, test } from 'vitest'

// ✅ 良い例
test('リソースを自動クリーンアップ', async () => {
  const db = connectDatabase()
  onTestFinished(() => db.close())
  
  const result = await db.query('SELECT * FROM users')
  expect(result).toBeDefined()
  // テスト終了時に自動的にdb.close()が呼ばれる
})
```

## UIコンポーネントのテスト

### React Testing Libraryとの統合

```typescript
import { render, screen } from '@testing-library/react'
import { describe, expect, test, vi } from 'vitest'
import userEvent from '@testing-library/user-event'
import MyButton from './MyButton'

// ✅ 良い例
describe('MyButton Component', () => {
  test('正しくレンダリングされる', () => {
    render(<MyButton label="Click me" />)
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument()
  })

  test('クリックイベントが発火する', async () => {
    const handleClick = vi.fn()
    const user = userEvent.setup()
    
    render(<MyButton onClick={handleClick} />)
    await user.click(screen.getByRole('button'))
    
    expect(handleClick).toHaveBeenCalledTimes(1)
  })
})
```

### Vue Testing Libraryとの統合

```typescript
import { render } from '@testing-library/vue'
import { describe, expect, test } from 'vitest'
import MyComponent from './MyComponent.vue'

// ✅ 良い例
describe('MyComponent', () => {
  test('正しくレンダリングされる', () => {
    const { getByText } = render(MyComponent, {
      props: {
        message: 'Hello Vue!',
      },
    })
    
    expect(getByText('Hello Vue!')).toBeTruthy()
  })
})
```

## よくある落とし穴と回避策

### 非同期テストでのawait忘れ

```typescript
// ❌ 間違い
test('データを取得', () => {
  expect(fetchData()).resolves.toBeDefined() // awaitがない
})

// ✅ 正しい
test('データを取得', async () => {
  await expect(fetchData()).resolves.toBeDefined()
})
```

### test.onlyのコミット

```typescript
// ❌ 間違い: 絶対にコミットしない
test.only('デバッグ中', () => {
  // 他のテストが実行されなくなる
})

// ✅ 正しい: コミット前にonlyを削除
test('デバッグ中', () => {
  // すべてのテストが実行される
})
```

### グローバルな状態の変更

```typescript
// ❌ 間違い - テスト間で状態が共有される
let counter = 0

test('カウンター増加1', () => {
  counter++
  expect(counter).toBe(1)
})

test('カウンター増加2', () => {
  counter++
  expect(counter).toBe(1) // 失敗: counterは2
})

// ✅ 正しい - 各テストで状態をリセット
describe('カウンターテスト', () => {
  let counter: number

  beforeEach(() => {
    counter = 0
  })

  test('カウンター増加1', () => {
    counter++
    expect(counter).toBe(1)
  })

  test('カウンター増加2', () => {
    counter++
    expect(counter).toBe(1) // 成功
  })
})
```

### テストの順序依存

```typescript
// ❌ 間違い - テストが順序に依存
let data: Data

test('データを作成', () => {
  data = createData()
  expect(data).toBeDefined()
})

test('データを更新', () => {
  updateData(data) // 前のテストに依存
  expect(data.updated).toBe(true)
})

// ✅ 正しい - 各テストが独立
describe('データ操作', () => {
  test('データを作成', () => {
    const data = createData()
    expect(data).toBeDefined()
  })

  test('データを更新', () => {
    const data = createData() // 独自のデータを用意
    updateData(data)
    expect(data.updated).toBe(true)
  })
})
```

## パフォーマンス最適化

### テストの並列実行

```typescript
// ✅ 並列実行可能なテスト
test.concurrent('並列テスト1', async () => {
  await someAsyncOperation()
})

test.concurrent('並列テスト2', async () => {
  await anotherAsyncOperation()
})

// 順次実行が必要なテスト
test.sequential('順次テスト1', async () => {
  await sequentialOperation()
})
```

### 適切なタイムアウト設定

```typescript
// 個別のテストでタイムアウトを指定
test('長時間実行されるテスト', async () => {
  await longRunningOperation()
}, 30000) // 30秒

// 設定ファイルでデフォルトタイムアウトを設定
// vitest.config.ts
export default defineConfig({
  test: {
    testTimeout: 10000, // 10秒
  },
})
```

## 設定

### 推奨設定例

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    environment: 'node', // または 'jsdom', 'happy-dom'
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'json'],
      include: ['src/**/*.ts'],
      exclude: ['**/*.test.ts', '**/*.spec.ts'],
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80,
      },
    },
    clearMocks: true,
    restoreMocks: true,
    testTimeout: 10000,
  },
})
```

### グローバルAPIの有効化

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true, // describe, test, expect をインポート不要に
  },
})

// tsconfig.json
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

## まとめ

### 重要なポイント

| ポイント | 説明 |
|---------|------|
| 明確なテスト名 | 何をテストし、何を期待するか明確に |
| 1テスト1概念 | テストは焦点を絞り、独立させる |
| 適切なマッチャー | エラーメッセージをわかりやすく |
| モックのクリーンアップ | テスト間の状態漏れを防ぐ |
| 非同期処理のawait | 特にresolves/rejects使用時は必須 |
| スナップショットの注意 | 動的な値を除外し、変更時はレビュー |
| 独立したテスト | テスト間の依存関係を避ける |
| test.onlyの削除 | コミット前に必ず確認 |

### チェックリスト

テスト作成時の確認事項:

- [ ] テスト名は明確で具体的か
- [ ] 各テストは独立して実行可能か
- [ ] 非同期処理は適切にawaitしているか
- [ ] モックは適切にクリーンアップされているか
- [ ] test.onlyやtest.skipがコミットに含まれていないか
- [ ] スナップショットに動的な値が含まれていないか
- [ ] エラーハンドリングのテストが含まれているか
- [ ] 適切なマッチャーを使用しているか
