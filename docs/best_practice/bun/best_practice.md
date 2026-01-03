## 概要
このドキュメントは、Bun をプロダクション利用する際の実務的なベストプラクティスをまとめたものです。
対象は主に「Bun ランタイム」「Bun のパッケージマネージャー（bun install / bun add）」「bunfig.toml」「bun test」「bun build（bundler）」です。

## 基本方針
- 再現性を最優先する（`bun.lock` をコミットし、CI では `bun ci` / `--frozen-lockfile` を使う）
- 供給網（サプライチェーン）リスクを前提にする（ライフサイクルスクリプト、最低公開経過時間、脆弱性スキャン、overrides を活用）
- 「Bun は Node.js 互換」を前提にしつつ、互換性テーブルを確認して移植コストを見積もる
- ビルド（bundler）と型チェック（tsc）を混同しない（Bun は TypeScript を型チェックしない）

## インストールとバージョン管理
- チーム/CI で Bun のバージョンを固定する（ローカル・CI・本番で差分を出さない）
- インストール後は `bun --version` と `bun --revision` で確認する
- 既存のパッケージマネージャーと混在させる場合は、移行範囲（lockfile・scripts・CI）を段階的に切る

## 依存関係管理（bun install / bun add）
### 再現性と CI
- `bun.lock` はコミットする（再現性の前提）
- CI では `bun install` ではなく `bun ci` を使う（`bun install --frozen-lockfile` と同等）
- `--production` を使う場合は「devDependencies を落とす」ことを明示し、本番向けジョブに限定する

```sh
# CI の基本形（再現性重視）
bun ci

# 本番向け（devDependencies を入れない）
bun install --production --frozen-lockfile
```

### lockfile の運用（生成・移行）
- `bun install --lockfile-only` で、`node_modules` を作らず lockfile だけ生成できる
- 既存の `bun.lockb`（バイナリ）を使っている場合は、必要に応じてテキスト形式の `bun.lock` に移行する（Bun v1.2 以降）

```sh
# lockfile だけ生成（node_modules を作らない）
bun install --lockfile-only

# bun.lockb → bun.lock（テキスト）へ移行する例
bun install --save-text-lockfile --frozen-lockfile --lockfile-only
```

### ライフサイクルスクリプト（trustedDependencies）
Bun はセキュリティ上の理由で、依存関係側の任意ライフサイクルスクリプト（例: postinstall）をデフォルトでは実行しません。
必要なものだけを明示的に許可する運用にします。

- 必要なパッケージだけ `trustedDependencies` に追加する（無差別に許可しない）
- `trustedDependencies` の変更はレビュー対象にする（実質的に任意コード実行の許可に近い）

```json
{
  "name": "my-app",
  "trustedDependencies": ["my-trusted-package"]
}
```

また、Bun には `trustedDependencies` を扱うためのコマンドが用意されています。
- `bun pm trust <package>` を使い、許可が必要な依存だけを追加する
- npm 以外（`file:` / `link:` / `git:` / `github:` など）由来の依存は、特に慎重に扱い、必要な場合のみ trust する

```sh
# trustedDependencies へ追加（必要なものだけ）
bun pm trust my-trusted-package
```

### インストール方式（linker）
- モノレポ/ワークスペースでは `isolated` が「ファントム依存（宣言していない依存が使えてしまう）」を避けやすい
- 既存プロジェクト移行では互換性のため `hoisted` が選ばれることがあるため、挙動差に注意する

`bunfig.toml` で明示できます。

```toml
[install]
linker = "isolated" # または "hoisted"
```

### 最低公開経過時間（minimumReleaseAge）
サプライチェーン対策として、公開直後のバージョンを解決から除外できます。
- 重要な本番プロジェクトでは `minimumReleaseAge` を検討する
- 例外（`minimumReleaseAgeExcludes`）は最小限にし、理由を残す

```toml
[install]
# 3日以上経過したバージョンのみ許可
minimumReleaseAge = 259200
minimumReleaseAgeExcludes = ["@types/node", "typescript"]
```

### overrides / resolutions（脆弱性対応・緊急固定）
- 脆弱性が入った「孫依存」を一時的に固定する用途に使う
- Bun は現時点でトップレベルの `overrides` のみをサポート（ネストは不可）

```json
{
  "overrides": {
    "bar": "~4.4.0"
  }
}
```

## Workspaces / モノレポ
- ルート `package.json` の `workspaces` を正しく定義し、必要に応じて除外パターン（`!`）も活用する
- ワークスペース間参照は `workspace:*` を基本にしてローカルリンクを明示する
- 部分的なインストール・実行が必要なら `--filter` を活用する

```sh
# 一部ワークスペースのみ
bun install --filter "./packages/pkg-*" --filter "!./packages/pkg-c"
```

## bunfig.toml（設定の分離と優先順位）
- プロジェクトルートに `bunfig.toml` を置くのが基本
- グローバル（`~/.bunfig.toml` など）とローカルはシャローにマージされ、ローカルが優先
- CLI フラグは bunfig を上書きするため、CI ではフラグ/設定の責務を明確にする

### CI キャッシュ
Bun はダウンロードしたパッケージをグローバルキャッシュ（例: `~/.bun/install/cache/`）に保存します。
- CI ではこのキャッシュを保存/復元するとインストールが安定しやすい
- ただし「キャッシュがあっても lockfile 固定（`bun ci` / `--frozen-lockfile`）を崩さない」

### .env 自動読み込みの扱い
Bun は `.env` を自動で読み込みます。
- 本番/CI では「システム環境変数のみ」を基本にし、必要なら `--no-env-file` または bunfig で無効化する
- どうしてもファイルを読む場合は `--env-file` で読み込み対象を明示する

```toml
# bunfig.toml
# 本番・CI を想定して自動読み込みを止める
env = false
```

```sh
# 明示的に .env を指定
bun --env-file=.env.production run start

# 自動 .env を無効化
bun run --no-env-file start
```

## TypeScript
- Bun は TypeScript を実行できるが、型チェックはしない
- CI では `bun test` と別に `tsc --noEmit`（または同等の型チェック）を走らせる
- `@types/bun` は devDependencies に入れる

```sh
bun add -d @types/bun
bunx tsc --noEmit
```

Bun ドキュメントには推奨 `compilerOptions` 例があるため、プロジェクトの方針（strictness、未使用検査など）に合わせて採用します。

## テスト（bun test）
### 実行と検出
- `bun test` は規定パターン（`*.test.*`, `*.spec.*` など）を再帰的に探索する
- CI では `bun test` を基本にし、必要なら JUnit 出力を使う

```sh
bun test
bun test --reporter=junit --reporter-outfile=./bun.xml
```

### 安定性とパフォーマンス
- フレーク検出には `--rerun-each`、順序依存検出には `--randomize` と `--seed` を使う
- リソースを使うテストは `test.serial` を使い、並列化の副作用を避ける
- 並列を有効化する場合は `--max-concurrency` で上限を設けて枯渇を防ぐ

```sh
bun test --randomize
bun test --seed 123456
bun test --concurrent --max-concurrency 4
```

### 落とし穴
- `done` コールバックを使うテストは、呼び忘れでハングする（可能なら `async`/`await` に寄せる）

## ビルド（bun build / Bun.build）
- `target` と `format` を明示し、実行環境に合わないバンドルを作らない
  - `target: "browser"` で Node 組み込み API を呼ぶと実行時に破綻しうる
- sourcemap 方針（none/linked/external/inline）を決める
- 環境変数のインライン化は漏洩リスクがあるため、プレフィックス方式（例: `MY_PUBLIC_*`）で絞る

```ts
await Bun.build({
  entrypoints: ["./src/index.ts"],
  outdir: "./dist",
  target: "bun",
  sourcemap: "external",
  env: "MY_PUBLIC_*",
});
```

## コンテナ / Docker（本番デプロイの最小ポイント）
- 公式の Bun イメージ（`oven/bun`）を使い、依存インストールをレイヤーキャッシュできるようにする
- 依存インストールは `--frozen-lockfile` を使い、`bun.lock` と `package.json` の不整合を許容しない
- 可能ならマルチステージビルドで、devDependencies を含まない `--production` なインストールを別レイヤーに分ける

```dockerfile
FROM oven/bun:1 AS base
WORKDIR /usr/src/app

FROM base AS install
RUN mkdir -p /temp/prod
COPY package.json bun.lock /temp/prod/
RUN cd /temp/prod && bun install --frozen-lockfile --production

FROM base AS run
COPY --from=install /temp/prod/node_modules ./node_modules
COPY . .
CMD ["bun", "run", "start"]
```

## セキュリティ
### セキュリティスキャナ
- 組織の要件がある場合、`[install.security]` の scanner を導入する
- CI では「warn が CI で失敗になる」挙動を前提に、運用ポリシー（許容/遮断）を合わせる

```toml
[install.security]
scanner = "@acme/bun-security-scanner"
```

### Bun Shell（$ テンプレートリテラル）
- Bun Shell は補間値をデフォルトでエスケープし、コマンド注入を防ぐ設計
- ただし `bash -c` のように外部シェルへ渡すと保護が効かないため、ユーザー入力をそのまま渡さない
- 引数注入（外部コマンドがフラグとして解釈する）もあり得るので、ユーザー入力はバリデーションする

## Node.js 互換性
- 移行時は互換性ページを確認し、「完全互換ではないモジュール/挙動」を早期に洗い出す
- Node 固有 API を多用する依存がある場合は、置換/回避策を検討する（例: `async_hooks` は使用が強く非推奨の領域がある）

## よくある落とし穴と回避
- `bun.lock` をコミットしない → 環境差で事故る（必ずコミット）
- CI で `bun install` を使い lockfile を更新してしまう → `bun ci` に統一
- `.env` 自動読み込みで本番に意図しない値が入る → 本番/CI は `env = false` or `--no-env-file`
- `trustedDependencies` を増やしすぎる → 依存の任意コード実行が広がる（最小化・レビュー）
- bundler の `env: "inline"` を安易に使う → シークレットがバンドルに混入（プレフィックス方式を基本に）

## 参考（公式ドキュメント）
- https://bun.com/docs/installation
- https://bun.com/docs/pm/cli/install
- https://bun.com/docs/pm/lockfile
- https://bun.com/docs/pm/workspaces
- https://bun.com/docs/runtime/bunfig
- https://bun.com/docs/runtime/environment-variables
- https://bun.com/docs/typescript
- https://bun.com/docs/test
- https://bun.com/docs/test/writing-tests
- https://bun.com/docs/bundler
- https://bun.com/docs/pm/security-scanner-api
- https://bun.com/docs/pm/overrides
- https://bun.com/docs/runtime/shell
- https://bun.com/docs/runtime/nodejs-compat

## 参考（外部）
- https://bun.com/docs/guides/ecosystem/docker
- https://hy2k.dev/en/blog/2025/10-15-bun-install-isolated-vs-hoisted/
- https://socket.dev/blog/bun-1-2-19-adds-isolated-installs-for-better-monorepo-support
- https://medium.com/@Modexa/the-practical-guide-to-bun-in-production-c51583bfe605
