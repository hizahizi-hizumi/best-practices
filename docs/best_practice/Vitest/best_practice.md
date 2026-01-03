# Vitest ベストプラクティス

## 目次

1. [テストの構成](#テストの構成)
2. [テストの書き方](#テストの書き方)
3. [アサーション](#アサーション)
4. [モック機能](#モック機能)
5. [非同期テスト](#非同期テスト)
6. [スナップショットテスト](#スナップショットテスト)
7. [カバレッジ](#カバレッジ)
8. [セットアップとクリーンアップ](#セットアップとクリーンアップ)
9. [パフォーマンス最適化](#パフォーマンス最適化)
10. [テスト環境とコンフィギュレーション](#テスト環境とコンフィギュレーション)
11. [UIコンポーネントのテスト](#uiコンポーネントのテスト)
12. [よくある落とし穴と回避策](#よくある落とし穴と回避策)

---

## テストの構成

### テストファイルの命名規則

- テストファイル名には `.test.` または `.spec.` を含める
- 例: `sum.test.ts`, `calculator.spec.ts`

```typescript
// ✅ 良い例
// sum.test.ts
// user.spec.ts

// ❌ 避けるべき例
// sum-test.ts
// test-user.ts
```

### describeを使ったテストの整理

関連するテストをグループ化することで、テストの可読性と保守性が向上します。

```typescript
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
```

### test.eachを使ったパラメータ化テスト

同じロジックを異なるデータでテストする場合は、`test.each`を使用します。

```typescript
import { expect, test } from 'vitest'

test.each([
  [1, 1, 2],
  [1, 2, 3],
  [2, 1, 3],
])('add(%i, %i) -> %i', (a, b, expected) => {
  expect(a + b).toBe(expected)
})
```

---

## テストの書き方

### テスト名は明確で具体的に

テスト名は、何をテストしているのかが明確にわかるようにします。

```typescript
// ✅ 良い例
test('ユーザーが存在しない場合、nullを返す', () => {
  expect(getUser('unknown')).toBeNull()
})

// ❌ 避けるべき例
test('getUser', () => {
  expect(getUser('unknown')).toBeNull()
})
```

### AAA パターン (Arrange-Act-Assert)

テストを3つのセクションに分けて構造化します。

```typescript
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

テストは焦点を絞り、1つの振る舞いや概念のみをテストするようにします。

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

// ❌ 避けるべき例
test('ユーザーが正しく作成される', () => {
  const user = createUser({ name: 'John', age: 30 })
  expect(user.name).toBe('John')
  expect(user.age).toBe(30)
  expect(user.isActive).toBe(true)
  expect(user.createdAt).toBeDefined()
})
```

### test.skip と test.only の適切な使用

- `test.skip`: 一時的にテストをスキップする（コードは削除しない）
- `test.only`: 特定のテストのみ実行してデバッグする

```typescript
// 一時的にテストをスキップ
test.skip('未実装の機能', () => {
  // TODO: 実装後にスキップを解除
})

// デバッグ中は特定のテストのみ実行
test.only('デバッグ中のテスト', () => {
  expect(debugFunction()).toBe(expectedValue)
})
```

⚠️ **注意**: `test.only`はコミット前に必ず削除してください。

---

## アサーション

### 適切なマッチャーの選択

Vitestは豊富なマッチャーを提供しています。適切なマッチャーを使用することで、エラーメッセージがより明確になります。

```typescript
// ✅ 良い例
expect(value).toBe(true)          // プリミティブの厳密な等価性
expect(obj).toEqual(expected)      // オブジェクトの深い等価性
expect(arr).toContain(item)        // 配列に要素が含まれるか
expect(str).toMatch(/pattern/)     // 文字列が正規表現にマッチするか
expect(fn).toThrow(Error)          // 関数がエラーをスローするか

// ❌ 避けるべき例
expect(value === true).toBe(true)  // 冗長
expect(JSON.stringify(obj)).toBe(JSON.stringify(expected))  // わかりにくい
```

### カスタムエラーメッセージ

複雑なテストでは、カスタムメッセージを追加して失敗時の理解を助けます。

```typescript
expect(
  calculateDiscount(price, percentage),
  `割引計算が正しくない: price=${price}, percentage=${percentage}`
).toBe(expectedDiscount)
```

### expect.softで複数のアサーションを実行

`expect.soft`を使用すると、最初のアサーション失敗後もテストを継続できます。

```typescript
test('expect.soft で複数のエラーを収集', () => {
  expect.soft(1 + 1).toBe(3) // 失敗してもテスト継続
  expect.soft(1 + 2).toBe(4) // 失敗してもテスト継続
  // すべてのエラーがレポートに表示される
})
```

---

## モック機能

### vi.fnでモック関数を作成

```typescript
import { expect, test, vi } from 'vitest'

test('コールバックが呼ばれることを確認', () => {
  const callback = vi.fn()

  processData(callback)

  expect(callback).toHaveBeenCalled()
  expect(callback).toHaveBeenCalledWith(expectedValue)
})
```

### vi.spyOnでメソッドをスパイ

```typescript
import { expect, test, vi } from 'vitest'
import * as utils from './utils'

test('関数が正しく呼ばれる', () => {
  const spy = vi.spyOn(utils, 'formatDate')

  const result = processDate(new Date())

  expect(spy).toHaveBeenCalledTimes(1)
  spy.mockRestore() // テスト後に元に戻す
})
```

### vi.mockでモジュール全体をモック

```typescript
import { test, vi } from 'vitest'

vi.mock('./api', () => ({
  fetchUser: vi.fn(() => Promise.resolve({ name: 'John' }))
}))

test('APIをモックする', async () => {
  const user = await fetchUser('123')
  expect(user.name).toBe('John')
})
```

### モックのクリーンアップ

テスト間でモックの状態が漏れないように、適切にクリーンアップします。

```typescript
import { afterEach, beforeEach, vi } from 'vitest'

beforeEach(() => {
  vi.clearAllMocks() // すべてのモックをクリア
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

---

## 非同期テスト

### async/awaitの使用

非同期テストには`async/await`を使用します。

```typescript
test('非同期データを取得する', async () => {
  const data = await fetchData()
  expect(data).toBeDefined()
})
```

### Promiseのアサーション

```typescript
test('Promiseが解決されることを確認', async () => {
  await expect(fetchUser()).resolves.toEqual({ name: 'John' })
})

test('Promiseが拒否されることを確認', async () => {
  await expect(fetchUser('invalid')).rejects.toThrow('User not found')
})
```

⚠️ **重要**: `resolves`と`rejects`を使用する場合は、必ず`await`してください。

### expect.pollでポーリング

条件が満たされるまで繰り返しチェックする場合は`expect.poll`を使用します。

```typescript
test('要素が表示されるまで待機', async () => {
  await expect.poll(() => document.querySelector('.element')).toBeTruthy()
})
```

### expect.assertionsで非同期アサーションを保証

非同期コード内でアサーションが呼ばれることを保証します。

```typescript
test('コールバック内のアサーション', async () => {
  expect.assertions(2)

  await doAsync((data) => {
    expect(data).toBeTruthy()
    expect(data.id).toBeDefined()
  })
})
```

---

## スナップショットテスト

### 基本的なスナップショット

```typescript
test('コンポーネントが正しくレンダリングされる', () => {
  const result = render(<MyComponent />)
  expect(result).toMatchSnapshot()
})
```

### インラインスナップショット

小さなスナップショットにはインラインスナップショットを使用します。

```typescript
test('データを正しくフォーマットする', () => {
  expect(formatData({ name: 'John' })).toMatchInlineSnapshot(`
    {
      "name": "John",
    }
  `)
})
```

### ファイルスナップショット

大きなHTMLやテキスト出力には、ファイルスナップショットを使用します。

```typescript
test('HTMLを正しく生成する', async () => {
  const html = renderHTML(component)
  await expect(html).toMatchFileSnapshot('./snapshots/output.html')
})
```

### スナップショットの更新

スナップショットを更新する必要がある場合:

```bash
vitest -u
# または watch モードで 'u' キーを押す
```

### スナップショットのベストプラクティス

- スナップショットは小さく、焦点を絞る
- 頻繁に変更される部分はスナップショットしない
- スナップショットの変更はレビュー時に注意深く確認
- 動的な値（日付、IDなど）は除外またはモックする

```typescript
test('動的な値を除外する', () => {
  const data = {
    id: generateId(),
    name: 'John',
    createdAt: new Date(),
  }

  expect(data).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  })
})
```

---

## カバレッジ

### カバレッジの有効化

```bash
vitest run --coverage
```

または設定ファイルで:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8', // または 'istanbul'
      reporter: ['text', 'html', 'json'],
      include: ['src/**/*.ts'],
      exclude: ['**/*.test.ts', '**/*.spec.ts'],
    },
  },
})
```

### カバレッジ目標の設定

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80,
      },
    },
  },
})
```

### カバレッジから除外

コメントを使用してカバレッジから特定のコードを除外できます。

```typescript
/* v8 ignore next 3 -- @preserve */
if (process.env.NODE_ENV === 'development') {
  console.log('Debug info')
}

/* istanbul ignore if -- @preserve */
if (condition) {
  // テスト不要なコード
}
```

⚠️ **注意**: TypeScriptを使用している場合は`@preserve`を追加してコメントが削除されないようにします。

---

## セットアップとクリーンアップ

### beforeEach と afterEach

各テストの前後で実行される処理を定義します。

```typescript
import { afterEach, beforeEach, describe, expect, test } from 'vitest'

describe('Database tests', () => {
  let db: Database

  beforeEach(async () => {
    // テスト前にデータベースをセットアップ
    db = await createTestDatabase()
    await db.seed()
  })

  afterEach(async () => {
    // テスト後にクリーンアップ
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

すべてのテストの実行前後で1回だけ実行される処理を定義します。

```typescript
import { afterAll, beforeAll, describe, test } from 'vitest'

describe('API tests', () => {
  let server: Server

  beforeAll(async () => {
    // すべてのテスト前にサーバーを起動
    server = await startTestServer()
  })

  afterAll(async () => {
    // すべてのテスト後にサーバーを停止
    await server.close()
  })

  test('エンドポイントにアクセスできる', async () => {
    const response = await fetch(`${server.url}/api/users`)
    expect(response.ok).toBe(true)
  })
})
```

### onTestFinished フック

テスト終了時にクリーンアップ処理を登録します。

```typescript
import { onTestFinished, test } from 'vitest'

test('リソースを自動クリーンアップ', async () => {
  const db = connectDatabase()
  onTestFinished(() => db.close())

  // テストロジック
  const result = await db.query('SELECT * FROM users')
  expect(result).toBeDefined()
})
```

---

## パフォーマンス最適化

### テストの並列実行

デフォルトでVitestはテストを並列実行しますが、特定のテストで並列実行を制御できます。

```typescript
// 並列実行
test.concurrent('並列テスト1', async () => {
  await someAsyncOperation()
})

test.concurrent('並列テスト2', async () => {
  await anotherAsyncOperation()
})

// 順次実行
test.sequential('順次テスト1', async () => {
  await sequentialOperation()
})
```

### テストの分離を無効化

副作用のないテストでは、`--no-isolate`フラグで分離を無効化してパフォーマンスを向上できます。

```bash
vitest --no-isolate
```

または設定ファイルで:

```typescript
export default defineConfig({
  test: {
    isolate: false,
  },
})
```

⚠️ **注意**: 分離を無効化すると、テスト間で状態が共有される可能性があります。

### プールの選択

```bash
# threads pool (デフォルト、高速だが互換性の問題がある場合あり)
vitest --pool=threads

# forks pool (互換性が高いが少し遅い)
vitest --pool=forks
```

### ワーカー数の調整

```bash
# 最大ワーカー数を指定
vitest --max-workers=4
```

---

## テスト環境とコンフィギュレーション

### テスト環境の設定

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    environment: 'node', // 'jsdom', 'happy-dom', 'edge-runtime'
  },
})
```

### グローバルAPIの有効化

```typescript
export default defineConfig({
  test: {
    globals: true, // describe, test, expect をインポート不要に
  },
})
```

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

### setupFilesの設定

テスト実行前に共通の設定を行います。

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    setupFiles: ['./tests/setup.ts'],
  },
})
```

```typescript
// tests/setup.ts
import { beforeAll } from 'vitest'

beforeAll(() => {
  // グローバルなセットアップ
  process.env.NODE_ENV = 'test'
})
```

### プロジェクト設定

複数の設定を持つプロジェクトでは、プロジェクト機能を使用します。

```typescript
export default defineConfig({
  test: {
    projects: [
      {
        name: 'unit',
        test: {
          include: ['src/**/*.test.ts'],
          environment: 'node',
        },
      },
      {
        name: 'integration',
        test: {
          include: ['tests/integration/**/*.test.ts'],
          environment: 'jsdom',
        },
      },
    ],
  },
})
```

### タイムアウトの設定

```typescript
export default defineConfig({
  test: {
    testTimeout: 10000, // 10秒
    hookTimeout: 10000,
  },
})
```

個別のテストでもタイムアウトを指定できます:

```typescript
test('長時間実行されるテスト', async () => {
  await longRunningOperation()
}, 30000) // 30秒
```

---UIコンポーネントのテスト

### React Testing Libraryとの統合

VitestはReact Testing Libraryと完全に互換性があり、Reactコンポーネントのテストに最適です。

```typescript
import { render, screen } from '@testing-library/react'
import { describe, expect, test } from 'vitest'
import userEvent from '@testing-library/user-event'
import MyButton from './MyButton'

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

### ブラウザモードでのテスト

実際のブラウザ環境でテストを実行する場合:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    browser: {
      enabled: true,
      name: 'chromium', // または 'firefox', 'webkit'
      provider: 'playwright', // または 'webdriverio'
      headless: true,
    },
  },
})
```

```typescript
import { page } from '@vitest/browser/context'
import { expect, test } from 'vitest'

test('ボタンをクリックすると状態が変わる', async () => {
  await page.goto('http://localhost:3000')
  const button = await page.getByRole('button')
  await button.click()
  await expect(button).toHaveText('Clicked!')
})
```

### Vue Testing Libraryとの統合

```typescript
import { render } from '@testing-library/vue'
import { describe, expect, test } from 'vitest'
import MyComponent from './MyComponent.vue'

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

### コンポーネントスナップショット

```typescript
test('コンポーネントのスナップショット', () => {
  const { container } = render(<MyComponent />)
  expect(container).toMatchSnapshot()
})
```

---

## よくある落とし穴と回避策

### 1. 非同期テストでのawait忘れ

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

### 2. test.onlyのコミット

```typescript
// ❌ 間違い: コミットしないこと
test.only('デバッグ中', () => {
  // ...
})

// ✅ 正しい: コミット前にonlyを削除
test('デバッグ中', () => {
  // ...
})
```

### 3. モックのクリーンアップ忘れ

```typescript
// ❌ 間違い
test('テスト1', () => {
  vi.mock('./module')
  // モックのクリーンアップなし
})

// ✅ 正しい
afterEach(() => {
  vi.clearAllMocks()
  vi.resetAllMocks()
})
```

### 4. グローバルな状態の変更

```typescript
// ❌ 間違い
let counter = 0
test('カウンター増加', () => {
  counter++
  expect(counter).toBe(1)
})
test('カウンター増加2', () => {
  counter++ // 前のテストの影響を受ける
  expect(counter).toBe(1) // 失敗する
})

// ✅ 正しい
describe('カウンターテスト', () => {
  let counter: number

  beforeEach(() => {
    counter = 0 // 各テスト前にリセット
  })

  test('カウンター増加', () => {
    counter++
    expect(counter).toBe(1)
  })

  test('カウンター増加2', () => {
    counter++
    expect(counter).toBe(1)
  })
})
```

### 5. 過度に具体的なスナップショット

```typescript
// ❌ 避けるべき: 動的な値を含む
test('ユーザー情報', () => {
  const user = {
    id: generateId(), // 毎回変わる
    createdAt: new Date(),
    name: 'John',
  }
  expect(user).toMatchSnapshot() // 毎回失敗する
})

// ✅ 正しい: 動的な値を除外
test('ユーザー情報', () => {
  const user = {
    id: generateId(),
    createdAt: new Date(),
    name: 'John',
  }
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  })
})
```

### 6. 並行テストでのグローバルexpect使用

```typescript
// ❌ 間違い: 並行テストでグローバルexpectを使用
test.concurrent('テスト1', async () => {
  expect(await getData()).toMatchSnapshot() // 正しいテストを追跡できない
})

// ✅ 正しい: テストコンテキストのexpectを使用
test.concurrent('テスト1', async ({ expect }) => {
  expect(await getData()).toMatchSnapshot()
})
```

### 7. テストの順序依存

```typescript
// ❌ 間違い: テストが順序に依存
let data: Data

test('データを作成', () => {
  data = createData()
  expect(data).toBeDefined()
})

test('データを更新', () => {
  // 前のテストに依存している
  updateData(data)
  expect(data.updated).toBe(true)
})

// ✅ 正しい: 各テストが独立
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

### 8. エラーメッセージの不足

```typescript
// ❌ 避けるべき
test('複雑な計算', () => {
  expect(complexCalculation(a, b, c)).toBe(expected)
})

// ✅ 正しい: 意味のあるエラーメッセージを追加
test('複雑な計算', () => {
  const result = complexCalculation(a, b, c)
  expect(result, `Failed for a=${a}, b=${b}, c=${c}`).toBe(expected)
})
```

---

## その他のベストプラクティス

### テストコンテキストの活用

```typescript
import { test } from 'vitest'

test('コンテキストを使用', ({ expect, task }) => {
  console.log(`テスト名: ${task.name}`)
  expect(2 + 2).toBe(4)
})
```

### エラーハンドリング

```typescript
test('エラーが正しくスローされる', () => {
  expect(() => {
    throw new Error('エラーメッセージ')
  }).toThrow('エラーメッセージ')
})

test('特定のエラータイプをチェック', () => {
  expect(() => {
    throw new TypeError('型エラー')
  }).toThrow(TypeError)
})

test('エラーオブジェクトの詳細を検証', () => {
  expect(() => {
    throw new ValidationError({ field: 'email', message: 'Invalid email' })
  }).toThrow(expect.objectContaining({
    field: 'email',
    message: expect.stringContaining('email'),
  }))
})
```

### 型のテスト

```typescript
import { assertType, expectTypeOf, test } from 'vitest'

test('型が正しい', () => {
  expectTypeOf<number>().toBeNumber()
  expectTypeOf({ a: 1 }).toMatchTypeOf<{ a: number }>()
  expectTypeOf({ a: 1, b: 2 }).toExtend<{ a: number }>()
})

test('関数の型をテスト', () => {
  function greet(name: string): string {
    return `Hello, ${name}`
  }

  expectTypeOf(greet).parameter(0).toBeString()
  expectTypeOf(greet).returns.toBeString()
})

test('型エラーをアサート', () => {
  // @ts-expect-error - 意図的に型エラーを起こす
  const num: number = 'string'

  assertType<string>('valid string')
})
```

### デバッグとトラブルシューティング

```typescript
test('デバッグ情報を出力', () => {
  const result = complexCalculation()
  console.log('Result:', result) // テスト実行時に表示
  expect(result).toBeGreaterThan(0)
})

// VS Codeでデバッグ
test('デバッガーで停止', () => {
  const data = getData()
  debugger // ここでブレークポイント
  expect(data).toBeDefined()
})
```

### テストのドキュメント化

```typescript
/**
 * ユーザー認証機能のテスト
 *
 * このテストスイートは以下を検証します:
 * - 有効な認証情報での認証成功
 * - 無効な認証情報での認証失敗
 * - セッションの有効期限管理
 */
describe('User Authentication', () => {
  test('有効な認証情報で認証成功', () => {
    // Given: 有効なユーザー情報
    const credentials = { username: 'user', password: 'pass' }

    // When: 認証を試みる
    const result = authenticate(credentials)

    // Then: 認証に成功する
    expect(result.success).toBe(true)
    expect(result.token).toBeDefined()
  }n()
  console.log('Result:', result) // テスト実行時に表示
  expect(result).toBeGreaterThan(0)
})
```

---

## まとめ

### 重要なポイント

1. **テストは明確で焦点を絞る** - 1テスト1概念
2. **適切なマッチャーを使用** - エラーメッセージをわかりやすく
3. **モックは適切にクリーンアップ** - テスト間の状態漏れを防ぐ
4. **非同期処理は必ずawait** - 特にresolves/rejects使用時
5. **スナップショットは慎重に** - 変更時は必ずレビュー
6. **カバレッジは目安** - 100%を目指すより質を重視
7. **セットアップ/クリーンアップは確実に** - リソースリークを防ぐ
8. **パフォーマンスを意識** - 並列実行や分離設定を活用

### 推奨設定例

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'json'],
      include: ['src/**/*.ts'],
      exclude: ['**/*.test.ts', '**/*.spec.ts', '**/types/**'],
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80,
      },
    },
    clearMocks: true,
    restoreMocks: true,
    unstubEnvs: true,
    testTimeout: 10000,
  },
})
```
