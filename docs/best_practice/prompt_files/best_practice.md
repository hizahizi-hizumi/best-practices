# VS Code Copilot Prompt Files（.prompt.md）のベストプラクティス

## 概要

Prompt Files は、VS Code の Copilot Chat で「繰り返し実行するタスク」を再利用可能なプロンプトとして定義する仕組みです。Custom Instructions（常時適用する規約）とは違い、Prompt Files は `/` で呼び出す「オンデマンド」なワークフローのテンプレートに向きます。

- 対象ファイル: `.prompt.md`
- 目的: 再現性（同じ入力で同じ成果）と安全性（最小権限・承認の明確化）

## 使い分け（Custom Instructions / Prompt Files / Custom Agents）

- Custom Instructions: リポジトリ全体の共通ルール（命名、テスト、出力形式など）
- Prompt Files: タスク単位の手順書（生成・レビュー・移行など）
- Custom Agents: 役割とツールセット（Plan/Implement/Review など）を切り替える

推奨:
- 共通規約は instructions に集約し、Prompt Files はそれを Markdown リンクで参照して重複を避ける

## 配置・スコープ・共有

### 既定の配置
- ワークスペース: `.github/prompts/` 配下（このワークスペース内で利用）
- ユーザープロファイル: VS Code のプロファイル配下（複数ワークスペースで利用）

### 探索場所の設定
- `chat.promptFilesLocations`: Prompt Files を探索するフォルダを追加/変更できる
- `chat.promptFilesRecommendations`: 新しいチャット開始時に「おすすめプロンプト」を出す（条件付きも可能）

### 同期
- ユーザープロファイル側の Prompt Files は Settings Sync（Prompts and Instructions）で端末間同期できる

## ファイル構造（YAML frontmatter + Markdown body）

Prompt Files は Markdown で、先頭に任意の YAML frontmatter を置けます。

### frontmatter（推奨項目）

公式で言及される主なフィールド:

- `description`: 短い説明（一覧や推奨表示で有用）
- `name`: `/` の後に入力する名前（未指定ならファイル名が使われる）
- `argument-hint`: 入力欄に表示されるヒント（利用者が何を入れるべきか明確化）
- `agent`: ask / edit / agent / custom agent 名
  - `tools` を指定していて、現在の agent が ask/edit の場合、既定 agent は `agent` になり得る
- `model`: 使うモデル（未指定ならモデルピッカーの選択に従う）
- `tools`: この Prompt Files で利用可能なツール/ツールセット
  - MCP 全ツールを許可する場合は `<serverName>/*` 形式を使える
  - 指定ツールが利用できない場合は無視される

#### ツール優先順位（重要）
ツールの有効化は次の優先順位で決まります:
1. Prompt Files で指定した `tools`
2. `agent` が参照する Custom Agent の `tools`
3. 選択中の built-in agent の既定ツール

結論:
- Prompt Files 側の `tools` は「最小権限」で明示する
- 迷ったら、まず read-only（検索/読み取り）だけにしてから段階的に増やす

## 変数・入力の設計

### 組み込み変数（例）
Prompt Files では `${...}` 構文で変数を参照できます。

- Workspace: `${workspaceFolder}`, `${workspaceFolderBasename}`
- Selection: `${selection}`, `${selectedText}`
- File context: `${file}`, `${fileBasename}`, `${fileDirname}`, `${fileBasenameNoExtension}`

推奨:
- 変数は「必要な箇所だけ」に限定し、意味（何が入る想定か）を本文に明記する
- `${selection}` を使う場合は「ユーザーが何を選択するべきか」も指示する

### 入力変数（Prompt のパラメータ化）
- `${input:variableName}`
- `${input:variableName:placeholder}`

推奨:
- 必須入力は `argument-hint` とセットで要求する
- 入力が足りない場合の挙動を本文で規定する（例: “不足している値を質問して停止”）

## 本文（body）の書き方

### 基本方針
- 最初に「ゴール」と「成果物（出力形式）」を固定する
- 手順を段階化し、途中で確認・停止条件を入れる
- 共通規約はリンク参照し、本文をタスク固有の情報に絞る

### 運用メタデータ（推奨）

Prompt Files は「繰り返し呼び出す手順書」になりやすいため、本文冒頭に運用メタデータを置くと保守しやすくなります。

- バージョン/最終更新日/メンテナ（連絡先）
- 適用範囲（どのパス/コンポーネント向けか）
- 許可する自動アクション（例: 単一ファイルの小改修、ドキュメント更新、read-onlyの調査）
- 禁止/要レビューのアクション（例: DBマイグレーション、依存関係の大幅更新、セキュリティ関連設定、インフラ変更）
- ビルド/テスト/フォーマットの最小コマンド（「何を通ればOKか」を明確化）
- 失敗時の扱い（例: 失敗時は変更を提案に留めて停止、ロールバック方針）

補足:
- frontmatter に独自フィールドを追加しても VS Code 側で無視される可能性があるため、重要情報は本文にも書くのが安全です。

### 参照（リンク）
- 他のファイル（instructions や設計資料）を Markdown 相対リンクで参照できる
- 重複排除のため、チーム標準は instructions に置き、Prompt Files はリンクする

### ツール参照
- Prompt Files 本文内でツールを参照する場合は `#tool:<tool-name>` を使う
- ただし、外部URL取得などは承認フローがあるため、無批判にチェーンしない（後述）

## セキュリティと運用上の注意（重要）

Prompt Files は「AI の指示書」である一方、Agent モードやツールと組み合わさるとファイル変更や外部アクセスを伴います。

### 承認・最小権限
- `tools` は最小セットにする（不要なツールを有効にしない）
- 破壊的操作（大量変更・削除・実行系）は手順に明示的な確認ステップを入れる

### URL approval と prompt injection
- Web取得（`fetch` など）には URL の事前承認・事後承認がある
- 事後承認は、信頼できるドメインでも「ユーザー生成コンテンツ」により prompt injection のリスクがある

推奨:
- 取得した外部テキストを“指示”として扱わない（一次情報として要約・抽出に限定する）
- ツール出力の連鎖（tool output chaining）を前提にした自動化は避け、要点の抽出→人間レビュー→次ステップの順にする

### 自動承認のリスク
- 全ツール自動承認（例: `chat.tools.global.autoApprove`）は重要な保護を無効化する
- ターミナル自動承認は検知漏れがあり得るため、防御策（deny ルール、限定許可、レビュー）を前提にする

### 情報露出（機密情報）
- 環境変数、設定、コード、外部サービス情報がモデルやツールのコンテキストに混入しうる
- Prompt Files に秘密情報（APIキー等）を埋め込まない

## Prompt Files と Instructions の適用関係（整理）

- Instructions は、基本的にすべてのチャット要求に対して“常時”適用される
- Prompt Files は `/...` で“明示的に呼び出したときだけ”適用される

推奨:
- 共有したい共通ルールは instructions に置く
- タスク固有の手順（入力、段取り、成果物、停止条件）は Prompt Files に置く

## 反復と資産化

### 実行とテスト
- `/promptName` で呼び出して結果を確認し、必要なら Prompt Files を更新して反復する
- エディタの再生ボタン（Run）で同一セッション/新規セッションで試せる

### 会話から Prompt Files を生成
- チャットでうまくいった手順は `/savePrompt` で `.prompt.md` 化して再利用する
- 生成されたファイルは「一般化・最小権限・入力変数化」を行ってから採用する

## よくある落とし穴と回避策

- ツールを広く許可しすぎる
  - 回避: `tools` を絞り、必要時だけ増やす（Prompt Files 側が最優先で効く）
- 入力が曖昧（何を渡すべきか不明）
  - 回避: `argument-hint` と `${input:...}` を使い、欠落時の質問・停止条件を明記
- 大量の規約を Prompt Files にコピペして肥大化
  - 回避: instructions に集約し、Prompt Files はリンク参照
- 外部取得テキストをそのまま指示として扱う
  - 回避: prompt injection 前提で、抽出/要約に限定し、次の操作は人間のレビュー後に

## 例（最小構成）

```markdown
---
description: 'コード変更の影響範囲を調査して要約する'
name: 'impact-summary'
argument-hint: '対象（PR番号/変更ファイル/機能名）を入力'
agent: 'agent'
tools: ['search', 'usages', 'read']
---
# 影響範囲サマリ

目的: 入力された対象について、影響箇所と確認観点を箇条書きでまとめる。

## Inputs
- 対象: ${input:target:例) auth, #123, src/foo.ts}

## Workflow
1. まず #tool:search と #tool:usages で関連箇所を洗い出す。
2. 重要ファイルのみ #tool:read で確認する。
3. 影響範囲、破壊的変更の可能性、追加すべきテスト観点を出力する。

## Output
- 変更点（要約）
- 影響範囲（ファイル/モジュール）
- リスクと確認項目
```

---

最終更新: 2026年1月4日
