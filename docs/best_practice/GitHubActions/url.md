## セキュリティ（最小権限・サプライチェーン・認証）
- https://docs.github.com/ja/actions/reference/security/secure-use — ワークフロー/サードパーティアクションの安全な利用（pinning、最小権限、マスク、ログ、PR作成制御など）の要点。
- https://docs.github.com/ja/actions/tutorials/authenticate-with-github_token — `GITHUB_TOKEN` の使い方と `permissions` での最小権限設定。
- https://docs.github.com/ja/actions/concepts/security/openid-connect — OIDCで長期シークレットを避けるための概念整理（クラウド連携の基本）。
- https://docs.github.com/ja/actions/reference/security/oidc — OIDCクレーム/権限（`id-token: write` 等）と取得・設計のリファレンス。
- https://docs.github.com/ja/actions/how-tos/secure-your-work/security-harden-deployments — デプロイをOIDC中心に安全化する実践ガイド（各プロバイダ設定への導線あり）。

## ワークフロー設計（構文・式・コンテキスト）
- https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-syntax — workflow.ymlの正確な構文リファレンス（`permissions`/`concurrency`/`strategy` 等もここ）。
- https://docs.github.com/ja/actions/reference/workflows-and-actions/expressions — `${{ }}` 式、関数、型変換、条件分岐の落とし穴を把握するための基礎。
- https://docs.github.com/ja/actions/reference/workflows-and-actions/contexts — `github`/`secrets`/`vars`/`matrix` などコンテキストの参照方法と可用範囲。
- https://docs.github.com/ja/actions/reference/workflows-and-actions/variables — 既定環境変数/varsの命名規則・優先順位・上限など、設定のベストプラクティスに直結。
- https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-commands — ログマスクや`GITHUB_STEP_SUMMARY`など、運用性・セキュリティに効くワークフローコマンド。

## ワークフロー設計（並行制御・マトリクス）
- https://docs.github.com/ja/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency — `concurrency`/`cancel-in-progress` で二重実行や競合を防ぐ。
- https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/run-job-variations — `strategy.matrix`/`fail-fast`/`max-parallel` でテスト/ビルドの最適化。

## 再利用（Reusable workflows / Composite actions）
- https://docs.github.com/ja/actions/how-tos/reuse-automations/reuse-workflows — 再利用可能ワークフローの呼び出し、入力、シークレット受け渡しの実装方法。
- https://docs.github.com/ja/actions/reference/workflows-and-actions/reusing-workflow-configurations — YAMLアンカー等で重複を減らす（DRY）ための公式整理。
- https://docs.github.com/ja/actions/tutorials/create-actions/create-a-composite-action — 複合アクションでステップを部品化し、標準化・再利用を進める。

## キャッシュと成果物（速度・コスト・再現性）
- https://docs.github.com/ja/actions/reference/workflows-and-actions/dependency-caching — キャッシュキー設計、制限、退避（eviction）など「効くキャッシュ」設計の要点。
- https://docs.github.com/ja/actions/concepts/workflows-and-actions/workflow-artifacts — アーティファクトの保存/共有/保持期間の考え方（キャッシュとの違い含む）。

## Secrets / Environments（デプロイ保護と取り扱い）
- https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets — シークレットの利用時の制約（fork/Dependabot等）と安全な渡し方。
- https://docs.github.com/ja/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments — 環境の作成・保護ルール・環境シークレット/変数の管理。
- https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/deploy-to-environment — workflowから`environment`を使う方法（承認ゲート等を組み込む前提）。

## ランナー（self-hosted / larger runners）
- https://docs.github.com/ja/actions/how-tos/manage-runners/self-hosted-runners — セルフホステッドランナーの導入・運用の入口（責務とリスク含む）。
- https://docs.github.com/ja/actions/how-tos/manage-runners/self-hosted-runners/manage-access — ランナーグループ等でアクセス制御し、意図しない実行を防ぐ。
- https://docs.github.com/ja/actions/concepts/runners/larger-runners — より大きなランナーの概念（性能/機能/課金）と利用判断の基礎。

## ガバナンスと継続的改善（設定・Dependabot）
- https://docs.github.com/ja/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository — Actionsの許可/制限、`GITHUB_TOKEN`既定権限、成果物・ログ保持などの管理。
- https://docs.github.com/ja/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot — `github-actions`エコシステムでActions参照（タグ/sha）を自動更新し、脆弱性/陳腐化を減らす。
