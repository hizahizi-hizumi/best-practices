# agent_skills ドキュメント URL

## ベストプラクティス作成に参考にするべきドキュメント

### 公式サイト（Overview / Concepts）
- [Overview](https://agentskills.io/home) - Agent Skillsの目的・前提・導入の入口
- [Get started（セクション）](https://agentskills.io/home#get-started) - 公式の推奨導線（概念/仕様/統合）
- [Open development（セクション）](https://agentskills.io/home#open-development) - 標準の背景と運用方針（互換性を意識した実装に重要）
- [What are skills?](https://agentskills.io/what-are-skills) - スキルの概念、何をどこに置くべきかの理解
- [Skill authoring best practices（Anthropic）](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) - スキル作成の実践ガイド（簡潔さ、分割、反復、アンチパターン）

### 仕様（SKILL.md / ディレクトリ設計）
- [Specification](https://agentskills.io/specification) - フォーマットの一次情報（全体構造）
- [SKILL.md frontmatter](https://agentskills.io/specification#frontmatter-required) - `name` / `description` 等の必須・任意フィールド
- [`name` field](https://agentskills.io/specification#name-field) - 命名規則（ディレクトリ名一致、文字種、長さ、禁止事項）
- [`description` field](https://agentskills.io/specification#description-field) - discovery精度を左右する説明の書き方（キーワード戦略）
- [`compatibility` field](https://agentskills.io/specification#compatibility-field) - 実行環境要件の宣言指針（必要時のみ）
- [`metadata` field](https://agentskills.io/specification#metadata-field) - 拡張メタ情報の付け方（衝突回避の推奨）
- [`allowed-tools` field](https://agentskills.io/specification#allowed-tools-field) - 事前許可ツールの宣言（安全・権限境界）
- [Optional directories](https://agentskills.io/specification#optional-directories) - `scripts/` / `references/` / `assets/` の役割
- [Progressive disclosure](https://agentskills.io/specification#progressive-disclosure) - コンテキスト最適化（読み込み段階、SKILL.md 500行目安）
- [File references](https://agentskills.io/specification#file-references) - 相対パス参照とネスト深さ制限（1階層推奨）
- [Validation](https://agentskills.io/specification#validation) - `skills-ref validate` の公式推奨

### 統合（エージェント側の実装）
- [Integrate skills into your agent](https://agentskills.io/integrate-skills) - discovery→activation→execution の全体像
- [Injecting into context](https://agentskills.io/integrate-skills#injecting-into-context) - system promptへのメタデータ注入例（XML）
- [Security considerations](https://agentskills.io/integrate-skills#security-considerations) - スクリプト実行時の安全対策（sandbox/allowlist/確認/監査）
- [Reference implementation](https://agentskills.io/integrate-skills#reference-implementation) - 公式リファレンス実装へのリンク

### 公式リポジトリ（実装・実例）
- [Standard repo（agentskills/agentskills）](https://github.com/agentskills/agentskills) - 仕様・運用の公式ソース
- [skills-ref（reference library）](https://github.com/agentskills/agentskills/tree/main/skills-ref) - validate / prompt生成の参照実装
- [Example skills（anthropics/skills）](https://github.com/anthropics/skills) - 実例（構成・粒度・分割の判断材料）
