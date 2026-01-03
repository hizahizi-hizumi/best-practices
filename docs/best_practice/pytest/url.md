# pytest 公式ドキュメント - 重要URL集

## ベストプラクティス・推奨事項

### [Good Integration Practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
pytestの統合に関するベストプラクティス。以下の重要トピックを網羅：
- テストディスカバリーの規約（命名規則、ファイル構造）
- テストレイアウト（src layoutの推奨、tests outside application code）
- インポートモードの選択（importlib推奨）
- tox、flake8-pytest-styleの使用
- strict modeの活用

### [Anatomy of a test](https://docs.pytest.org/en/stable/explanation/anatomy.html)
テストの基本構造（Arrange-Act-Assert-Cleanup）を解説。良いテスト設計の基礎を理解するために必須。

### [About fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html)
フィクスチャの概念と利点を解説：
- フィクスチャとは何か
- xUnit-styleのsetup/teardownとの比較・改善点
- フィクスチャエラーの扱い方
- テストデータの共有方法

## フィクスチャの使い方

### [How to use fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
フィクスチャの実践的な使い方を網羅した最重要ページ：
- フィクスチャのリクエスト方法
- フィクスチャの再利用性とスコープ（function, class, module, package, session）
- yield fixturesによるセットアップ/ティアダウン
- autofixturesの使い方
- パラメータ化されたフィクスチャ
- フィクスチャのファクトリーパターン
- フィクスチャのオーバーライド
- 複数のassertステートメントを安全に実行する方法

## テストの書き方

### [How to write and report assertions in tests](https://docs.pytest.org/en/stable/how-to/assert.html)
assertの書き方とベストプラクティス：
- assert文の使い方（pytestの詳細な出力）
- 近似値の比較（pytest.approx）
- 例外のテスト（pytest.raises）
- 例外グループのテスト（pytest.RaisesGroup）
- カスタムアサーションの説明
- アサーションの内省（introspection）の仕組み

### [How to parametrize fixtures and test functions](https://docs.pytest.org/en/stable/how-to/parametrize.html)
パラメータ化テストの実装方法：
- @pytest.mark.parametrizeの使用
- pytest_generate_testsフックによる動的パラメータ化
- フィクスチャのパラメータ化

### [How to mark test functions with attributes](https://docs.pytest.org/en/stable/how-to/mark.html)
マーカーの登録と使用方法：
- カスタムマーカーの登録
- strict_markersによる厳格なマーカー管理

### [How to use skip and xfail to deal with tests that cannot succeed](https://docs.pytest.org/en/stable/how-to/skipping.html)
テストのスキップと期待される失敗の処理：
- @pytest.mark.skipとpytest.skip()
- @pytest.mark.skipif（条件付きスキップ）
- @pytest.mark.xfail（期待される失敗）
- パラメータ化テストでのskip/xfailの使用

## テストの実行・設定

### [How to invoke pytest](https://docs.pytest.org/en/stable/how-to/usage.html)
pytestの起動方法とコマンドラインオプション。

### [Managing pytest's output](https://docs.pytest.org/en/stable/how-to/output.html)
テスト出力の管理とカスタマイズ。

### [How to manage logging](https://docs.pytest.org/en/stable/how-to/logging.html)
ログ出力の管理方法。

### [How to capture stdout/stderr output](https://docs.pytest.org/en/stable/how-to/capture-stdout-stderr.html)
標準出力/エラー出力のキャプチャ方法。

### [How to capture warnings](https://docs.pytest.org/en/stable/how-to/capture-warnings.html)
警告のキャプチャとフィルタリング。

## テストの構造化・整理

### [How to use temporary directories and files in tests](https://docs.pytest.org/en/stable/how-to/tmp_path.html)
一時ディレクトリ・ファイルの使用方法（tmp_path, tmpdir）。

### [How to monkeypatch/mock modules and environments](https://docs.pytest.org/en/stable/how-to/monkeypatch.html)
モジュールや環境変数のモック/パッチング方法。

### [How to re-run failed tests and maintain state between test runs](https://docs.pytest.org/en/stable/how-to/cache.html)
失敗したテストの再実行とテスト間の状態維持（--lf, --ff）。

### [How to handle test failures](https://docs.pytest.org/en/stable/how-to/failures.html)
テスト失敗の処理方法。

## 実践例・カスタマイズ

### [Examples and customization tricks](https://docs.pytest.org/en/stable/example/index.html)
様々なユースケースの実例集：
- [Basic patterns and examples](https://docs.pytest.org/en/stable/example/simple.html) - 基本パターンと実例
- [Parametrizing tests](https://docs.pytest.org/en/stable/example/parametrize.html) - パラメータ化の詳細例
- [Working with custom markers](https://docs.pytest.org/en/stable/example/markers.html) - カスタムマーカーの活用
- [Changing standard (Python) test discovery](https://docs.pytest.org/en/stable/example/pythoncollection.html) - テスト検出のカスタマイズ

### [Demo of Python failure reports with pytest](https://docs.pytest.org/en/stable/example/reportingdemo.html)
pytestの詳細なエラーレポートのデモ。アサーション失敗時の出力例を多数掲載。

## プラグイン・拡張

### [How to install and use plugins](https://docs.pytest.org/en/stable/how-to/plugins.html)
プラグインのインストールと使用方法。

### [Writing plugins](https://docs.pytest.org/en/stable/how-to/writing_plugins.html)
プラグインの作成方法。

### [Writing hook functions](https://docs.pytest.org/en/stable/how-to/writing_hook_functions.html)
フック関数の実装方法。

## その他の重要トピック

### [How to use subtests](https://docs.pytest.org/en/stable/how-to/subtests.html)
サブテストの使用方法（パラメータ化の代替）。

### [How to run doctests](https://docs.pytest.org/en/stable/how-to/doctest.html)
doctestsの実行方法。

### [How to use pytest with an existing test suite](https://docs.pytest.org/en/stable/how-to/existingtestsuite.html)
既存のテストスイートとの統合。

### [How to use unittest-based tests with pytest](https://docs.pytest.org/en/stable/how-to/unittest.html)
unittestベースのテストとの互換性。

### [How to implement xunit-style set-up](https://docs.pytest.org/en/stable/how-to/xunit_setup.html)
xunitスタイルのセットアップ実装。

### [pytest import mechanisms and sys.path/PYTHONPATH](https://docs.pytest.org/en/stable/explanation/pythonpath.html)
pytestのインポートメカニズムとパス管理。

### [Typing in pytest](https://docs.pytest.org/en/stable/explanation/types.html)
型ヒント・型チェックの使用方法。

### [CI Pipelines](https://docs.pytest.org/en/stable/explanation/ci.html)
CI/CDパイプラインでのpytest活用。

### [Flaky tests](https://docs.pytest.org/en/stable/explanation/flaky.html)
不安定なテスト（flaky tests）への対処方法。

## リファレンス

### [How-to guides](https://docs.pytest.org/en/stable/how-to/index.html)
全ハウツーガイドのインデックス。

### [Reference guides](https://docs.pytest.org/en/stable/reference/index.html)
API・設定・プラグインの完全リファレンス。

### [Explanation](https://docs.pytest.org/en/stable/explanation/index.html)
概念・仕組みの詳細説明。

## 最初に読むべきページ

1. [Get Started](https://docs.pytest.org/en/stable/getting-started.html) - 基本的な使い方
2. [Good Integration Practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html) - ベストプラクティス
3. [Anatomy of a test](https://docs.pytest.org/en/stable/explanation/anatomy.html) - テスト構造の理解
4. [How to use fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html) - フィクスチャの実践
5. [How to write and report assertions in tests](https://docs.pytest.org/en/stable/how-to/assert.html) - アサーションの書き方

## ベストプラクティス策定時の重点確認事項

- **テスト構造**: Arrange-Act-Assert-Cleanupパターン
- **命名規則**: test_*.py / *_test.py, test_*関数, Test*クラス
- **レイアウト**: src layoutの採用、tests outside application code
- **インポートモード**: importlib推奨（prepend/appendは非推奨）
- **フィクスチャ**: 適切なスコープ、yield fixtures、依存関係の明確化
- **パラメータ化**: 同じロジックの複数パターンテスト
- **マーカー**: skip, xfail, parametrizeの適切な使用
- **アサーション**: シンプルなassert文、詳細なエラーメッセージ
- **分離**: テスト間の独立性、状態の共有を避ける
- **可読性**: 明確なテスト名、適切なコメント
