---
description: '安全で保守性の高いTypeScriptコードを書くためのベストプラクティス規則'
applyTo: '**/*.ts, **/*.tsx'
---

# TypeScript ベストプラクティス

## 目的と範囲

TypeScriptコードを生成・変更する際、型安全性・保守性・変更容易性を確保するためのガイドラインである。

## 基本原則

- 公開API(エクスポートされる関数・クラス・型)には型を明示的に指定する
- 実装の詳細を型として公開しない
- 型推論を活用し、ローカル変数への冗長な型注釈は避ける
- 型の複雑さよりも可読性を優先する

## 型安全性

### `any`を避ける

- `any`は使用しない
- 不確実な値(外部入力、JSON、環境変数など)は`unknown`として扱う
- 型ガードまたはnarrowingで安全に処理する

**理由**: `any`は型チェックを無効化し、ランタイムエラーのリスクを高めるためである

```ts
// 推奨
function parseJson(text: string): unknown {
  return JSON.parse(text)
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null
}

const value = parseJson('{"x": 1}')
if (isRecord(value) && typeof value.x === 'number') {
  console.log(value.x)
}
```

```ts
// 非推奨
function parseJson(text: string): any {
  return JSON.parse(text)
}
```

### Union型とNarrowingをデフォルトとする

- 条件分岐を伴うデータには判別可能なUnion型を使用する
- `switch`と`never`を使って分岐の網羅性をチェックする

**理由**: コンパイラが未処理のケースを検出し、リファクタリング時の漏れを防ぐためである

```ts
// 推奨
type Result =
  | { kind: 'ok'; value: string }
  | { kind: 'err'; message: string }

export function unwrap(result: Result): string {
  switch (result.kind) {
    case 'ok':
      return result.value
    case 'err':
      throw new Error(result.message)
    default: {
      const _exhaustive: never = result
      return _exhaustive
    }
  }
}
```

### Genericsは最小限にする

- 型パラメータを多数追加しない
- Union型や具体的な型で表現できる場合はGenericsを導入しない
- 必要な場合は`extends`で制約を付ける

**理由**: 過度なGenericsは理解とメンテナンスのコストを増大させるためである

```ts
// 推奨
export function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

### Utility型を優先する

- 標準のUtility型(`Partial` / `Required` / `Readonly` / `Pick` / `Omit` / `Record`)を使用する
- 用途ごとに型を分離し(create, update, save)、型全体を`Partial`にしない

**理由**: 標準の型は広く認識されており、意図が明確に伝わるためである

```ts
// 推奨
type User = { id: string; name: string; email: string }
type UserPatch = Partial<Pick<User, 'name' | 'email'>>
```

### モジュール境界と`import type`

- 型のみのインポートには`import type`を使用する
- ランタイム依存を増やさない

**理由**: 循環依存を防ぎ、バンドルサイズを削減するためである

```ts
// 推奨
import type { User } from './types'

export function greet(user: User): string {
  return `Hello, ${user.name}`
}
```

### 型ガードを活用する

- カスタム型ガード(`is`)を使い、型を明示的に絞り込む
- `Array.isArray()`や`typeof`などの組み込みガードを優先する

**理由**: 型安全性を保ちつつ、条件分岐を明確にするためである

```ts
// 推奨
function isString(value: unknown): value is string {
  return typeof value === 'string'
}

function processInput(input: unknown): string {
  if (isString(input)) {
    return input.trim()
  }
  throw new Error('Expected string')
}
```

```ts
// 非推奨
function processInput(input: unknown): string {
  return (input as string).trim()
}
```

### 型アサーション(`as`)と非null アサーション(`!`)を多用しない

- `as`は最後の手段として扱う
- 可能な限り型ガードで置き換える
- 外部入力、環境変数、I/Oに対して`x!`を使わない

**理由**: アサーションは型チェックを回避し、ランタイムエラーを引き起こすためである

```ts
// 推奨
function processValue(value: unknown): string {
  if (typeof value === 'string') {
    return value.toUpperCase()
  }
  throw new Error('Expected string')
}

// 非推奨
function processValue(value: unknown): string {
  return (value as string).toUpperCase()
}
```

## TypeScript設定

### 推奨される`tsconfig.json`設定

- `strict: true`を有効化する(段階的な移行も可)
- `useUnknownInCatchVariables: true`で例外処理を安全に保つ
- `noUncheckedIndexedAccess: true`でインデックスアクセスの見落としを防ぐ
- `noUnusedLocals` / `noUnusedParameters`で不要なコードを早期に除去する

**理由**: 厳格な型チェックにより、バグを早期発見し、保守コストを削減するためである

```json
{
  "compilerOptions": {
    "strict": true,
    "useUnknownInCatchVariables": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

### ESLint / typescript-eslint

- `tsc`で検出できない問題をESLintで補完する
- `typescript-eslint`の共有設定をベースにする
  - 型情報なし: `recommended` / `stylistic`
  - 型情報あり: `recommended-type-checked` / `stylistic-type-checked`

**注意点**:
- `@typescript-eslint/consistent-type-imports`とTypeScriptの`verbatimModuleSyntax`は競合する可能性があるため、どちらかに統一する
- レガシーデコレータ(`experimentalDecorators`と`emitDecoratorMetadata`)を使用する場合は、`consistent-type-imports`の注意事項を確認する

## 検証コマンド

### 型チェック

```bash
tsc --noEmit
```

### 設定確認

```bash
tsc --showConfig
```

### Lint(統合している場合)

```bash
eslint .
```

## エラーハンドリング

### 型安全な例外処理

- `catch`節では`unknown`型を扱う(`useUnknownInCatchVariables: true`)
- エラーの型を判定してから処理する

**理由**: JavaScriptではあらゆる値がthrowされる可能性があるため、型を仮定せずに処理するためである

```ts
// 推奨
try {
  await fetchData()
} catch (error: unknown) {
  if (error instanceof Error) {
    console.error(error.message)
  } else {
    console.error('Unknown error occurred')
  }
}
```

```ts
// 非推奨
try {
  await fetchData()
} catch (error) {
  console.error(error.message) // errorは'any'
}
```

### Result型パターンの活用

- 例外ではなく、Result型で成功・失敗を表現する
- 呼び出し側に明示的なエラー処理を強制する

**理由**: エラーハンドリングの漏れをコンパイル時に検出するためである

```ts
// 推奨
type Result<T, E> =
  | { success: true; value: T }
  | { success: false; error: E }

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { success: false, error: 'Division by zero' }
  }
  return { success: true, value: a / b }
}

const result = divide(10, 2)
if (result.success) {
  console.log(result.value)
} else {
  console.error(result.error)
}
```

## 命名規則

### 一般的な命名パターン

- 変数・関数: `camelCase`
- 型・インターフェース・クラス: `PascalCase`
- 定数: `UPPER_SNAKE_CASE`または`camelCase`
- プライベートプロパティ: `#privateField`または`_privateField`

**理由**: 一貫した命名規則によりコードの可読性が向上するためである

```ts
// 推奨
const MAX_RETRY_COUNT = 3
type UserProfile = { id: string; name: string }

class DatabaseConnection {
  #connectionString: string
  
  constructor(connectionString: string) {
    this.#connectionString = connectionString
  }
}

function fetchUserData(userId: string): Promise<UserProfile> {
  // ...
}
```

### 型名のPrefix・Suffix

- インターフェースに`I`プレフィックスを付けない(`IUser` → `User`)
- 型名は具体的で説明的なものにする
- Props型には`Props`サフィックスを付ける(Reactの場合)

```ts
// 推奨
interface User {
  id: string
  name: string
}

type ButtonProps = {
  label: string
  onClick: () => void
}
```

```ts
// 非推奨
interface IUser {
  id: string
  name: string
}
```

## パフォーマンス最適化

### 型計算のパフォーマンス

- 過度に複雑な型定義を避ける
- `infer`や再帰型の使用は最小限にする
- 型の計算コストが高い場合は具体的な型に置き換える

**理由**: 複雑な型はコンパイル時間を増大させるためである

```ts
// 推奨: シンプルな型定義
type ApiResponse = {
  data: unknown
  status: number
  headers: Record<string, string>
}

// 非推奨: 過度に複雑な型
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T
```

### `const`アサーションを活用する

- `as const`でリテラル型を保持する
- オブジェクトや配列の不変性を表現する

**理由**: より具体的な型推論が可能になり、意図しない変更を防ぐためである

```ts
// 推奨
const COLORS = ['red', 'green', 'blue'] as const
type Color = typeof COLORS[number] // 'red' | 'green' | 'blue'

const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000
} as const
```

```ts
// 非推奨
const COLORS = ['red', 'green', 'blue'] // string[]
```

## 参考資料

- TypeScript Handbook: https://www.typescriptlang.org/docs/handbook/intro.html
- TSConfig Reference: https://www.typescriptlang.org/tsconfig/
- TSConfig `noUncheckedIndexedAccess`: https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html
- typescript-eslint Shared Configs: https://typescript-eslint.io/users/configs/
- typescript-eslint `consistent-type-imports`: https://typescript-eslint.io/rules/consistent-type-imports/
- Changes to consistent-type-imports with decorators: https://typescript-eslint.io/blog/changes-to-consistent-type-imports-with-decorators/
