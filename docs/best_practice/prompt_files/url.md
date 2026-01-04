# Prompt Files 関連ドキュメントURL

このドキュメントは、VS Code の Prompt Files（`.prompt.md`）機能および関連するカスタマイゼーション機能のベストプラクティスガイド作成に役立つ、公式ドキュメントのURLをまとめたものです。

## 公式ドキュメント - Core

### [Prompt Files](https://code.visualstudio.com/docs/copilot/customization/prompt-files)
`.prompt.md` の基本仕様（YAML frontmatter、変数参照、入力変数、エージェント/ツール/モデル指定、保存場所と探索場所など）の一次情報。

### [Customization Overview](https://code.visualstudio.com/docs/copilot/customization/overview)
Custom Instructions / Prompt Files / Agents / MCP など、カスタマイゼーション全体像と使い分けの整理。

### [Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
Prompt Files と併用する「恒常的な指示（品質基準、出力形式、禁止事項）」の設計・共有方法の一次情報。

### [Custom Agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents)
Prompt Files の `agent` 指定と関連する、エージェント（役割・ツール・モデル）の束ね方と再利用の考え方。

## 公式ドキュメント - Prompt/Context の実務

### [Chat sessions](https://code.visualstudio.com/docs/copilot/chat/chat-sessions)
会話から `/savePrompt` で `.prompt.md` を生成するなど、プロンプト資産化の運用に関連。

### [Tips and tricks](https://code.visualstudio.com/docs/copilot/copilot-tips-and-tricks)
プロンプト作成、コンテキスト付与、モデル選択、インデックスなどの実践的なヒント。

### [Workspace context](https://code.visualstudio.com/docs/copilot/reference/workspace-context)
@workspace / #codebase を含むコンテキスト収集の基本と、インデックスの挙動・期待値調整。

## 公式ドキュメント - ツール/拡張/セキュリティ

### [Chat Tools](https://code.visualstudio.com/docs/copilot/chat/chat-tools)
Prompt Files から利用する `tools` の設計・承認・URL承認など、運用と安全性に直結。

### [MCP Servers](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)
MCP ツールを使う Prompt Files を設計する際の、設定共有・トラスト・運用上の注意点。

### [Security](https://code.visualstudio.com/docs/copilot/security)
プロンプトインジェクションや信頼境界（Workspace Trust 等）など、Prompt Files共有時のリスク整理。

### [Copilot settings reference](https://code.visualstudio.com/docs/copilot/reference/copilot-settings)
Prompt Files / Instructions / Agents の探索場所や推奨表示など、設定レベルの確認に利用。

## 公式ドキュメント - 参照情報

### [Variables reference](https://code.visualstudio.com/docs/reference/variables-reference)
`${workspaceFolder}` 等の変数仕様（一般機能）。Prompt Files の変数利用時の前提として参照。

## GitHub Copilot 公式ドキュメント

### [GitHub Copilot Best Practices](https://docs.github.com/en/copilot/get-started/best-practices)
プロンプトの具体化、分割、検証など、汎用のプロンプト作法（Prompt Filesの文章設計に流用可能）。

### [Copilot Chat Cookbook](https://docs.github.com/en/copilot/tutorials/copilot-chat-cookbook)
よくあるタスクの“プロンプトの型”。Prompt Files のテンプレート設計の参考。

### [Model hosting](https://docs.github.com/en/copilot/reference/ai-models/model-hosting)
データ取り扱いと運用上の注意点（組織で Prompt Files を共有・運用する前提知識）。

### [Change chat model in VS Code](https://docs.github.com/en/copilot/how-tos/use-ai-models/change-the-chat-model?tool=vscode)
VS Code 上でのモデル切り替えの考え方・注意点（Prompt Files の `model` 指定方針の判断材料）。

## 外部参考（公式以外）

### [Prompt Files and Instructions Files Explained（Microsoft DevBlogs）](https://devblogs.microsoft.com/dotnet/prompt-files-and-instructions-files-explained/)
Prompt/Instructions の使い分け、および prompt 側に含めるとよい項目（許可/禁止アクション、メンテ情報、コマンド等）の整理。

### [Prompt examples for chat in VS Code](https://code.visualstudio.com/docs/copilot/chat/prompt-examples)
チャットでのプロンプト例と、効果的なプロンプト作成のヒント（Prompt Files 本文の書き方の参考）。

### [Questions about using prompts and instructions（GitHub Community Discussion）](https://github.com/orgs/community/discussions/181190)
Prompt は明示呼び出し時のみ適用、Instructions は常時適用という理解の補強（コミュニティ回答）。

### [Beginner's Guide to Prompt Files in VS Code（Aridane Martín）](https://aridanemartin.dev/blog/vscode-prompt-files-in-vscode/)
frontmatter の概観や具体例（コミュニティ記事）。

---

最終更新: 2026年1月4日
