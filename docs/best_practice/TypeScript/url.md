# TypeScript ベストプラクティス関連URL集

TypeScript公式ドキュメント（ https://www.typescriptlang.org/docs/ 配下）から抽出した、ベストプラクティス作成に参考になるページのURL一覧です。

## 入口・全体像

### Handbook
- **URL**: https://www.typescriptlang.org/docs/handbook/
- **説明**: Handbookの全体構造（目次）を把握でき、必要な章に素早く辿れる

## 型・言語機能（安全な書き方の基礎）

### Everyday Types
- **URL**: https://www.typescriptlang.org/docs/handbook/2/everyday-types.html
- **説明**: 基本型・ユニオン/リテラル・`any` 等の「日常的に使う型」の前提を押さえられ、型設計の共通言語になる

### Narrowing
- **URL**: https://www.typescriptlang.org/docs/handbook/2/narrowing.html
- **説明**: 条件分岐・型ガード・判別可能ユニオン等による絞り込みを体系的に確認でき、実行時安全性と型安全性を一致させやすい

### More on Functions
- **URL**: https://www.typescriptlang.org/docs/handbook/2/functions.html
- **説明**: 引数/戻り値・オーバーロード・コールバック等の型付け方針を整理でき、API設計の事故（曖昧なシグネチャ）を減らせる

### More on Objects
- **URL**: https://www.typescriptlang.org/docs/handbook/2/objects.html
- **説明**: オブジェクト型、オプショナル、readonly、インデックス等の扱いを確認でき、ドメインモデルの型表現に直結する

### Generics
- **URL**: https://www.typescriptlang.org/docs/handbook/2/generics.html
- **説明**: 再利用可能で型安全なAPIを作るためのジェネリクス設計（制約/推論/デフォルト）を整理できる

### Creating Types from Types
- **URL**: https://www.typescriptlang.org/docs/handbook/2/types-from-types.html
- **説明**: `keyof`/`typeof`/条件型/マップ型など、型変換の道具箱を体系化でき「型のコピペ地獄」を避けやすい

### Utility Types
- **URL**: https://www.typescriptlang.org/docs/handbook/utility-types.html
- **説明**: `Partial`/`Pick`/`Omit` 等の標準ユーティリティ型の使いどころを揃え、型の再利用性と可読性を上げられる

### Type Inference
- **URL**: https://www.typescriptlang.org/docs/handbook/type-inference.html
- **説明**: 「注釈を省く/書く」の判断基準を作りやすく、冗長な型注釈を減らせる

### Type Compatibility
- **URL**: https://www.typescriptlang.org/docs/handbook/type-compatibility.html
- **説明**: 構造的型付けの互換性ルールを理解でき、代入・引数渡しの想定外エラーを減らせる

### Understanding Errors
- **URL**: https://www.typescriptlang.org/docs/handbook/2/understanding-errors.html
- **説明**: エラーメッセージの読み方の共通ルールを揃え、レビューやトラブルシュートを速くできる

### Type Declarations
- **URL**: https://www.typescriptlang.org/docs/handbook/2/type-declarations.html
- **説明**: 型宣言（注釈）の考え方を整理でき、型の“置き場所”をチームで統一しやすい

### Variable Declarations
- **URL**: https://www.typescriptlang.org/docs/handbook/variable-declarations.html
- **説明**: `var`/`let`/`const` の違いと落とし穴の一次情報として参照できる

### Declaration Merging
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-merging.html
- **説明**: 宣言のマージ（拡張）を安全に扱うための前提がまとまり、型汚染を避けやすい

## tsconfig / strictness（設定で安全性を担保）

### What is a tsconfig.json
- **URL**: https://www.typescriptlang.org/docs/handbook/tsconfig-json.html
- **説明**: `tsconfig.json` の探索規則、`files`/`include`/`exclude`、`extends` 等の基本を押さえ、設定の意図を明文化しやすい

### tsc CLI Options
- **URL**: https://www.typescriptlang.org/docs/handbook/compiler-options.html
- **説明**: `--project/-p`、`--build/-b`、`--showConfig` 等を含むCLI全体像と主要オプションを確認でき、CI/ローカルの差分を減らせる

## プロジェクト構成（大規模化・モノレポ）

### Project References
- **URL**: https://www.typescriptlang.org/docs/handbook/project-references.html
- **説明**: `references` と `composite`、`tsc -b` による増分ビルドなどを理解でき、依存関係がある複数tsconfig運用の基本になる

## モジュール設計・ESM/CJS 相互運用

### Modules - Introduction
- **URL**: https://www.typescriptlang.org/docs/handbook/modules/introduction.html
- **説明**: Modulesドキュメントの全体構造（theory/guides/reference/appendices）への入口で、必要な章に迷わず辿れる

### Modules - Theory
- **URL**: https://www.typescriptlang.org/docs/handbook/modules/theory.html
- **説明**: 「ホスト（Node/バンドラ）」と`module`/`moduleResolution`の関係をモデル化して説明しており、設定ミス由来の実行時エラーを減らせる

### Modules - Choosing Compiler Options
- **URL**: https://www.typescriptlang.org/docs/handbook/modules/guides/choosing-compiler-options.html
- **説明**: 「バンドラ」「Nodeで実行」「ライブラリ」等の前提ごとに推奨設定の考え方が整理され、環境に合わない`module`設定を避けられる

### Modules - Reference
- **URL**: https://www.typescriptlang.org/docs/handbook/modules/reference.html
- **説明**: `module`/`moduleResolution` の仕様と落とし穴（例: `paths` は emit を変えない等）を確認でき、移植性の高い構成にしやすい

### ESM/CJS Interoperability
- **URL**: https://www.typescriptlang.org/docs/handbook/modules/appendices/esm-cjs-interop.html
- **説明**: Nodeとバンドラで異なる相互運用ルールや `esModuleInterop`/`verbatimModuleSyntax` などの意図を把握でき、importの事故を防ぎやすい

## 型宣言（.d.ts）と公開/利用

### Declaration Files - Introduction
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html
- **説明**: 宣言ファイルの役割と章構成の入口で、型を「配布する/消費する」観点の全体像を掴める

### Declaration Files - Do’s and Don’ts
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html
- **説明**: `any` 回避や boxed types 回避など、利用者の型安全性を壊しやすいパターンを明確に避けられる

### Declaration Files - Library Structures
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html
- **説明**: 代表的なライブラリ構造ごとの宣言の書き方を整理でき、公開APIの型設計を現実に合わせやすい

### Declaration Files - Publishing
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html
- **説明**: 公開パッケージに型を同梱する際の作法を把握でき、利用者側で「型が見つからない」問題を減らせる

### Declaration Files - Consumption
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-files/consumption.html
- **説明**: 宣言ファイルの取り込み/解決の考え方を確認でき、型の導入時のトラブルシュートに使える

## JavaScript 運用（JSDoc / 段階的移行）

### Intro to JavaScript with TypeScript
- **URL**: https://www.typescriptlang.org/docs/handbook/intro-to-js-ts.html
- **説明**: JS/TS混在プロジェクトの前提を揃え、移行期の規約（strict化の段階など）を作りやすい

### Type Checking JavaScript Files
- **URL**: https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html
- **説明**: `checkJs` 等でJSを型チェックする際の挙動を理解でき、移行期でも安全性を上げやすい

### JSDoc Reference
- **URL**: https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html
- **説明**: 対応JSDocタグ（`@type`/`@param`/`@template`/`@satisfies` 等）を確認でき、JSコードでも型情報を維持できる

### Creating .d.ts Files from .js
- **URL**: https://www.typescriptlang.org/docs/handbook/creating-dts-files-from-js.html
- **説明**: `allowJs` + `declaration` などでJSから型を生成する方法がまとまっており、段階的移行や型公開の足場になる

### Declaration Files - .d.ts from JavaScript
- **URL**: https://www.typescriptlang.org/docs/handbook/declaration-files/dts-from-js.html
- **説明**: JS資産から型定義を用意する流れを別観点で確認でき、移行/公開の設計に役立つ

### Migrating from JavaScript
- **URL**: https://www.typescriptlang.org/docs/handbook/migrating-from-javascript.html
- **説明**: JS→TS移行の進め方を公式の観点で整理でき、ルール化（移行順序/許容範囲）に使える

## ビルド・ツール連携（運用の安定化）

### Integrating with Build Tools
- **URL**: https://www.typescriptlang.org/docs/handbook/integrating-with-build-tools.html
- **説明**: Babel/webpack/Rollup/Vite 等の典型構成と注意点がまとまっており、型チェックとトランスパイルの責務分離に役立つ

### Configuring Watch
- **URL**: https://www.typescriptlang.org/docs/handbook/configuring-watch.html
- **説明**: `--watch` の仕組みと `watchOptions` の調整指針（特にLinuxの制約）を理解でき、開発体験と安定性を両立しやすい

## JSX（React等）

### JSX
- **URL**: https://www.typescriptlang.org/docs/handbook/jsx.html
- **説明**: `.tsx`/JSX設定と型付けの基礎を押さえ、UI開発時の型の決め方を揃えられる
