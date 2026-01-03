---
description: 'Bun（ランタイム/パッケージマネージャー/テスト/バンドラ）を使うプロジェクトで、再現性・セキュリティ・運用性を高めるためのベストプラクティス規則'
applyTo: '**/package.json, **/bun.lock*, **/bunfig.toml, **/*.ts, **/*.tsx, **/*.js, **/*.jsx, **/*.mjs, **/*.cjs, **/*.cts, **/*.mts, Dockerfile, **/Dockerfile.*, **/.github/workflows/**/*.yml, **/.github/workflows/**/*.yaml'
---

# Bun ベストプラクティス（Copilot向け）

GitHub Copilot が Bun を使うコード/設定/CI を生成・更新する際に、再現性・セキュリティ・保守性を損なわないための指示です。

## 基本方針

- ロックファイル（`bun.lock*`）を前提に「再現可能なインストール」を最優先する
- 依存インストール時の任意コード実行（postinstall 等）を最小化する
- TypeScript は「実行」と「型検査」を分離する（Bun は実行時に型検査しない）
- モノレポは workspaces と linker の前提を揃え、ファントム依存を防ぐ
- `.env` 自動読み込みを前提にしすぎず、CI/本番での振る舞いを明示する

## 依存関係とCI（再現性）

- CI では `bun install` ではなく `bun ci`（または `bun install --frozen-lockfile`）を使う
- `bun.lock*` は必ずコミットし、CI では lockfile の更新を許容しない
- 本番向けイメージ/ジョブでは `--production` を使い、devDependencies を入れない

**✅ 良い例**
```yaml
# .github/workflows/ci.yml（例）
- run: bun ci
- run: bunx tsc --noEmit
- run: bun test
```

**❌ 悪い例（CIでlockfileを更新し得る）**
```yaml
- run: bun install
```

## セキュリティ（trustedDependencies / minimumReleaseAge / overrides）

- 依存のライフサイクルスクリプトを無差別に許可しない
- 必要な依存のみ `trustedDependencies` で許可し、変更はレビュー対象にする
- `file:` / `link:` / `git:` / `github:` 由来の依存は、特に慎重に trust する
- サプライチェーン対策として `minimumReleaseAge` の導入を検討する
- トランジティブ依存の緊急固定は `overrides` を使い、理由（CVE/互換性）を残す

**✅ 良い例（必要最小限の trust）**
```json
{
  "trustedDependencies": ["my-trusted-package"]
}
```

**✅ 良い例（緊急固定の overrides）**
```json
{
  "overrides": {
    "some-transitive-dep": "1.2.3"
  }
}
```

**❌ 悪い例（許可を広げすぎる）**
```json
{
  "trustedDependencies": ["*"]
}
```

## Workspaces / モノレポ

- ワークスペース間依存は `workspace:*`（または `workspace:^`）で明示する
- 他パッケージの `src` を相対パスで直参照しない（公開API経由に統一する）
- linker（`isolated`/`hoisted`）の前提を `bunfig.toml` で明示し、開発者間・CI間で差が出ないようにする

**✅ 良い例（workspaceプロトコル）**
```json
{
  "workspaces": ["packages/*"],
  "dependencies": {
    "@acme/core": "workspace:*"
  }
}
```

**❌ 悪い例（パッケージ境界の破壊）**
```ts
import { internalThing } from "../../packages/core/src/internal";
```

## bunfig.toml と環境変数

- プロジェクト設定は `bunfig.toml` に集約し、チームで共有する
- `.env` は開発便利機能として扱い、CI/本番では「環境変数注入」を基本にする
- シークレットはコード/バンドルに含めない

**✅ 良い例（CI/本番で .env を無効化する前提）**
```toml
# bunfig.toml
env = false
```

## TypeScript（型検査の分離）

- Bun 実行だけで型安全を担保したつもりにならない
- CI では `bunx tsc --noEmit`（または同等）を必ず実行する
- `@types/bun` は devDependencies に入れる

**✅ 良い例**
```json
{
  "scripts": {
    "dev": "bun run src/index.ts",
    "typecheck": "bunx tsc --noEmit",
    "test": "bun test"
  }
}
```

**❌ 悪い例（型検査が無い）**
```json
{
  "scripts": {
    "check": "bun run src/index.ts"
  }
}
```

## テスト（bun test）

- 基本は `bun test` を使い、安定性を優先する
- フレーク対策として、順序依存が疑われる場合は `--randomize` / `--seed` を活用する
- I/O を伴うテストは並列性の影響を受けやすいので、必要に応じて直列化や分離を行う

## ビルド（bun build / Bun.build）

- `target` と `format` は明示し、実行環境（browser/node/bun）と齟齬のない設定にする
- 環境変数のインライン化は最小限にし、公開してよいキーに限定する

## Shell 実行の安全性

- ユーザー入力を外部シェル（例: `bash -c`）にそのまま渡さない
- スクリプトでシェルを使う場合は、変数を必ずクォートし、失敗を見逃さない

## Docker（最小要件）

- 公式イメージ（`oven/bun`）を使い、依存インストールのキャッシュが効く `COPY` 順序にする
- `--frozen-lockfile` を使い、イメージ内で lockfile が変わる状態を許容しない
- 可能ならマルチステージで `--production` な依存だけを実行イメージに入れる

**✅ 良い例（依存を先にCOPY）**
```dockerfile
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production
```

**❌ 悪い例（キャッシュが効きにくい）**
```dockerfile
COPY . .
RUN bun install
```

## 検証（生成後に意識すること）

- CI: `bun ci` → `bunx tsc --noEmit` → `bun test` の順で通る構成にする
- 依存追加/更新: `trustedDependencies` / `overrides` / `minimumReleaseAge` の影響を確認する
