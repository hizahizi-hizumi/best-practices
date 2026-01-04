# Custom Instructions 関連ドキュメントURL

このドキュメントは、VS Code の Custom Instructions 機能および関連するカスタマイゼーション機能のベストプラクティスガイド作成に役立つ公式ドキュメントのURLをまとめたものです。

## 公式ドキュメント - Core

### [Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
カスタムインストラクションの基本的な使い方を解説。`.github/copilot-instructions.md`、`.instructions.md`、`AGENTS.md`の3つのファイルタイプの使い分け、YAML frontmatter の設定方法、Markdown リンクによるファイル参照、自動適用とマニュアル適用の切り替え、ワークスペース単位とユーザー単位の使い分けなどを網羅。

### [Customization Overview](https://code.visualstudio.com/docs/copilot/customization/overview)
VS Code における AI カスタマイゼーションの全体像を解説。Custom Instructions、Agent Skills、Prompt Files、Custom Agents、Language Models、MCP Servers の6つの主要なカスタマイゼーション手法を紹介し、それぞれのユースケースと使い分けを説明。段階的な導入方法も提示。

### [Prompt Files](https://code.visualstudio.com/docs/copilot/customization/prompt-files)
再利用可能なプロンプトファイルの作成方法を解説。`.prompt.md` 拡張子を使用し、YAML frontmatter でエージェント、ツール、モデルを指定。変数参照（`${workspaceFolder}`、`${selection}` など）や入力変数の使用方法、ワークスペースとユーザープロファイルでのスコープ管理について説明。

### [Custom Agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents)
特定の役割やタスクに特化したカスタムエージェントの作成方法を解説。`.agent.md` ファイルを使用し、利用可能なツールの指定、専門的なインストラクションの定義、handoff 機能による段階的なワークフローの構築方法を説明。Planning、Implementation、Code Review などの役割別エージェントの例も紹介。

### [Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
Agent Skills（オープンスタンダード）の作成と使用方法を解説。`.github/skills/` ディレクトリに `SKILL.md` ファイルを配置し、インストラクション、スクリプト、リソースをまとめて管理。Custom Instructions との違いや、Progressive Disclosure による効率的なコンテキスト管理について説明。

## 公式ドキュメント - 関連機能

### [Language Models](https://code.visualstudio.com/docs/copilot/customization/language-models)
AI 言語モデルの選択と管理方法を解説。モデルピッカーの使用方法、Auto Model Selection、Bring Your Own Key (BYOK) による外部モデルの追加、ローカルモデルの使用方法などを説明。タスクに応じた最適なモデルの選択方法も紹介。

### [MCP Servers](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)
Model Context Protocol (MCP) サーバーの設定と使用方法を解説。`mcp.json` による設定方法、stdio/HTTP/SSE サーバーの接続方法、ツールとリソースの使用方法、セキュリティとトラスト境界の管理について説明。GitHub MCP server registry からのインストール方法も紹介。

### [Chat Tools](https://code.visualstudio.com/docs/copilot/chat/chat-tools)
チャットで使用できるツールの種類と管理方法を解説。Built-in Tools、MCP Tools、Extension Tools の3つのタイプ、ツールの有効化と無効化、ツール承認の管理、URL 承認の2段階プロセス、ターミナルコマンドの自動承認設定などを説明。

### [Security](https://code.visualstudio.com/docs/copilot/security)
AI 機能使用時のセキュリティ考慮事項を解説。Trust Boundaries（Workspace Trust、Extension Publisher Trust、MCP Server Trust）、自動承認機能のリスク、プロンプトインジェクション攻撃への対策、ユーザーの責任とベストプラクティスについて説明。

## 公式ドキュメント - Getting Started

### [Copilot Chat](https://code.visualstudio.com/docs/copilot/chat/copilot-chat)
VS Code におけるチャット機能の基本的な使い方を解説。Chat View、Inline Chat、Quick Chat の3つのアクセス方法、エージェントの切り替え（Agent、Plan、Ask、Edit）、コンテキストの追加方法、効果的なプロンプトの書き方などを説明。

### [Copilot Setup](https://code.visualstudio.com/docs/copilot/setup)
GitHub Copilot の初期設定方法を解説。GitHub アカウントでのサインイン、GitHub Enterprise (GHE) アカウントの使用、ワークスペースやプロファイルごとの異なるアカウント使用、AI 機能の無効化方法などを説明。

### [Copilot Cheat Sheet](https://code.visualstudio.com/docs/copilot/reference/copilot-vscode-features)
GitHub Copilot の機能とキーボードショートカットのチートシート。チャット体験、コンテキストの追加、ツール、スラッシュコマンド、チャット参加者、エージェント、カスタマイゼーション、エディター機能、ソース管理、テスト生成、デバッグなど、幅広い機能を網羅。

## コミュニティリソース

### [Awesome Copilot Repository](https://github.com/github/awesome-copilot/tree/main)
コミュニティによる Custom Instructions、Prompt Files、Custom Agents の実例集。様々なプログラミング言語、フレームワーク、開発タスクに対応した実用的な例が豊富に含まれており、ベストプラクティスの参考になる。

### [Agent Skills Standard](https://agentskills.io/)
Agent Skills のオープンスタンダード仕様。VS Code、GitHub Copilot CLI、GitHub Copilot Coding Agent など複数の AI エージェント間で再利用可能なスキルの作成方法を定義。

### [Anthropic Skills Repository](https://github.com/anthropics/skills)
Anthropic による Agent Skills のリファレンス実装集。様々なタスクに対応したスキルの実例が含まれており、自分のスキルを作成する際の参考になる。

## 関連ドキュメント

### [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
Model Context Protocol (MCP) の公式仕様ドキュメント。MCP サーバーの開発方法、プロトコルの詳細、セキュリティ考慮事項などが説明されている。

### [GitHub MCP Server Registry](https://github.com/mcp)
GitHub による公式 MCP サーバーレジストリ。VS Code から直接インストール可能な MCP サーバーが登録されており、様々な外部サービスとの連携が可能。

### [MCP Server Repository](https://github.com/modelcontextprotocol/servers)
Model Context Protocol の公式サーバーリポジトリ。ファイルシステム操作、データベース連携、Web サービスなど、様々な機能を提供する公式および コミュニティ提供のサーバーが含まれる。

## GitHub Copilot 公式ドキュメント

### [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
GitHub Copilot の公式ドキュメント。VS Code 以外のエディターでの使用方法、GitHub Copilot CLI、GitHub Codespaces での使用方法、エンタープライズ向けの管理機能などを説明。

### [Available AI Models](https://docs.github.com/en/copilot/using-github-copilot/ai-models/changing-the-ai-model-for-copilot-chat?tool=vscode)
GitHub Copilot で使用可能な AI モデルの一覧と特徴。各モデルの得意分野、コンテキストサイズ、料金情報などが記載されている。

### [Choosing the Right AI Model](https://docs.github.com/en/copilot/using-github-copilot/ai-models/choosing-the-right-ai-model-for-your-task)
タスクに応じた最適な AI モデルの選び方を解説。速度重視のタスク、複雑な推論が必要なタスク、ビジョン機能が必要なタスクなど、それぞれに適したモデルを紹介。

---

最終更新: 2026年1月4日
