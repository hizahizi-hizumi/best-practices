# Vitest ドキュメント URL

## ベストプラクティス作成に参考にするべきドキュメント

### 基本ガイド
- [Getting Started](https://vitest.dev/guide/) - Vitestの基本的な使い方とセットアップ
- [Features](https://vitest.dev/guide/features) - Vitestの主要機能の概要
- [CLI](https://vitest.dev/guide/cli) - コマンドラインインターフェースの使い方

### テストの書き方
- [Test API Reference](https://vitest.dev/api/) - test, describe, expect等のAPIリファレンス
- [Test Context](https://vitest.dev/guide/test-context) - テストコンテキストとfixtures
- [Test Filtering](https://vitest.dev/guide/filtering) - テストのフィルタリングと選択実行

### アサーション
- [Expect](https://vitest.dev/api/expect) - expect APIと各種matcher（toBe, toEqual等）
- [Extending Matchers](https://vitest.dev/guide/extending-matchers) - カスタムmatcherの作成方法

### モック機能
- [Mocking](https://vitest.dev/guide/mocking) - モック機能の概要とチートシート
- [Mocking Modules](https://vitest.dev/guide/mocking/modules) - モジュールのモック方法
- [Mocking Functions](https://vitest.dev/guide/mocking/functions) - 関数のモック方法
- [Mocking Classes](https://vitest.dev/guide/mocking/classes) - クラスのモック方法
- [Mocking Globals](https://vitest.dev/guide/mocking/globals) - グローバル変数のモック
- [Mocking Requests](https://vitest.dev/guide/mocking/requests) - HTTPリクエストのモック
- [Mocking Timers](https://vitest.dev/guide/mocking/timers) - タイマーのモック
- [Mocking Dates](https://vitest.dev/guide/mocking/dates) - 日付のモック
- [Vi Utility](https://vitest.dev/api/vi) - vi.mock, vi.spyOn, vi.fn等のユーティリティAPI

### 非同期テスト
- [Test API Reference - async](https://vitest.dev/api/) - 非同期テストの書き方（Promise, async/await対応）
- [Expect - resolves/rejects](https://vitest.dev/api/expect#resolves) - Promiseのアサーション

### スナップショットテスト
- [Snapshot](https://vitest.dev/guide/snapshot) - スナップショットテストの使い方
- [Expect - toMatchSnapshot](https://vitest.dev/api/expect#tomatchsnapshot) - スナップショットマッチャー

### カバレッジ
- [Coverage](https://vitest.dev/guide/coverage) - コードカバレッジの設定と使用方法
- [Config - Coverage](https://vitest.dev/config/#coverage) - カバレッジ設定オプション

### セットアップとTeardown
- [Test API - Setup and Teardown](https://vitest.dev/api/#setup-and-teardown) - beforeEach, afterEach, beforeAll, afterAll
- [Test Hooks](https://vitest.dev/api/#test-hooks) - onTestFinished, onTestFailed

### パフォーマンス最適化
- [Improving Performance](https://vitest.dev/guide/improving-performance) - テストパフォーマンスの向上方法
- [Profiling Test Performance](https://vitest.dev/guide/profiling-test-performance) - テストパフォーマンスのプロファイリング
- [Parallelism](https://vitest.dev/guide/parallelism) - 並列実行の設定

### 設定とベストプラクティス
- [Config Reference](https://vitest.dev/config/) - 設定オプションの完全リファレンス
- [Test Environment](https://vitest.dev/guide/environment) - テスト環境の設定（node, jsdom, happy-dom）
- [Test Projects](https://vitest.dev/guide/projects) - マルチプロジェクト設定

### その他の重要なトピック
- [In-Source Testing](https://vitest.dev/guide/in-source) - ソースコード内でのテスト
- [Testing Types](https://vitest.dev/guide/testing-types) - 型のテスト
- [Common Errors](https://vitest.dev/guide/common-errors) - よくあるエラーとその解決方法
- [Migration Guide](https://vitest.dev/guide/migration) - JestからVitestへの移行ガイド
- [Debugging](https://vitest.dev/guide/debugging) - デバッグ方法
- [IDE Integration](https://vitest.dev/guide/ide) - IDE統合（VS Code等）
- [Vitest UI](https://vitest.dev/guide/ui) - UIモードの使用方法
