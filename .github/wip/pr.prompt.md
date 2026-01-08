---
description: '差分を分析して PR 粒度ルールに沿う PR 設計と PR 本文を作成'
agent: 'agent'
tools: ['read', 'runCommands']
---

# PR 粒度の設計と PR 本文作成（PR Helper）

## 目的
現在の作業ブランチの差分を分析し、`.github/instructions/pr.instructions.md` のルールに沿って **適切な PR 粒度**（必要なら分割案）と、レビューしやすい **PR タイトル/本文**を作成する。

## 設定変数
${input:baseBranch:main} : 比較対象のベースブランチ（例: main / master / develop）
${input:scopeHint:} : 変更の目的・背景（例: 「検索条件の追加」「null 対応」「CI 修正」）

---

## ワークフロー

### フェーズ 0: ルールの読み込み（必須）
1. #tool:read/readFile `.github/instructions/pr.instructions.md` を読み、以降の判断基準として適用する。

### フェーズ 1: 発見（差分の把握）
次の情報を収集し、要点を短く要約する。

- ブランチ情報
  - `git rev-parse --abbrev-ref HEAD`
  - `git status --porcelain=v1 -uall`
- ベースとの差分（PR 対象の差分を把握）
  - `git fetch --all --prune`（必要な場合のみ）
  - `git log --oneline ${baseBranch}..HEAD`
  - `git diff --name-status ${baseBranch}..HEAD`
  - `git diff --stat ${baseBranch}..HEAD`
  - `git diff ${baseBranch}..HEAD`（必要に応じて）
- 既存の PR テンプレート・規約の確認（存在する場合）
  - 例: `.github/pull_request_template.md` や `.github/PULL_REQUEST_TEMPLATE/*` を確認する
- CI/テストコマンドの推定（既存の README / package scripts / Makefile 等があれば優先）

### フェーズ 2: 粒度判定（分割が必要か）
`.github/instructions/pr.instructions.md` のルールに基づき、差分を分類する。

- 1 つの目的に収まっているか（single purpose）
- 機械的変更（整形・リネーム・移動）と意味のある変更が混ざっていないか
- リファクタと挙動変更が混ざっていないか
- 依存更新・CI・開発環境とアプリ挙動が混ざっていないか
- サイズがレビュー可能か（大きすぎる場合は分割案を作る）

分割が必要なら、以下のいずれかで「小さい PR の列」に設計する。
- 垂直スライス（小さく使える単位で機能を完成）
- 段階導入（機能フラグ、互換レイヤ、移行の段階化）
- スタック（依存関係を明示した複数 PR：チームが許容する場合）

### フェーズ 3: PR 設計の提示（必須）
- PR が 1 つで適切か、複数に分割すべきかを結論として述べる。
- 分割する場合は、提案 PR を番号付きで列挙し、それぞれに以下を付ける。
  - PR タイトル（候補）
  - 目的（1 文）
  - 対象ファイル群（パスのまとまり）
  - 依存関係（先に入れる PR / 後続 PR）
  - リスクとロールバック観点

### フェーズ 4: PR 本文の生成（必須）
最終的に「いま出す PR」について、Markdown の PR 本文を生成する。

- タイトル: 1 行で具体的に
- 本文: 既存テンプレートがあればそれに合わせる。無ければ以下を使う。
  - 概要（What）
  - 背景（Why）
  - 変更点（How / 実装方針）
  - テスト（How to test）
  - 影響範囲・リスク
  - ロールアウト / 移行（必要な場合）
  - 関連 Issue / PR
- サイズが大きい場合は、レビューの推奨順序や「なぜ分割できないか」を本文に含める。

### フェーズ 5: 検証（任意だが推奨）
存在する場合のみ、プロジェクトの慣習に合わせて検証コマンドを提案・実行する。
- 例: `bundle exec rspec`, `rails test`
- 例: `npm test`, `pnpm test`, `yarn test`
- 例: `npm run lint`, `pnpm run lint`, `yarn lint`

---

## 制約・禁止事項
- 無関係な変更を同一 PR に混ぜない。
- ロジック変更とフォーマットのみ変更を同一 PR に混ぜない。
- 依存更新と機能開発を同一 PR に束ねない（前提なら理由を本文に書く）。
- 巨大 PR を正当化せず、分割・段階導入・スタック案を先に検討する。
- PR 作成（push / PR 作成コマンド / マージ）などの破壊的操作は、明示的な依頼がない限り実行しない。

---

## 出力
- 差分の要約（変更ファイル、主な変更点、混在の有無）
- PR 粒度の結論（単一 PR or 分割案）
- 「いま出す PR」のタイトル案と PR 本文（Markdown）
- （必要なら）次の PR / 後続 PR の計画
