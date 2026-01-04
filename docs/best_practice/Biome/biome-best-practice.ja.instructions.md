---
description: 'JavaScript/TypeScriptプロジェクトのためのBiomeフォーマッターとリンターのベストプラクティス'
applyTo: 'biome.json, biome.jsonc, **/*.{js,ts,jsx,tsx,json}'
---

# Biome ベストプラクティス

Biomeを高性能なコード品質ツール（ESLint + Prettierより13倍高速）として使用するためのガイドライン。

## 目的と範囲

JavaScript/TypeScriptプロジェクトにおけるフォーマット、リンティング、インポート整理にBiomeを使用する。
これらのルールは、Biomeが管理する設定ファイルとソースコードに適用される。

## プロジェクト概要

**ツール**: Biome - Rust製のフォーマッター/リンター、Prettierとの互換性97%
**パフォーマンス**: 450ファイルを2.9秒で処理（ESLint + Prettierは38秒）
**統合**: ESLint + Prettier + インポート整理ツールを1つのツールで代替

## コア原則

### 優先順位
1. セキュリティ: セキュリティ関連のルールは明確な正当化なしに無効化しない
2. パフォーマンス: ビルド成果物とnode_modulesを明示的に除外する
3. チームの一貫性: バージョンロックのために`--save-exact`を使用する
4. VCS統合: `.gitignore`の尊重を自動的に有効化する
5. CI/CD: 継続的インテグレーションでは`biome ci`（読み取り専用モード）を使用する

### 最小限の変更原則
フォーマット/リンティングが変更を要求しない限り、既存のコード構造を保持する。
変更されたファイルのみを対象とするには`biome check --write --changed`を使用する。


## インストールとセットアップ

### バージョンロックを使用したインストール
```bash
npm install --save-dev --save-exact @biomejs/biome
```
**根拠**: チームメンバーとCI環境間での微妙な動作の違いを防ぐ。

### 設定の初期化
```bash
npx @biomejs/biome init        # JSON形式
npx @biomejs/biome init --jsonc  # コメント付きJSONC
```

## 設定ファイル (biome.json)

### 必須フィールド
IDEのオートコンプリートとバリデーションのために`$schema`を含める。
`.gitignore`を自動的に尊重するためにVCS統合を有効化する。
ベースラインとしてリンタールールの`recommended: true`を設定する。

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
  }
}
```

### ビルド成果物の除外
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
**根拠**: パフォーマンスを向上させ、生成されたコードのリンティングを防ぐ。

## フォーマットルール

### Biomeのデフォルトを使用する
チームの議論を減らすために設定を最小限にする。
プロジェクトの規約が逸脱を要求する場合のみオーバーライドする。

| オプション | デフォルト | 説明 |
|--------|---------|-------------|
| `indentStyle` | `"tab"` | タブまたはスペースを使用 |
| `indentWidth` | `2` | インデントレベルごとのスペース数 |
| `lineWidth` | `80` | 行の折り返し列 |
| `lineEnding` | `"lf"` | Unix形式の改行 |

### JavaScript固有のオプション
```json
{
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always"
    }
  }
}
```

### フォーマットの選択的な抑制
```javascript
// biome-ignore format: Matrix layout must be preserved for readability
const matrix = [
  (2 * n) / (r - l), 0, (r + l) / (r - l), 0,
  0, (2 * n) / (t - b), (t + b) / (t - b), 0,
];
```
**ルール**: `biome-ignore`コメントには常に具体的な正当化を提供する。

## リンティングルール

### 推奨ルールの有効化
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
        "useConst": "error"
      },
      "correctness": {
        "noUnusedVariables": "error"
      }
    }
  }
}
```

### 重要度レベル
- `"error"`: CIを失敗させる（正確性とセキュリティの問題に使用）
- `"warn"`: 情報提供（スタイル設定に使用）
- `"info"`: 低優先度の提案
- `"off"`: ルールを完全に無効化

### 正当化を伴うリンティングの抑制
```javascript
// biome-ignore lint/suspicious/noDebugger: Debugging production issue #1234
debugger;

// biome-ignore lint/suspicious/noConsoleLog: Required for troubleshooting logs
console.log("User action:", action);
```
**ルール**: 具体的な理由なしで`biome-ignore`を使用しない。

## コマンド

### 統合チェックコマンド
```bash
biome check --write ./src          # フォーマット、リンティング、インポート整理
biome check --write --changed      # 変更されたファイルのみ
biome check --write --staged       # ステージされたファイルのみ
biome ci ./src                     # CIモード（読み取り専用、問題があれば失敗）
```
**根拠**: 単一のコマンドが複雑さを軽減し、一貫性を保証する。

### フォーマットのみのコマンド
```bash
biome format ./src                 # フォーマットのみをチェック
biome format --write ./src         # フォーマットを適用
```

### リンティングのみのコマンド
```bash
biome lint ./src                   # リンティングのみをチェック
biome lint --write ./src           # 安全な修正を適用
biome lint --write --unsafe ./src  # 安全でない修正を適用（注意）
```

### 推奨されるpackage.jsonスクリプト
```json
{
  "scripts": {
    "check": "biome check .",
    "check:write": "biome check --write .",
    "check:ci": "biome ci .",
    "format": "biome format --write .",
    "lint": "biome lint --write ."
  }
}
```

## エディタ統合

### VS Code設定
公式のBiome拡張機能（`biomejs.biome`）をインストールする。
チームの一貫性のために`.vscode/settings.json`を設定する。

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

## CI/CD統合

### GitHub Actions
最適なキャッシングのために公式の`biomejs/setup-biome`アクションを使用する。

```yaml
name: Code Quality
on: [push, pull_request]

jobs:
  biome:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: biomejs/setup-biome@v2
        with:
          version: latest
      - run: biome ci .
```
**根拠**: 公式アクションがより優れたパフォーマンスとキャッシングを提供する。

### 避けるべき一般的なCIの間違い
```yaml
# 悪い例: CIで--writeを使用（ファイルを変更する）
- run: biome check --write .

# 良い例: ciコマンドを使用（読み取り専用）
- run: biome ci .
```

## Git Hooks

### HuskyとLint-stagedを使用したPre-commit
```json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{js,ts,jsx,tsx,json}": [
      "biome check --write --no-errors-on-unmatched"
    ]
  }
}
```

**.husky/pre-commit:**
```bash
npx lint-staged
```

## ESLint/Prettierからの移行

### 自動移行
```bash
biome migrate eslint --write              # ESLint設定を移行
biome migrate eslint --write --include-inspired  # インスパイアされたルールを含める
biome migrate prettier --write            # Prettier設定を移行
```

### 段階的な導入戦略
1. まずフォーマッターを有効にし、既存のリンターを維持する
2. リンタールールを`"warn"`レベルで有効にする
3. 警告を徐々にエラーに昇格させる
4. バリデーション後にESLint/Prettierを削除する

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

### 移行中の並行運用
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

## パフォーマンス最適化

### ソース以外のディレクトリを除外する
ビルド成果物、依存関係、生成されたファイルを明示的に除外する。

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

### パフォーマンス問題の調査
```bash
biome check --log-level=tracing --log-kind=json --log-file=biome.log ./src
```

### プロジェクトスコープルールの制限
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
**根拠**: プロジェクトスコープルールは完全な型チェックを必要とし、パフォーマンスに影響する。

## モノレポ設定

### パッケージ固有のルールにはオーバーライドを使用する
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

### 設定の継承
```json
{
  "extends": ["../biome.json"],
  "formatter": {
    "indentStyle": "space"
  }
}
```

## トラブルシューティング

### よくある問題

| 問題 | 解決策 |
|-------|----------|
| フォーマットが適用されない | VS Code設定の`editor.defaultFormatter`を確認する |
| 特定のファイルが無視される | biome.jsonの`files.includes`パターンを確認する |
| パフォーマンスが遅い | `files.includes`に除外を追加し、`--log-level=tracing`で調査する |
| CIの失敗 | `biome check --write`の代わりに`biome ci`コマンドを使用する |
| 設定が読み込まれない | `$schema`フィールドを確認し、エディタのキャッシュをクリアする |

## アンチパターン

### 設定スキーマの欠落
```json
// 悪い例: スキーマなし
{
  "formatter": {
    "enabled": true
  }
}

// 良い例: IDEサポートのためのスキーマあり
{
  "$schema": "https://biomejs.dev/schemas/2.0.5/schema.json",
  "formatter": {
    "enabled": true
  }
}
```

### VCS統合の無効化
```json
// 悪い例: VCS統合が無効
{
  "vcs": {
    "enabled": false
  }
}

// 良い例: VCS統合が有効
{
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  }
}
```

### CIで--writeを使用
```bash
# 悪い例: CIでファイルを変更する
biome check --write .

# 良い例: CIで読み取り専用チェック
biome ci .
```

### 正当化なしでの無視
```javascript
// 悪い例: 理由が提供されていない
// biome-ignore lint
console.log("debug");

// 良い例: 具体的な正当化
// biome-ignore lint/suspicious/noConsoleLog: Required for debugging issue #1234
console.log("debug");
```

## リファレンス

- [Biome Official Documentation](https://biomejs.dev/)
- [Biome Playground](https://biomejs.dev/playground)
- [GitHub: biomejs/biome](https://github.com/biomejs/biome)
- [Biome vs Prettier Differences](https://biomejs.dev/formatter/differences-with-prettier)
