# Prompt Files コミュニティベストプラクティス（awesome-copilot 分析）

## 概要

このドキュメントは、GitHub awesome-copilot リポジトリの 126個の `.prompt.md` ファイルを分析し、実際のコミュニティユースケースから抽出したベストプラクティスをまとめたものです。公式ドキュメントとは異なり、実運用での書き方・構造・パターンに焦点を当てます。

参照: [docs/best_practice/prompt_files/url.2.md](url.2.md)

---

## 1. frontmatter の実践パターン

### 必須フィールドの使い方

#### `description`
- **目的**: ユーザーが一覧で見たときに「何ができるか」を即座に判断できる
- **長さ**: 実例では 50-150文字程度が主流
- **書き方のパターン**:
  - 明確な動詞で開始: "Create...", "Generate...", "Review...", "Analyze..."
  - 達成する成果物を明記: "...for AI-optimized decision documentation"
  - 対象技術やドメインを含める: "...for .NET", "...with XUnit"

例（実例より）:
```yaml
description: 'Create an Architectural Decision Record (ADR) document for AI-optimized decision documentation.'
```

```yaml
description: 'Add educational comments to the file specified, or prompt asking for file to comment if one is not provided.'
```

```yaml
description: 'Technology-agnostic blueprint generator for creating comprehensive copilot-instructions.md files...'
```

#### `agent`
- **実践の傾向**:
  - `agent: 'agent'` が最も多い（自律的な複数ステップ実行）
  - `ask` は質問・分析系（コード変更なし）に使用
  - `edit` は直接的なファイル編集に特化
  - カスタムエージェント名は稀（ほとんどが標準agent）

#### `tools`
- **最小権限の実例**:
  - Read-only: `['search', 'search/codebase', 'usages']`
  - Documentation系: `['edit/editFiles', 'fetch', 'todos']`
  - Full workflow: `['changes', 'search/codebase', 'edit/editFiles', 'problems', 'githubRepo', 'runCommands/...']`
  
- **パターン**: タスクに必要な最小セットを列挙し、破壊的操作を含む場合は本文で確認ステップを明記

例:
```yaml
tools: ['changes', 'search/codebase', 'edit/editFiles', 'problems', 'search']
```

### オプショナルフィールド

#### `name`
- ファイル名で十分なら省略される例が多い
- 明示する場合は短く呼びやすい名前（kebab-case推奨）

#### `model`
- ほとんどの実例で**指定されていない**（ユーザーのモデルピッカーに委ねる）
- 特定機能が必要な場合のみ明記

---

## 2. 本文構造の実践パターン

### 主要セクション構成（頻出順）

1. **Role / Purpose**
   - プロンプトが想定する役割（"You are an expert educator...", "You're a senior expert software engineer..."）
   - 目標を明確化

2. **Task / Objectives**
   - 何をするか（ステップバイステップ）
   - 達成する成果物の定義

3. **Inputs / Configuration**
   - 入力変数の一覧（`${input:...}`）
   - デフォルト値と選択肢
   - 入力検証ルール

4. **Workflow / Process**
   - 段階的な手順（Phase 1, Phase 2...）
   - 各ステップの詳細と停止条件

5. **Requirements / Rules**
   - 制約条件、禁止事項
   - セキュリティ/品質基準

6. **Output / Expected Result**
   - 成果物のフォーマット
   - 保存先とファイル命名規則

7. **Examples** (optional)
   - before/after、good/bad の比較
   - 具体的なコード片

8. **Validation / Checklist** (optional)
   - 完了確認項目
   - テスト/検証コマンド

### 実例: ADR作成プロンプト（create-architectural-decision-record.prompt.md）

```markdown
# Create Architectural Decision Record

## Inputs
- **Context**: `${input:Context}`
- **Decision**: `${input:Decision}`
- **Alternatives**: `${input:Alternatives}`
- **Stakeholders**: `${input:Stakeholders}`

## Input Validation
If any of the required inputs are not provided or cannot be determined 
from the conversation history, ask the user to provide the missing 
information before proceeding with ADR generation.

## Requirements
- Use precise, unambiguous language
- Follow standardized ADR format with front matter
- Include both positive and negative consequences
- Document alternatives with rejection rationale
- Structure for machine parsing and human reference
- Use coded bullet points (3-4 letter codes + 3-digit numbers) for multi-item sections

The ADR must be saved in the `/docs/adr/` directory using the naming 
convention: `adr-NNNN-[title-slug].md`...

## Required Documentation Structure
[Template with frontmatter, sections, codes...]
```

**学び**:
- 入力が必須の場合は検証ロジックを明記
- 保存先とファイル命名規則を厳密に指定
- 構造化されたテンプレートを埋め込む

---

## 3. 入力変数とコンフィギュレーション

### 設定可能変数の設計パターン

**Configuration Variablesセクション** の実例（copilot-instructions-blueprint-generator.prompt.md）:

```markdown
## Configuration Variables

${PROJECT_TYPE="Auto-detect|.NET|Java|JavaScript|TypeScript|React|Angular|Python|Multiple|Other"}
${ARCHITECTURE_STYLE="Layered|Microservices|Monolithic|Domain-Driven|Event-Driven|Serverless|Mixed"}
${CODE_QUALITY_FOCUS="Maintainability|Performance|Security|Accessibility|Testability|All"}
${DOCUMENTATION_LEVEL="Minimal|Standard|Comprehensive"}
${TESTING_REQUIREMENTS="Unit|Integration|E2E|TDD|BDD|All"}
${VERSIONING="Semantic|CalVer|Custom"}
```

**学び**:
- 選択肢をパイプ区切りで列挙してユーザーに明示
- デフォルト値を最初に置く（"Auto-detect"など）
- 本文で条件分岐: `${PROJECT_TYPE == ".NET" ? "..." : ""}`

### 入力が欠けた場合の対応

```markdown
## Workflow
1. **Confirm Inputs** – Ensure at least one target file is provided. 
   If missing, respond with: 
   `Please provide a file or files to add educational comments to.`
```

**学び**: 入力不足時のメッセージをプロンプト内で定義し、エージェントに質問させる

---

## 4. ワークフロー設計の実践

### フェーズ分割パターン

大規模な変換・生成タスクでは段階を明確化:

```markdown
### Phase 1: Comparative State Analysis
- Compare folder structure between ${SOURCE_REFERENCE} and ${TARGET_REFERENCE}
- Identify moved, renamed, or deleted files
- Analyze changes in configuration files
- Document new dependencies and removed ones

### Phase 2: Migration Instructions Generation
Create a `.github/copilot-migration-instructions.md` file with this structure...

### Phase 3: Contextual Examples Generation
For each identified pattern, generate:
   // BEFORE (${SOURCE_REFERENCE})
   [OLD_CODE_EXAMPLE]
   // AFTER (${TARGET_REFERENCE}) 
   [NEW_CODE_EXAMPLE]

### Phase 4: Validation and Optimization
- Apply instructions on test code
- Verify transformation consistency
- Adjust rules based on results
```

**学び**:
- 各フェーズで何をするか・何を出力するかを明確にする
- 検証フェーズを最後に含める

### 反復と条件付き実行

```markdown
2. **Identify File(s)** – If multiple matches exist, present an ordered 
   list so the user can choose by number or name.
3. **Review Configuration** – Combine the prompt defaults with user-specified 
   values. Interpret obvious typos (e.g., `Line Numer`) using context.
```

**学び**: あいまいな状況（複数候補、typo）への対処手順も含める

---

## 5. セキュリティと制約の明示

### 許可・禁止アクションの列挙

Blueprint生成プロンプトでは、Copilot が自律的に実行できる範囲を明記:

```markdown
### Allowed autonomous actions
- Precise list (typo fixes, docs updates, single-file refactors with tests, formatting).
- Include limits (e.g., max files changed, no infra changes).

### Disallowed actions / require human review
- DB schema/migrations, secrets, production config, large refactors, major dependency upgrades.
```

**学び**: "何ができるか" と "何をしてはいけないか" を対で記述し、agent mode での暴走を防ぐ

### 安全性ルール（Educational Commentsの例）

```markdown
## Educational Commenting Rules
### Safety and Compliance
- Do not alter namespaces, imports, module declarations, or encoding headers 
  in a way that breaks execution.
- Avoid introducing syntax errors
- Input data as if typed on the user's keyboard.
```

**学び**: 破壊的変更の禁止事項をルール化

---

## 6. テンプレートとスニペットの活用

### 埋め込みテンプレート

ADR作成プロンプトでは、最終成果物のMarkdownテンプレート全文を本文に含める:

```markdown
## Required Documentation Structure
The documentation file must follow the template below...

\`\`\`md
---
title: "ADR-NNNN: [Decision Title]"
status: "Proposed"
date: "YYYY-MM-DD"
...
\`\`\`
```

**学び**: 出力フォーマットを全文で示すことで、Copilot が正確に再現できる

### 条件分岐でテンプレート出し分け

```markdown
${CODE_QUALITY_FOCUS.includes("Maintainability") || CODE_QUALITY_FOCUS == "All" ? 
`### Maintainability
- Write self-documenting code with clear naming
- Follow the naming and organization conventions evident in the codebase
...` : ""}
```

**学び**: 設定に応じて必要なセクションのみ生成

---

## 7. ドキュメント参照と外部リンク

### 公式ドキュメントへのリンク

```markdown
- Fetch List:
  - <https://peps.python.org/pep-0263/>
```

XUnit プロンプトでは標準プラクティスを参照する形で記載:

```markdown
## Project Setup
- Use a separate test project with naming convention `[ProjectName].Tests`
- Reference Microsoft.NET.Test.Sdk, xunit, and xunit.runner.visualstudio packages
```

**学び**: 権威ある外部情報をリンクし、プロンプト本文は「何をするか」に絞る

### 既存コードベースからのパターン抽出

```markdown
## Codebase Scanning Instructions
When context files don't provide specific guidance:
1. Identify similar files to the one being modified or created
2. Analyze patterns for:
   - Naming conventions
   - Code organization
   - Error handling
3. Follow the most consistent patterns found in the codebase
```

**学び**: プロジェクト固有の規約は「コードを読んで学習させる」指示を入れる

---

## 8. エラー処理と失敗時の挙動

### 入力不足・検証失敗

```markdown
## Input Validation
If any of the required inputs are not provided..., ask the user to 
provide the missing information before proceeding.
```

### 自動/手動エスカレーション

```markdown
## Validation and Security
### Manual Escalation
Situations requiring human intervention:
- [COMPLEX_CASES_LIST]
- [ARCHITECTURAL_DECISIONS]
- [BUSINESS_IMPACTS]
```

**学び**: 自動化できる範囲を超えたら「停止して人間に質問」を設計に組み込む

---

## 9. 行数・量的制約の管理

Educational Commentsプロンプトでは明確な行数ルールを設定:

```markdown
## Objectives
3. Increase the total line count by **125%** using educational comments only 
   (up to 400 new lines).

### Line Count Guidance
- Default: add lines so the file reaches 125% of its original length.
- Hard limit: never add more than 400 educational comment lines.
- Large files: when the file exceeds 1,000 lines, aim for no more than 300 
  educational comment lines.
```

**学び**: 量的な制約（行数・ファイル数・変更範囲）を数値で明示する

---

## 10. 成果物の具体例（before/after）

### マイグレーション系プロンプト

```markdown
#### Transformation Examples
For each identified pattern, generate:

\`\`\`
// BEFORE (${SOURCE_REFERENCE})
[OLD_CODE_EXAMPLE]

// AFTER (${TARGET_REFERENCE}) 
[NEW_CODE_EXAMPLE]

// COPILOT INSTRUCTIONS
When you see this pattern [TRIGGER], transform it to [NEW_PATTERN] 
following these steps: [STEPS]
\`\`\`
```

**学び**: 変換ルールを "BEFORE → AFTER → 手順" のセットで示す

---

## 11. メタデータ・バージョン管理

### 運用情報の埋め込み

実例では frontmatter や本文にメンテ情報を含むものが多い:

```markdown
---
title: "ADR-NNNN: [Decision Title]"
status: "Proposed"
date: "YYYY-MM-DD"
authors: "[Stakeholder Names/Roles]"
tags: ["architecture", "decision"]
supersedes: ""
superseded_by: ""
---
```

**学び**: Prompt Files 自体をバージョン管理し、成果物にもメタデータを埋め込む

---

## 12. 複数ファイル対応とスケーラビリティ

### ファイル選択ロジック

```markdown
2. **Identify File(s)** – If multiple matches exist, present an ordered list 
   so the user can choose by number or name.
```

### ディレクトリ構造の整理

```markdown
.github/
├── copilot-instructions.md
├── instructions/
│   ├── python-standards.instructions.md
│   └── ...
├── prompts/
│   ├── create-component.prompt.md
│   └── ...
```

**学び**: プロンプトをカテゴリ別に整理し、関連 instructions をリンク参照

---

## 13. コミュニティの傾向まとめ

### 多く見られるユースケース（カテゴリ別）

1. **Blueprint/Template生成**: README、ADR、folder-structure、instructions生成
2. **コード品質**: Review-and-refactor、documentation、educational-comments
3. **テスト生成**: xUnit、JUnit、Playwright等のフレームワーク別ベストプラクティス
4. **マイグレーション/リファクタリング**: フレームワークバージョンアップ、アーキテクチャ変更の自動適用
5. **GitHub連携**: Issue/PR作成、conventional-commit、workflow仕様生成
6. **MCP Server生成**: 言語別（C#, Java, Kotlin）のMCPサーバースキャフォールド

### よく使われる構造パターン

- **Role → Task → Workflow → Output**: 最も標準的な流れ
- **Configuration Variables → Generated Prompt → Expected Output**: 設定駆動型の生成プロンプト
- **Inputs → Validation → Requirements → Template**: データ駆動型のドキュメント生成

### ツール選択の傾向

- 調査・分析系: `search`, `search/codebase`, `usages`, `githubRepo`
- 編集系: `edit/editFiles`, `changes`
- 統合系: `runCommands/terminalLastCommand`, `problems`, `testFailure`

### agent vs ask/edit の使い分け

- `agent`: 複数フェーズのワークフロー、ファイル/リソース横断の操作
- `ask`: 質問回答、コードレビュー、推奨提案（変更なし）
- `edit`: 単一ファイルの直接編集、フォーマット、ドキュメント追記

---

## 14. アンチパターン（コミュニティ実例で避けられている点）

- **曖昧な description**: 「Helps with...」「Does stuff」→ 明確な動詞と成果物を書く
- **tools の全指定**: すべてのツールを列挙しない（必要最小限に絞る）
- **エラーハンドリングの欠如**: 入力不足や検証失敗時の挙動が未定義
- **巨大な一枚岩プロンプト**: 段階を分けず、全ステップを1つに詰め込む
- **テンプレートの省略**: 出力フォーマットを「よしなに」任せる（再現性が落ちる）

---

## 15. 実践への活かし方

### 新規プロンプト作成時のチェックリスト（コミュニティパターン準拠）

- [ ] `description` に動詞+成果物+対象技術を含めた
- [ ] `agent` をタスク特性に合わせて選択（ask/edit/agent）
- [ ] `tools` を最小権限で列挙した
- [ ] Role/Purpose セクションで役割を明確化した
- [ ] Inputs に `${input:...}` と検証ロジックを書いた
- [ ] Workflow を段階（Phase）に分割した
- [ ] Requirements に制約・禁止事項を列挙した
- [ ] Output にフォーマット/保存先を明記した
- [ ] テンプレートやコード例を埋め込んだ
- [ ] 失敗時・入力不足時の挙動を定義した
- [ ] 外部ドキュメントや既存コードへの参照を追加した

### 既存プロンプトの改善ポイント

1. **入力検証**: `## Input Validation` セクションがあるか？
2. **フェーズ分割**: 長いワークフローが Phase 1/2/3 に分かれているか？
3. **量的制約**: "最大○行"、"最大○ファイル" の上限があるか？
4. **エスカレーション**: 人間介入が必要な条件を明記しているか？
5. **テンプレート**: 成果物の全構造を示しているか？

---

## 参考リンク

- [awesome-copilot/prompts](https://github.com/github/awesome-copilot/tree/main/prompts) - 126個のコミュニティプロンプト
- [url.2.md](url.2.md) - 本文書で参照した全URLリスト
- [公式 Prompt Files ドキュメント](https://code.visualstudio.com/docs/copilot/customization/prompt-files)
- [Microsoft DevBlogs: Prompt Files and Instructions Files Explained](https://devblogs.microsoft.com/dotnet/prompt-files-and-instructions-files-explained/)

---

最終更新: 2026年1月4日
