# TypeScript ベストプラクティス

このドキュメントは、TypeScript 公式ドキュメントに基づいた、実践的な TypeScript コーディングのベストプラクティスをまとめたものです。

## 基本方針（型設計の考え方）

- **型は「利用者が安全に使える最小限」を公開**し、実装詳細は隠す
- **型推論を活かしつつ、境界（入出力・公開API）に型注釈を置く**
- **ユニオン + ナローイング**を基本にし、無理に複雑なジェネリクスで表現しない
- 迷ったら「読み手が追えるか？」を優先する（型は保守コスト）

## `any` を避ける（`unknown` とナローイングを使う）

- **`any` は最後の手段**として扱う
  - `any` は型チェックを実質無効化し、誤りが伝播しやすい
- 外部入力や不確かな値はまず **`unknown`** として受け、**型ガード**や **`in` / `instanceof` / `typeof`** などで絞り込む

```ts
function parseJson(text: string): unknown {
  return JSON.parse(text)
}

const value = parseJson('{"x": 1}')
if (typeof value === 'object' && value !== null && 'x' in value) {
  // value は object まで絞れた（ここから先は必要に応じて更に絞る）
}
```

## `null` / `undefined` を前提にする（`strictNullChecks` 想定）

- `null` / `undefined` は **明示的に扱う**
- 省略可能（optional）と、値が存在しない（`undefined`）を混同しない
- truthiness チェックは便利だが、**0 / "" などの値**で意図せず落ちる可能性があるため注意する

```ts
function printLength(value?: string) {
  // value が空文字の可能性があるなら、truthy チェックだけに頼らない
  if (value === undefined) return
  console.log(value.length)
}
```

## ユニオン型とナローイング（判別可能ユニオンを基本にする）

- 条件分岐のあるデータは **判別可能ユニオン（discriminated union）** を基本形にする
- `switch` と `never` を使った **網羅性チェック**で分岐漏れを検知する

```ts
type Result =
  | { kind: 'ok'; value: string }
  | { kind: 'err'; message: string }

function handle(result: Result) {
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

## オブジェクト型の設計

- オブジェクトの形を表現する際、**必要なプロパティだけを持つ型**を定義する
- 変更してはいけない値は **`readonly`** を検討する
- リテラル型を活かしたい定数は **`as const`** を検討する

```ts
const status = {
  ready: 'ready',
  running: 'running',
} as const

type Status = (typeof status)[keyof typeof status]
```

## 関数（引数・コールバック・オーバーロード）

### コールバックの戻り値は原則 `void`

- コールバックの戻り値を利用しない設計なら、**戻り値を `void`** にする
  - 呼び出し側が「戻り値を使うべき」と誤解しにくい

```ts
function forEach<T>(items: T[], callback: (item: T) => void) {
  for (const item of items) callback(item)
}
```

### オーバーロードよりユニオンを優先（必要なときだけオーバーロード）

- 可能なら **ユニオン型**で表現し、オーバーロードは本当に必要な場合に限定する

```ts
function format(value: string | number) {
  return typeof value === 'string' ? value : value.toFixed(2)
}
```

## ジェネリクス（型パラメータを増やしすぎない）

- 型パラメータは**増やしすぎない**
- 型パラメータが不要なら導入しない（ユニオン型や具体型で十分なことが多い）
- 必要な場合は **制約（`extends`）** を付けて利用範囲を明確にする

```ts
function first<T>(values: T[]): T | undefined {
  return values[0]
}

function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

## Utility Types を適切に使う

- 標準のユーティリティ型（例: `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record` など）を優先する
- 「便利だから全部 `Partial`」ではなく、**意図（入力・更新・保存）に合わせて型を分ける**

```ts
type User = { id: string; name: string; email: string }

type UserPatch = Partial<Pick<User, 'name' | 'email'>>
```

## モジュール（type-only import とモジュール境界）

- 型だけを読み込む場合は **`import type`** を使う
  - 実行時依存を増やさず、ビルド出力や循環依存の誤解を減らす
- ファイルが「モジュール」として扱われることが前提なら、必要に応じて `export {}` を置く

```ts
import type { User } from './types'

export function greet(user: User) {
  return `Hello, ${user.name}`
}
```

## 静的解析（ESLint / typescript-eslint）

- `tsc` の型チェックだけでは拾いにくい問題（Promiseの取りこぼし、危険な型アサーション、importの一貫性など）は、ESLint で補う
- TypeScript 向けの ESLint 設定は `typescript-eslint` の共有設定（recommended / recommended-type-checked など）から始める

### 型情報ありのLint（type-aware linting）を有効化する

- `typescript-eslint` のドキュメントでは、型情報を用いるルールを使う場合に `recommended-type-checked` などの設定を推奨している
- 型情報を使うLintはコストが上がるため、`**/*.js` を対象外にするなど運用で調整する

例（Legacy config / `.eslintrc.*` のイメージ）:

```js
module.exports = {
  root: true,
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended-type-checked',
  ],
  parserOptions: {
    projectService: true,
    tsconfigRootDir: __dirname,
  },
  overrides: [
    {
      files: ['**/*.js'],
      extends: ['plugin:@typescript-eslint/disable-type-checked'],
    },
  ],
}
```

### `import type` の運用を自動化する

- `@typescript-eslint/consistent-type-imports` は、型としてのみ使われる import を `import type` に揃えるためのルール
- TypeScript 5 以降には `verbatimModuleSyntax` があり、両方を同時に有効にすると競合しうるため、原則どちらか片方に寄せる
- 旧デコレータ（`experimentalDecorators` と `emitDecoratorMetadata`）の組み合わせでは、`consistent-type-imports` の自動修正が実行時挙動に影響しうる注意点がある

## 型アサーション（`as`）と non-null アサーション（`!`）は最小化する

- `as` は「コンパイラが知らない事実」を補う道具だが、濫用すると型チェックの恩恵を失う
- `x!`（non-null アサーション）は `undefined` を握りつぶすため、外部入力や環境変数など不確かな値には特に注意する
- 可能なら **`unknown` で受けてナローイング**し、どうしても必要な場所だけにアサーションを閉じ込める

```ts
type User = { id: string }

function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'id' in value
}

function readUser(input: unknown): User {
  if (!isUser(input)) throw new Error('invalid')
  return input
}
```

## `tsconfig.json` とコンパイル運用

- まずは **`tsc --showConfig`** などで「実際に適用される設定」を確認する
- `tsconfig.json` の探索規則（どの `tsconfig.json` が使われるか）を理解した上で、ワークスペース構成を決める
- `extends` を使って共通設定を再利用する

### 追加で検討したい `tsconfig` オプション

- `strict: true` をベースに、必要に応じて段階的に厳格化する
- `noUncheckedIndexedAccess: true` を検討する
  - インデックスアクセスや未宣言プロパティの参照に `undefined` が付くため、境界での取りこぼしを減らせる
- `useUnknownInCatchVariables: true` を検討する
  - `catch` 変数を `unknown` として扱い、例外の取り扱いを安全側に寄せられる
- `noUnusedLocals` / `noUnusedParameters` を有効化し、不要コードを早めに落とす

### Project References（大規模化したら検討）

- 大規模リポジトリでは **Project References** を使い、`tsc -b` によるビルドを検討する
- 参照される側は `composite: true` が必要

## ビルドツール統合時の注意（型チェックの責務を分ける）

- ツールによっては **型チェックを実行しない**（例: 変換のみを行う構成）
- 「ビルド」と「型チェック」を分離し、CI で `tsc --noEmit` などを実行する運用を検討する

## 宣言ファイル（`.d.ts`）と型公開の注意点

- ライブラリ公開時は、宣言ファイルのガイド（Do’s and Don’ts）に従って **利用者が誤用しにくい型**を提供する
- **boxed types（`String`, `Number`, `Boolean` など）を使わない**（プリミティブの `string`, `number`, `boolean` を使う）

## エラーの読み方（Understanding Errors）

- エラーを「どこで期待される型が決まったか」まで辿って読む
- まずは最小再現にして、ユニオンの絞り込み・ジェネリクス・推論のどこでズレたかを切り分ける

## 参考（公式ドキュメント）

- Handbook（ガイド全体）: https://www.typescriptlang.org/docs/handbook/intro.html
- Everyday Types: https://www.typescriptlang.org/docs/handbook/2/everyday-types.html
- Narrowing: https://www.typescriptlang.org/docs/handbook/2/narrowing.html
- More on Functions: https://www.typescriptlang.org/docs/handbook/2/functions.html
- Utility Types: https://www.typescriptlang.org/docs/handbook/utility-types.html
- Modules: https://www.typescriptlang.org/docs/handbook/modules.html
- TSConfig Reference: https://www.typescriptlang.org/tsconfig/
- tsc CLI Options: https://www.typescriptlang.org/docs/handbook/compiler-options.html
- Project References: https://www.typescriptlang.org/docs/handbook/project-references.html
- Declaration Files Do’s and Don’ts: https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html
- Integrating with Build Tools: https://www.typescriptlang.org/docs/handbook/integrating-with-build-tools.html
- Understanding Errors: https://www.typescriptlang.org/docs/handbook/understanding-errors.html

## 参考（一般知見）

- typescript-eslint Shared Configs: https://typescript-eslint.io/users/configs/
- typescript-eslint `consistent-type-imports`: https://typescript-eslint.io/rules/consistent-type-imports/
- Changes to consistent-type-imports with decorators: https://typescript-eslint.io/blog/changes-to-consistent-type-imports-with-decorators/
- TSConfig `noUncheckedIndexedAccess`: https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html
