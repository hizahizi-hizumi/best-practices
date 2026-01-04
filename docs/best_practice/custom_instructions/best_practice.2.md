---
post_title: "custom_instructions: コミュニティユースケースのベストプラクティス"
author1: "moritakohey"
post_slug: "custom-instructions-community-best-practices"
microsoft_alias: "moritakohey"
featured_image: ""
tags:
  - copilot
  - custom-instructions
  - vscode
  - community
ai_note: "AI支援で作成"
summary: "awesome-copilot の *.instructions.md 実例から、カスタムインストラクション設計・運用の共通パターンを要約する。"
post_date: "2026-01-04"
---

## 概要

このドキュメントは、GitHub の [awesome-copilot](https://github.com/github/awesome-copilot) リポジトリの
[instructions ディレクトリ](https://github.com/github/awesome-copilot/tree/main/instructions) にある
コミュニティ提供の `*.instructions.md` 実例を読み取り、共通する設計・記述・運用パターンを要約したものです。

補足:
- 同ディレクトリ配下に、厳密なファイル名 `instructions.md` は見当たりませんでした。
- 実例は `*.instructions.md`（例: `security-and-owasp.instructions.md`）として提供されているため、本資料ではこれらを分析対象とします。

## コミュニティで頻出する「強い」原則

### 1. 指示の優先順位（ヒエラルキー）を明確にする

- 「ユーザーの明示指示が最優先」のように、衝突時の判断基準を先頭に置く
- 「不確かな外部情報はツールで検証」など、事実確認の優先度を明文化する

狙い:
- 複数の instructions が合成された時でも、挙動がブレにくくなる

### 2. 変更は最小・局所（Surgical）にする

- 依頼されていないリファクタ・整形・命名変更をしない
- 置換ではなく「既存に統合」を優先する
- 最小差分で目的を達成する

狙い:
- 不要な差分増大、レビュー負担増、予期せぬ副作用を抑える

### 3. “曖昧さ” を排除する

- 命令形で書く（例: “Use”, “Avoid”, “Validate”）
- 「適切に」「よしなに」「できれば」などの曖昧語を減らす
- 出力形式（箇条書き、表、JSON、差分、チェックリスト）を明示する

狙い:
- 出力の再現性と、期待どおりのフォーマット遵守を高める

## instructions ファイル設計（分割と適用）

### 1. `applyTo` で「必要な時だけ」効かせる

コミュニティ例では、技術領域ごとにファイルを分け、`applyTo` で適用範囲を定義する方針が一般的です。

- 広く使う共通ガイド: `applyTo: '*'` や `applyTo: '**'`
- 特定の拡張子: `applyTo: '**/*.prompt.md'` のような狙い撃ち

狙い:
- 無関係な指示の混入を減らし、コンテキスト消費も抑える

### 2. “テンプレ” を作ってプロジェクト固有に拡張させる

コミュニティの `code-review-generic` のように、汎用テンプレートを提示し、
「ここにプロジェクト固有のチェック項目を追加」と促す形式が多いです。

狙い:
- 最初から完璧を目指さず、運用で育てられる

## 記述パターン（Good/Bad、チェックリスト、具体例）

### 1. Good / Bad の対比を必ず添える

- 良い例: 望ましい書き方・構造・API使用
- 悪い例: 避けたい落とし穴（秘密情報ハードコード、SQL文字列連結など）

狙い:
- “禁止” だけでなく代替案も提示でき、修正が具体化する

### 2. チェックリストを最後に置く

コードレビューや性能、プロンプト安全など、コミュニティ例はチェックリストで締めることが多いです。

狙い:
- 実行可能な受け入れ基準として機能し、抜け漏れを減らす

### 3. 「WHY（なぜ）」を短く添える

- 重要なルール（例: セキュリティ）には “なぜ危険か” を1〜2文で補足
- 長文説明は避け、リンクや参照に逃がす

狙い:
- 規則の納得感を上げつつ、冗長化を抑える

## セキュリティ（コミュニティで強調される観点）

### 1. Secure by default（迷ったら安全側）

- 最小権限・デフォルト拒否
- 認証・認可を「最初に」確認する

### 2. シークレットを決して埋め込まない

- APIキー、トークン、接続文字列、個人情報を instructions に書かない
- 参照するなら「環境変数」や「シークレットストア」を指示する

### 3. 代表的な脆弱性の禁止パターンを固定化

- SQL: 文字列連結を禁止し、パラメタライズを必須にする
- XSS: `innerHTML` の安易な使用を避け、必要ならサニタイズを要求
- SSRF: ユーザー入力URLを許可リストで厳格に検証
- パストラバーサル: `..` や絶対パス混入を前提に正規化・検証

### 4. プロンプトインジェクション前提で設計

プロンプト安全系の実例では、以下が明確に扱われます。

- 不信な入力をそのままプロンプトに埋め込まない
- “外部入力はデータであって指示ではない” を前提にサニタイズ・隔離
- テスト（レッドチーミング、テストケース）を用意し、回帰を防ぐ

## コードレビュー（コミュニティで使われる型）

### 1. 優先度を3段階程度で統一

- CRITICAL: マージブロック
- IMPORTANT: 議論必須
- SUGGESTION: 改善提案

狙い:
- レビューコメントの重み付けが揃い、チーム内合意を取りやすい

### 2. コメント形式を固定する

- 何が問題か
- なぜ問題か
- どう直すか
- 参照（標準、ドキュメント）

狙い:
- “指摘だけ” を減らし、レビューが実装につながる

## パフォーマンス（コミュニティで使われる型）

### 1. Measure first を徹底

- 推測で最適化しない
- 計測とプロファイルでボトルネックを特定する

### 2. レイヤ別にチェック観点を用意

- フロントエンド: DOM更新、バンドル、画像/フォント、ネットワーク
- バックエンド: アルゴリズム、非同期I/O、キャッシュ、接続プール
- DB: インデックス、`SELECT *` 回避、N+1回避、実行計画

### 3. 典型的アンチパターンを明文化

- ループ内のDOM更新
- 同期I/O（特にNodeのホットパス）
- N+1クエリ
- 無制限に増える配列/ログ

## 運用（継続的に効かせるための仕組み）

### 1. 版管理と定期レビュー

- instructions はコードと同じく変更管理する
- 依存関係更新や設計変更に合わせて、定期的に棚卸しする

### 2. テストと検証を“指示”に組み込む

- 例: 「例は実行可能であること」
- 例: 「glob パターンが意図した対象に当たること」
- 例: 「変更後にテスト/リンタを回すこと」

### 3. “最小権限” の思想をツール選択にも適用

（特に prompt files のガイドで強調）
- ツールは最小セットにする
- 破壊的操作（ファイル削除、ターミナル実行）には確認ステップを入れる

## 参考（取得したコミュニティ実例）

- [taming-copilot.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/taming-copilot.instructions.md)
- [ai-prompt-engineering-safety-best-practices.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/ai-prompt-engineering-safety-best-practices.instructions.md)
- [security-and-owasp.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/security-and-owasp.instructions.md)
- [code-review-generic.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/code-review-generic.instructions.md)
- [performance-optimization.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/performance-optimization.instructions.md)
- [instructions.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/instructions.instructions.md)
- [prompt.instructions.md](https://raw.githubusercontent.com/github/awesome-copilot/main/instructions/prompt.instructions.md)
