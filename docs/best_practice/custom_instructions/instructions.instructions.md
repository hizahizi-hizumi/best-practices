---
description: 'GitHub Copilot Custom Instructions ファイル作成のベストプラクティス'
applyTo: '**/*.instructions.md, .github/copilot-instructions.md'
---

# GitHub Copilot Custom Instructions ベストプラクティス

VS Code の Custom Instructions を効果的に活用するための要点をまとめたガイドラインです。

## コア原則

### 1. 簡潔性と明確性
- 各指示は単一のシンプルな文で表現
- 複数の概念は複数の指示に分割
- 具体的で実行可能な指示を記述
- 抽象的な表現を避ける

### 2. 命令形と具体性
- "Use", "Avoid", "Validate" などの動詞で開始
- 「適切に」「よしなに」「できれば」を使わない
- 出力形式を明示（箇条書き、表、JSON等）

**推奨**: `変数名には camelCase を使用する`
**非推奨**: `適切な命名規則を使用する`

### 3. 変更の最小化
- 依頼されていないリファクタ・整形・命名変更をしない
- 置換ではなく「既存に統合」を優先
- 最小差分で目的を達成

### 4. 優先順位の明示
複数の instructions が合成される環境では、衝突時の判断基準を明記:

1. ユーザーの明示的な指示が最優先
2. セキュリティ要件は常に優先
3. プロジェクト固有のルール
4. 一般的なベストプラクティス
5. 不確かな外部情報はツールで検証

### 5. ファイルサイズ
- 300〜500行を目安に作成
- 重要性の高い要点のみを記載
- 再現性を最大限に担保

## ファイルタイプの選択

### `.github/copilot-instructions.md`
- **特徴**: ワークスペース全体に自動適用される単一ファイル
- **用途**: プロジェクト全体の基本ルール、技術スタック情報
- **設定**: `"github.copilot.chat.codeGeneration.useInstructionFiles": true`

### `.instructions.md`
- **特徴**: 複数作成可能、glob パターンで条件付き適用
- **用途**: 言語固有の規約、フレームワーク固有パターン
- **スコープ**: ワークスペース（`.github/instructions/`）またはユーザー（プロファイルフォルダ）

推奨ディレクトリ構造:
```
.github/
├── copilot-instructions.md
└── instructions/
    ├── python-standards.instructions.md
    ├── react-standards.instructions.md
    └── security-guidelines.instructions.md
```

## YAML Frontmatter

### 基本構造
```yaml
---
description: 'ファイルの目的と範囲（人間向けの簡潔な説明）'
applyTo: '**/*.py'  # glob パターン
---
```

### applyTo パターン例
```yaml
applyTo: '**/*.py'                    # Python ファイルすべて
applyTo: '**/*.{jsx,tsx}'             # React コンポーネント
applyTo: 'src/api/**/*'               # 特定ディレクトリ
applyTo: '**/*.{test,spec}.{ts,js}'   # テストファイル
```

## Instructions の書き方

### 推奨セクション
1. **目的とスコープ**: 何を制御するか1文で説明
2. **適用範囲**: どのファイル/ディレクトリに適用されるか明示
3. **プロジェクト概要**: 技術スタック、主要機能、対象ユーザー
4. **ツールとバージョン**: 正確なコマンドとバージョン情報
5. **コーディング規約**: 命名規則、スタイル、フォーマット
6. **API 契約**（オプション）: 型定義や API 仕様への参照
7. **セキュリティとプライバシー** (オプション): Secure by Default、機密情報の除外、脆弱性対策
8. **パフォーマンス最適化** (オプション): Measure First、典型的アンチパターンの回避

### Good/Bad 例の提示
```typescript
// 推奨
interface User {
  id: string;
  name: string;
}

// 非推奨
function getUser(id: any): any { }
```

### WHY（理由）の添付
重要なルールには「なぜ」を1〜2文で補足:

```markdown
- SQL クエリにはパラメタライズドクエリを使用する
  **理由**: 文字列連結は SQL インジェクション攻撃の原因となる
```

### 参照の活用
- **プロジェクト内**: `詳細は types/api.ts を参照`
- **外部**: `[Google Style Guide](https://...)`
- **コードファイル**: settings.json で `{ "file": "database/schema.sql" }` 指定

## スコープ管理

### 条件付き適用
技術領域ごとにファイルを分け、`applyTo` で必要な時だけ効かせる:

```yaml
applyTo: '**/*.prompt.md'  # 特定の拡張子に狙い撃ち
```

**効果**: 無関係な指示の混入を減らし、コンテキスト消費も抑える

### 階層的構造
```markdown
# .github/copilot-instructions.md
- DRY 原則に従う

# .github/instructions/python-standards.instructions.md
---
applyTo: '**/*.py'
---
プロジェクト全体のルールに加えて:
- PEP 8 に準拠
- 型ヒントを使用
```

## ファイル作成時の注意点

### 信頼境界の設定

**Workspace Trust**: 信頼されていないワークスペースでは機能制限を推奨
**ツール自動承認**: 破壊的操作には確認ステップを必須化

```json
{
  "github.copilot.chat.tool.approval": "user",  // 推奨
  "github.copilot.chat.terminal.autoApprove": {
    "commands": ["git status", "npm test"]  // 最小権限
  }
}
```

## チーム開発での運用

### バージョン管理
```gitignore
# ユーザー固有を除外
.vscode/settings.json

# チーム共有を含める
!.github/copilot-instructions.md
!.github/instructions/
!.github/prompts/
```

### 定期メンテナンス
3〜6ヶ月ごと:
- 不要ルールの削除
- 新ベストプラクティスの追加
- チームフィードバックの反映
- プロジェクト変化との整合性確認

### オンボーディング
新メンバー向けに:
- Instructions ファイルの場所と目的
- 有効化に必要な設定
- 利用可能な prompts

### PR テンプレート統合
```markdown
## Copilot チェックリスト
- [ ] 生成コードをレビュー済み
- [ ] セキュリティガイドライン準拠
- [ ] 意図しない変更なし（最小変更原則）
```

## トラブルシューティング

### Instructions が効かない場合
1. **設定確認**: `useInstructionFiles: true` か
2. **配置確認**: `.github/copilot-instructions.md` の場所
3. **パターン確認**: `applyTo: '**/*.py'` が正しいか

### 期待と異なる生成
1. 指示の明確性を確認（曖昧語排除）
2. Good/Bad 例を追加
3. 関連ファイルを参照
4. 優先順位を明示

### 共通アンチパターン

#### 冗長な説明
```markdown
# 非推奨: 1文に複数要素
すべての関数には適切なドキュメント、型、エラー処理を...

# 推奨: 分割
- 関数に JSDoc を記述
- 型を明示
- エラーケースを記載
```

#### 曖昧な指示
```markdown
# 非推奨
良いコードを書く

# 推奨
関数は単一責任原則に従う
```

#### 矛盾する指示
```markdown
# 問題: 同じスコープで矛盾
ファイル1: 詳細コメント必須
ファイル2: コメント最小限

# 解決: 優先順位明示
1. 自己説明的コード優先
2. 「なぜ」のみコメント
```

## 参考リソース

- [VS Code - Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [awesome-copilot](https://github.com/github/awesome-copilot)
