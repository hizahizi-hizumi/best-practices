---
description: 'Biomeフォーマッタおよびリンタのベストプラクティス規則'
applyTo: 'biome.json, biome.jsonc, **/*.js, **/*.ts, **/*.jsx, **/*.tsx, **/*.json'
---

# Biome フォーマッタ・リンタ ベストプラクティス

GitHub Copilot が Biome を使用したコード品質管理を行う際に従うべきガイドラインとベストプラクティス。

## プロジェクトコンテキスト

- **対象ツール**: Biome（JavaScript、TypeScript、JSX、TSX、JSON、HTML、CSS、GraphQL向けの高速フォーマッタ・リンタ）
- **主な特徴**: Rust実装、ESLint + Prettierの約13倍高速、97%のPrettier互換性
- **パフォーマンス**: 450ファイルで38秒 → 2.9秒
- **統合ツール**: フォーマット、リント、インポート整理を単一ツールで提供

## 一般的な指示

- JavaScript/TypeScriptプロジェクトではBiomeを標準のコード品質ツールとして使用
- ESLint + Prettierの代わりにBiomeの単一コマンドを使用
- `biome.json` または `biome.jsonc` を必ずプロジェクトルートに配置
- 正確なバージョンを固定するため `--save-exact` オプションでインストール
- VCS統合を有効化して `.gitignore` のファイルを自動的に無視
- CI/CD環境では必ず `biome ci` コマンドを使用（読み取り専用モード）

## インストールとセットアップ

### パッケージマネージャーでのインストール

**必須**: `--save-exact` オプションを使用してバージョンを固定：

```bash
npm install --save-dev --save-exact @biomejs/biome
```

**理由**: 全員が完全に同じバージョンを使用することを保証し、パッチリリースでの微妙な動作の違いを防ぐ

### 初期設定の作成

```bash
# JSON形式の設定ファイル作成
npx @biomejs/biome init

# JSONC形式が必要な場合
npx @biomejs/biome init --jsonc
```

## 設定ファイル（biome.json）のベストプラクティス

### 基本設定構造

**必須**: `$schema` フィールドを設定してIDEのオートコンプリートを有効化

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

### ファイル除外設定

**推奨**: 生成ファイルやビルド成果物を明示的に除外

```json
{
  "files": {
    "includes": [
      "src/**",
      "!**/dist/**",
      "!**/build/**",
      "!**/*.generated.js",
      "!**/coverage/**",
      "!**/node_modules/**",
      "!**/.next/**"
    ]
  }
}
```

### Good Example - 推奨される設定

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.5/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true,
    "defaultBranch": "main"
  },
  "files": {
    "includes": [
      "src/**",
      "!**/dist/**",
      "!**/node_modules/**"
    ]
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
      "recommended": true,
      "suspicious": {
        "noDebugger": "error",
        "noConsoleLog": "warn"
      },
      "style": {
        "useConst": "error"
      }
    }
  }
}
```

### Bad Example - 避けるべき設定

```json
{
  "formatter": {
    "enabled": true
  },
  "linter": {
    "enabled": true
  }
}
```

**問題点**:
- `$schema` が設定されていない（IDEサポート無効）
- VCS統合が有効化されていない
- ファイル除外設定がない
- ルール設定が具体的でない

## コードフォーマットのベストプラクティス

### デフォルト設定の活用

**推奨**: Biomeのデフォルト設定をできるだけ使用し、チーム内の議論を最小化

| オプション | デフォルト値 | 説明 |
|-----------|------------|------|
| `indentStyle` | `"tab"` | `"tab"` または `"space"` |
| `indentWidth` | `2` | インデントレベルごとのスペース数 |
| `lineWidth` | `80` | コードを折り返す列幅 |
| `lineEnding` | `"lf"` | `"lf"` または `"crlf"` |

### JavaScript固有のフォーマットオプション

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

**重要**: `biome-ignore` コメントには必ず理由を記述

### フォーマットコマンドの使用

```bash
# フォーマットのチェックのみ
npx @biomejs/biome format ./src

# フォーマットを適用
npx @biomejs/biome format --write ./src

# 変更されたファイルのみ
npx @biomejs/biome format --write --changed
```

## リンタールールのベストプラクティス

### 推奨ルールの有効化

**必須**: `rules.recommended: true` で推奨ルールを有効化

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
      },
      "correctness": {
        "noUnusedVariables": "error"
      }
    }
  }
}
```

### ルールの重大度レベル

| 重大度 | 説明 | CI動作 |
|-------|------|--------|
| `"error"` | エラーとして扱う | CI失敗 |
| `"warn"` | 警告として表示 | `--error-on-warnings` でCI失敗 |
| `"info"` | 情報として表示 | CI成功 |
| `"off"` | ルールを無効化 | - |

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

### リントの無視

特定のコード行でリントを無視する場合：

```javascript
// biome-ignore lint: 理由を記述
debugger;

// biome-ignore lint/suspicious/noDebugger: デバッグ中のため
debugger;

// 複数行の無視
// biome-ignore lint/suspicious/noConsoleLog: トラブルシューティング用ログ
console.log("Debug info:", data);
```

**重要**: `biome-ignore` コメントには必ず具体的な理由を記述

### リントコマンドの使用

```bash
# リントチェックのみ
npx @biomejs/biome lint ./src

# 安全な修正のみ適用
npx @biomejs/biome lint --write ./src

# 安全ではない修正も適用（要注意）
npx @biomejs/biome lint --write --unsafe ./src

# 特定のルールのみ実行
npx @biomejs/biome lint --only=style/useNamingConvention --only=a11y

# 特定のルールをスキップ
npx @biomejs/biome lint --skip=style --skip=suspicious/noExplicitAny
```

## エディタ統合の推奨事項

### VS Code設定

**必須**: 公式のBiome VS Code拡張機能をインストール

プロジェクトの `.vscode/settings.json` で設定を共有：

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
  },
  "[jsonc]": {
    "editor.defaultFormatter": "biomejs.biome"
  }
}
```

### その他のエディタ

| エディタ | サポート状況 |
|---------|------------|
| IntelliJ IDEA | 公式プラグイン利用可能 |
| Zed | ビルトインサポート |
| その他 | サードパーティ拡張機能を参照 |

## CI/CD統合の推奨事項

### GitHub Actions設定

**推奨**: 公式の `biomejs/setup-biome` アクションを使用

```yaml
name: Code Quality
on:
  push:
    branches: [main, develop]
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
- name: Run Biome with reviewdog
  uses: mongolyy/reviewdog-action-biome@v1
  with:
    github_token: ${{ secrets.github_token }}
    reporter: github-pr-review
    level: warning
```

### Good Example - CI/CD設定

```yaml
name: Code Quality
on:
  push:
  pull_request:

jobs:
  biome:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: biomejs/setup-biome@v2
        with:
          version: latest
      - name: Run Biome
        run: biome ci .
      - name: Check formatting
        run: biome format --check .
```

### Bad Example - 避けるべきCI設定

```yaml
jobs:
  biome:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: npx @biomejs/biome check --write .
```

**問題点**:
- `ci` コマンドではなく `check --write` を使用（CI環境では読み取り専用にすべき）
- バージョンが固定されていない
- 公式アクションを使用していない

## VCS統合のベストプラクティス

### Git統合の有効化

**必須**: `.gitignore` ファイルを自動的に尊重

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

### Git Hooks設定

**推奨**: Huskyとlint-stagedを組み合わせる

**package.json:**

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

**.husky/pre-commit:**

```bash
npx lint-staged
```

## パフォーマンス最適化のルール

### 処理対象の最適化

**推奨**: ビルド成果物や依存関係を明示的に除外

```json
{
  "files": {
    "includes": [
      "src/**",
      "!**/dist/**",
      "!**/build/**",
      "!**/node_modules/**",
      "!**/.next/**",
      "!**/coverage/**",
      "!**/.git/**"
    ]
  }
}
```

### パフォーマンスの調査

遅延が発生した場合：

```bash
biome check --log-level=tracing --log-kind=json --log-file=biome.log ./src
```

### projectドメインルールの制限

**注意**: projectドメインのルールはパフォーマンスに影響するため、必要な場合のみ有効化

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

## 大規模プロジェクトでの使用

### 複数の設定ファイル

プロジェクト構造に応じて複数の `biome.json` を配置：

```
app/
├── biome.json           # 共通設定
├── backend/
│   └── biome.json      # バックエンド固有の設定
└── frontend/
    └── biome.json      # フロントエンド固有の設定
```

### 設定の継承

```json
{
  "extends": ["../biome.json"],
  "formatter": {
    "indentStyle": "space"
  }
}
```

### モノレポ設定

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
    },
    {
      "include": ["packages/ui/**"],
      "javascript": {
        "formatter": {
          "semicolons": "asNeeded"
        }
      }
    }
  ]
}
```

## コマンドのベストプラクティス

### check コマンドの活用

**推奨**: フォーマット、リント、インポート整理を一度に実行

```bash
# すべての処理を実行
biome check --write ./src

# CI環境での使用（読み取り専用）
biome ci ./src

# 変更されたファイルのみ
biome check --write --changed

# ステージされたファイルのみ
biome check --write --staged
```

### package.jsonスクリプト

**推奨設定**:

```json
{
  "scripts": {
    "check": "biome check .",
    "check:write": "biome check --write .",
    "check:ci": "biome ci .",
    "format": "biome format --write .",
    "lint": "biome lint .",
    "lint:write": "biome lint --write ."
  }
}
```

## 既存プロジェクトへの移行

### ESLint/Prettierからの移行

**推奨ステップ**:

1. 既存設定のバックアップ
2. Biomeの移行コマンド実行
3. 生成された設定の確認・調整
4. テスト実行
5. 既存ツールの無効化

```bash
# ESLintの設定を移行
npx @biomejs/biome migrate eslint --write

# インスパイアされたルールも含める
npx @biomejs/biome migrate eslint --write --include-inspired

# Prettierの設定を移行
npx @biomejs/biome migrate prettier --write
```

### 段階的な導入

**推奨アプローチ**:

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

### 既存ツールとの並行運用

移行期間中の設定例：

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

## トラブルシューティング

### よくある問題と解決策

| 問題 | 解決策 |
|------|--------|
| フォーマットが適用されない | `.vscode/settings.json` で `editor.defaultFormatter` を確認 |
| 特定ファイルが無視される | `files.includes` の設定を確認 |
| パフォーマンスが遅い | `files.includes` で除外パターンを追加、`--log-level=tracing` で調査 |
| CIで失敗する | `biome ci` コマンドを使用、`--locked` フラグは不要 |
| 設定が反映されない | `$schema` フィールドが正しいか確認、キャッシュのクリア |

## まとめ

Biomeを効果的に活用するための重要なポイント：

### 必須事項

1. **バージョン固定**: `--save-exact` でインストール
2. **設定ファイル**: プロジェクトルートに `biome.json` を配置し `$schema` を設定
3. **VCS統合**: 必ず有効化して `.gitignore` を尊重
4. **CI環境**: `biome ci` コマンドを使用
5. **エディタ統合**: 公式拡張機能をインストールし保存時フォーマットを有効化

### 推奨事項

6. **単一コマンド**: `biome check --write` で全処理を一度に実行
7. **Git Hooks**: Huskyとlint-stagedで自動チェック
8. **除外設定**: ビルド成果物を明示的に除外
9. **ルール設定**: `recommended: true` から始めて段階的に調整
10. **移行ツール**: `biome migrate` で既存設定を自動変換

### パフォーマンス上の利点

- **速度**: ESLint + Prettierの約13倍高速（450ファイルで38秒 → 2.9秒）
- **統合**: 複数ツールの設定競合を排除
- **リアルタイム**: エディタでのリアルタイムフィードバック
- **単一コマンド**: フォーマット、リント、インポート整理を一度に実行

## 参考リソース

- [Biome公式ドキュメント](https://biomejs.dev/ja/)
- [Biomeプレイグラウンド](https://biomejs.dev/playground)
- [GitHub: biomejs/biome](https://github.com/biomejs/biome)
- [Biome vs ESLint + Prettier比較](https://biomejs.dev/ja/formatter/differences-with-prettier)
- [Biome Discord コミュニティ](https://biomejs.dev/chat)
