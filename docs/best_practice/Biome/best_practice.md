# Biome ベストプラクティス

このドキュメントでは、Biomeを使用してコード品質を管理する際のベストプラクティスをまとめています。

## Biomeとは

BiomeはJavaScript、TypeScript、JSX、TSX、JSON、HTML、CSS、GraphQL向けの高速なフォーマッタおよびリンタです。

### 主な特徴

- **高速性**: ESLint + Prettierと比較して約13倍高速（450ファイルで38秒 → 2.9秒）
- **Prettierとの高い互換性**: 97%の互換性を持ち、スムーズな移行が可能
- **統合ツール**: フォーマット、リント、インポート整理を単一のツールで提供
- **Rust製**: 高パフォーマンスを実現するためにRustで実装
- **豊富なルール**: 410以上のルールを提供（ESLint、TypeScript ESLint等から）
- **リアルタイムフォーマット**: マルフォーマットされたコードもエディタで入力中にフォーマット可能

## 目次

- [セットアップとインストール](#セットアップとインストール)
- [設定ファイルの管理](#設定ファイルの管理)
- [Formatter（フォーマッタ）の活用](#formatterフォーマッタの活用)
- [Linter（リンタ）の活用](#linterリンタの活用)
- [エディタ統合](#エディタ統合)
- [CI/CD統合](#cicd統合)
- [VCS統合](#vcs統合)
- [既存プロジェクトへの導入](#既存プロジェクトへの導入)
- [大規模プロジェクトでの使用](#大規模プロジェクトでの使用)
- [パフォーマンス最適化](#パフォーマンス最適化)

## セットアップとインストール

### パッケージマネージャーでのインストール

**推奨事項：**
- `npm`、`pnpm`、`yarn`、`bun`などのパッケージマネージャーを使用してインストールする
- **正確なバージョンを固定する**ために `--save-exact` オプションを使用する

```bash
npm install --save-dev --save-exact @biomejs/biome
```

**理由：**
- 全員が完全に同じバージョンのBiomeを使用することを保証
- パッチリリースでも微妙に異なる動作を起こすことがあるため

### 初期設定

プロジェクトルートで以下のコマンドを実行して設定ファイルを作成：

```bash
npx @biomejs/biome init
```

JSONCフォーマットが必要な場合：

```bash
npx @biomejs/biome init --jsonc
```

## 設定ファイルの管理

### 基本設定

**推奨事項：**
- プロジェクトごとに `biome.json` または `biome.jsonc` を作成する
- `$schema` フィールドを設定してIDEでのオートコンプリートを有効化する

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.5/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true,
    "defaultBranch": "main"
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "tab",
    "indentWidth": 2,
    "lineWidth": 80
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all"
    }
  }
}
```

### ファイルの除外設定

**推奨事項：**
- 生成されたファイルやビルド成果物は明示的に除外する
- `files.includes` を使用してglobパターンで処理対象を制御する

```json
{
  "files": {
    "includes": [
      "**",
      "!**/dist",
      "!**/build",
      "!**/*.generated.js",
      "!**/coverage"
    ]
  }
}
```

### VCS統合の有効化

**推奨事項：**
- VCS統合を有効にして `.gitignore` のファイルを自動的に無視する
- デフォルトブランチを設定して `--changed` オプションを活用する

```json
{
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true,
    "defaultBranch": "main"
  }
}
```

## Formatter（フォーマッタ）の活用

### デフォルト設定の活用

**推奨事項：**
- Biomeのデフォルト設定をできるだけ使用する
- オプション設定についての議論を最小限に抑える

**主要オプション：**
- `indentStyle`: `"tab"` または `"space"` （デフォルト: `"tab"`）
- `indentWidth`: インデントレベルごとのスペース数 （デフォルト: `2`）
- `lineWidth`: コードを折り返す列幅 （デフォルト: `80`）
- `lineEnding`: `"lf"` または `"crlf"` （デフォルト: `"lf"`）

### JavaScript固有のオプション

```json
{
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always",
      "bracketSpacing": true,
      "bracketSameLine": false
    }
  }
}
```

### フォーマットの抑制

特定のコードブロックでフォーマットを無効にする場合：

```javascript
// biome-ignore format: この配列はフォーマットさせない
const matrix = [
  (2 * n) / (r - l), 0, (r + l) / (r - l), 0,
  0, (2 * n) / (t - b), (t + b) / (t - b), 0,
];
```

### CLI使用方法

```bash
# フォーマットのチェック
npx @biomejs/biome format ./src

# フォーマットを適用
npx @biomejs/biome format --write ./src
```

## Linter（リンタ）の活用

### 推奨ルールの有効化

**推奨事項：**
- `rules.recommended: true` で推奨ルールを有効化する
- 必要に応じて個別のルールを調整する

```json
{
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": {
        "noDebugger": "error",
        "noConsoleLog": "warn"
      },
      "style": {
        "useConst": "error",
        "useTemplate": "warn"
      }
    }
  }
}
```

### ルールの重大度

- `"error"`: エラーとして扱い、CI失敗の原因となる
- `"warn"`: 警告として表示（`--error-on-warnings` でCI失敗可）
- `"info"`: 情報として表示、CI失敗しない
- `"off"`: ルールを無効化

### 修正の適用

```bash
# 安全な修正のみ適用
npx @biomejs/biome lint --write ./src

# 安全ではない修正も適用（要注意）
npx @biomejs/biome lint --write --unsafe ./src
```

### コードの無視

特定のコード行でリントを無視する場合：

```javascript
// biome-ignore lint: 理由を記述
debugger;

// biome-ignore lint/suspicious/noDebugger: デバッグ中のため
debugger;
```

### ルール固有の設定

```json
{
  "linter": {
    "rules": {
      "correctness": {
        "noUnusedVariables": {
          "level": "error",
          "fix": "none"
        }
      },
      "style": {
        "useConst": {
          "level": "warn",
          "fix": "safe"
        }
      }
    }
  }
}
```

## エディタ統合

### VS Code

**推奨事項：**
- 公式の [Biome VS Code拡張機能](https://marketplace.visualstudio.com/items?itemName=biomejs.biome) をインストールする
- プロジェクトの `.vscode/settings.json` で設定を共有する

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "[javascript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[json]": {
    "editor.defaultFormatter": "biomejs.biome"
  }
}
```

### その他のエディタ

- **IntelliJ IDEA**: 公式プラグインが利用可能
- **Zed**: ビルトインサポート
- その他: [サードパーティ拡張機能](https://biomejs.dev/ja/guides/editors/third-party-extensions)を参照

## CI/CD統合

### GitHub Actions

**推奨事項：**
- 公式の `biomejs/setup-biome` アクションを使用する
- `biome ci` コマンドを使用（読み取り専用モード）

```yaml
name: Code Quality
on:
  push:
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          persist-credentials: false

      - name: Setup Biome
        uses: biomejs/setup-biome@v2
        with:
          version: latest

      - name: Run Biome
        run: biome ci .
```

### reviewdog統合

プルリクエストにコメントを追加する場合：

```yaml
- uses: mongolyy/reviewdog-action-biome@v1
  with:
    github_token: ${{ secrets.github_token }}
    reporter: github-pr-review
```

## VCS統合

### Git統合の有効化

**推奨事項：**
- `.gitignore` ファイルを自動的に尊重する
- 変更されたファイルのみを処理するオプションを活用する

```json
{
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true,
    "defaultBranch": "main"
  }
}
```

### 変更されたファイルのみを処理

```bash
# デフォルトブランチとの差分のみ
biome check --changed

# 特定のブランチとの差分
biome check --changed --since=develop

# ステージされたファイルのみ
biome check --staged
```

### Git Hooks

**推奨事項：**
- `pre-commit` フックで自動チェックを実行する
- `husky` や `lint-staged` と組み合わせる

**Huskyのセットアップ例：**

```bash
# Huskyのインストール
npm install --save-dev husky lint-staged
npx husky init
```

**package.jsonの設定：**

```json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{js,ts,jsx,tsx,json,css,html}": [
      "biome check --write --no-errors-on-unmatched"
    ]
  }
}
```

**.husky/pre-commitファイル：**

```bash
npx lint-staged
```

**単一コマンドでの実行：**

Biomeの最大の利点の一つは、単一のコマンドで複数の処理を実行できることです：

```bash
# フォーマット、リント、インポート整理を一度に実行
biome check --write ./src

# または
npm run biome:check
```

これにより、ESLint + Prettierのように複数のコマンドを実行する必要がなくなります。

## 既存プロジェクトへの導入

### ESLint/Prettierからの移行

**推奨事項：**
- `biome migrate` コマンドを使用して設定を自動移行する
- 設定ファイルを自動的に変換して移行を簡素化

```bash
# ESLintの設定を移行
npx @biomejs/biome migrate eslint --write

# インスパイアされたルールも含める
npx @biomejs/biome migrate eslint --write --include-inspired

# Prettierの設定を移行
npx @biomejs/biome migrate prettier --write
```

**移行後の推奨手順：**

1. 既存のESLintとPrettierの設定をバックアップ
2. Biomeの移行コマンドを実行
3. 生成された`biome.json`を確認・調整
4. テストを実行して問題がないか確認
5. 既存のリンター/フォーマッターを削除または無効化

**注意点：**
- BiomeとPrettierの互換性は97%であり、完全な一致ではない
- 一部のESLintルールはBiomeに存在しない場合がある
- 必要に応じて設定を微調整する

### 既存ツールとの並行運用

移行期間中は、Biomeが処理するファイルに対して他のツールを無効化することを推奨：

**.eslintrc.json:**
```json
{
  "ignorePatterns": ["src/**"]
}
```

**.prettierignore:**
```
src/**
```

### 段階的な導入

**推奨事項：**
1. まずフォーマッタのみを有効化
2. リンタは警告レベルで有効化
3. 徐々にエラーレベルに引き上げ

```json
{
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": "warn"
    }
  }
}
```

## 大規模プロジェクトでの使用

### 複数の設定ファイル

**推奨事項：**
- プロジェクト構造に応じて複数の `biome.json` を配置する
- Biomeは最も近い設定ファイルを自動的に検索する

```
app/
├── biome.json           # 共通設定
├── backend/
│   └── biome.json      # バックエンド固有の設定
└── frontend/
    └── biome.json      # フロントエンド固有の設定
```

### 設定の継承

**推奨事項：**
- `extends` を使用して共通設定を継承する

```json
{
  "extends": ["../biome.json"],
  "formatter": {
    "indentStyle": "space"
  }
}
```

### モノレポでの使用

**推奨事項：**
- ルートに `biome.json` を配置
- `overrides` を使用してパッケージごとに設定を調整

```json
{
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "overrides": [
    {
      "include": ["packages/logger/**"],
      "linter": {
        "rules": {
          "suspicious": {
            "noConsoleLog": "off"
          }
        }
      }
    }
  ]
}
```

## パフォーマンス最適化

### 処理対象の最適化

**推奨事項：**
- ビルド成果物や依存関係を明示的に除外する
- `files.includes` で処理対象を絞り込む

```json
{
  "files": {
    "includes": [
      "src/**",
      "!**/dist/**",
      "!**/build/**",
      "!**/node_modules/**",
      "!**/.next/**",
      "!**/coverage/**"
    ]
  }
}
```

### パフォーマンスの調査

遅延が発生した場合、トレーシング機能を使用：

```bash
biome check --log-level=tracing --log-kind=json --log-file=biome.log ./src
```

### projectドメインルールの影響

**注意点：**
- projectドメインのルールはスキャナーを起動し、パフォーマンスに影響する
- 必要な場合のみ有効化する

```json
{
  "linter": {
    "rules": {
      "project": {
        "noFloatingPromises": "off"
      }
    }
  }
}
```

## コマンドのベストプラクティス

### check コマンドの活用

**推奨事項：**
- `check` コマンドで複数の処理を一度に実行する

```bash
# フォーマット、リント、インポート整理を実行
biome check --write ./src
```

### CI環境での使用

**推奨事項：**
- `ci` コマンドを使用（読み取り専用）

```bash
biome ci ./src
```

### 特定のルールのみ実行

```bash
# 特定のルールのみ実行
biome lint --only=style/useNamingConvention --only=a11y

# 特定のルールをスキップ
biome lint --skip=style --skip=suspicious/noExplicitAny
```

## まとめ

Biomeを効果的に活用するための重要なポイント：

### 導入時

1. **バージョンの固定**: `--save-exact` でバージョンを固定する
2. **設定ファイルの管理**: プロジェクトごとに `biome.json` を作成する
3. **段階的導入**: まず警告レベルで導入し、徐々に厳格化する
4. **移行ツールの活用**: `biome migrate` で既存設定を自動変換

### 日常的な使用

5. **単一コマンドの活用**: `biome check --write` で全処理を一度に実行
6. **エディタ統合**: 公式拡張機能をインストールし、保存時フォーマットを有効化
7. **VCS統合**: `.gitignore` を尊重し、変更ファイルのみを処理
8. **Git Hooks**: Huskyと組み合わせてコミット前に自動チェック

### プロジェクト管理

9. **CI/CD統合**: GitHub Actionsなどで自動チェックを実施
10. **パフォーマンス最適化**: ビルド成果物を除外し、処理対象を最適化
11. **設定の継承**: 大規模プロジェクトでは `extends` と `overrides` を活用
12. **モノレポ対応**: ルートに設定を置き、`overrides` でパッケージごとに調整

### パフォーマンス上の利点

- **速度**: ESLint + Prettierと比較して約13倍高速
- **統合**: 複数ツールの設定競合を排除
- **リアルタイム**: エディタでのリアルタイムフィードバック

これらのベストプラクティスに従うことで、Biomeを使った効率的で一貫性のあるコード品質管理が実現できます。Biomeは単なるフォーマッタやリンタではなく、開発体験を向上させる統合ツールチェーンとして、モダンなWeb開発プロジェクトに最適です。

## 参考リソース

- [Biome公式ドキュメント](https://biomejs.dev/ja/)
- [Biomeプレイグラウンド](https://biomejs.dev/playground)
- [GitHub: biomejs/biome](https://github.com/biomejs/biome)
- [Biome vs ESLint + Prettier比較](https://biomejs.dev/ja/formatter/differences-with-prettier)
- [Biome Discord コミュニティ](https://biomejs.dev/chat)
