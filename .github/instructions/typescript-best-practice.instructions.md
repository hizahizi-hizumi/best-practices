---
description: 'TypeScript で安全・保守性の高いコードを書くためのベストプラクティス規則'
applyTo: '**/*.ts, **/*.tsx'
---

# TypeScript ベストプラクティス

GitHub Copilot が TypeScript コードを生成・修正する際に従うべきガイドラインです。型安全性、保守性、変更容易性を優先してください。

## 基本方針

- 公開API（外部に見える関数・クラス・型）では、型を明確にする
- 実装詳細は型として公開しない（利用者が依存しない設計にする）
- 型推論を活かし、局所変数の冗長な型注釈は避ける
- 型の複雑さより可読性を優先する（読み手が追える型設計にする）

## `any` を避ける

- `any` は原則使わない
- 不確かな値（外部入力、JSON、環境変数など）はまず `unknown` として受け、型ガードやナローイングで安全に扱う

### Good Example

```ts
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

### Bad Example

```ts
function parseJson(text: string): any {
  return JSON.parse(text)
}
```

## ユニオン型とナローイングを基本にする

- 条件分岐のあるデータは判別可能ユニオン（discriminated union）を優先する
- `switch` と `never` を使って分岐の網羅性チェックを行う

```ts
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

## ジェネリクスは最小限にする

- 型パラメータは増やしすぎない
- ユニオンや具体型で表現できるなら、ジェネリクスを導入しない
- 必要な場合は `extends` による制約を付ける

```ts
export function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

## Utility Types を優先する

- `Partial` / `Required` / `Readonly` / `Pick` / `Omit` / `Record` など標準ユーティリティ型を優先する
- 「全部 `Partial`」にせず、用途（作成、更新、保存）に合わせて型を分ける

```ts
type User = { id: string; name: string; email: string }

type UserPatch = Partial<Pick<User, 'name' | 'email'>>
```

## モジュール境界と `import type`

- 型だけを参照する import は `import type` を使う
- ランタイム依存を増やさない（循環依存を起こしにくくする）

```ts
import type { User } from './types'

export function greet(user: User): string {
  return `Hello, ${user.name}`
}
```

## 型アサーション（`as`）と non-null アサーション（`!`）を濫用しない

- `as` は最後の手段として扱い、可能な限り型ガードで置き換える
- `x!` は `undefined` を握りつぶすため、外部入力・環境変数・I/O では特に避ける

## `tsconfig`（型チェックの品質）

- `strict: true` を前提に設計する（必要なら段階的に移行する）
- `useUnknownInCatchVariables: true` を検討し、例外処理を安全側に寄せる
- `noUncheckedIndexedAccess: true` を検討し、インデックスアクセスの取りこぼしを減らす
- `noUnusedLocals` / `noUnusedParameters` を有効化し、不要コードを早めに除去する

## ESLint / typescript-eslint（一般知見）

- `tsc` だけでは拾いにくい問題を ESLint で補う
- `typescript-eslint` の共有設定をベースにする
  - 型情報なし: `recommended` / `stylistic`
  - 型情報あり: `recommended-type-checked` / `stylistic-type-checked`

注意:
- `@typescript-eslint/consistent-type-imports` と TypeScript の `verbatimModuleSyntax` は競合しうるため、原則どちらか片方に寄せる
- 旧デコレータ（`experimentalDecorators` と `emitDecoratorMetadata`）利用時は `consistent-type-imports` の注意事項を確認する

## 検証（推奨）

- 型チェック: `tsc --noEmit`
- 設定確認: `tsc --showConfig`
- （導入している場合）Lint: `eslint .`

## 参考

- TypeScript Handbook: https://www.typescriptlang.org/docs/handbook/intro.html
- TSConfig Reference: https://www.typescriptlang.org/tsconfig/
- TSConfig `noUncheckedIndexedAccess`: https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html
- typescript-eslint Shared Configs: https://typescript-eslint.io/users/configs/
- typescript-eslint `consistent-type-imports`: https://typescript-eslint.io/rules/consistent-type-imports/
- Changes to consistent-type-imports with decorators: https://typescript-eslint.io/blog/changes-to-consistent-type-imports-with-decorators/
