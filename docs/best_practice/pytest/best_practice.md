# pytest ベストプラクティス

本ドキュメントは、pytestを使用した自動単体テスト作成における一般的なベストプラクティスをまとめたものです。

## 目次

1. [テストの基本構造](#テストの基本構造)
2. [テストディスカバリーと命名規則](#テストディスカバリーと命名規則)
3. [プロジェクト構造とレイアウト](#プロジェクト構造とレイアウト)
4. [フィクスチャの使用](#フィクスチャの使用)
5. [アサーションの書き方](#アサーションの書き方)
6. [テストのパラメータ化](#テストのパラメータ化)
7. [マーカーの使用](#マーカーの使用)
8. [テストのスキップとXFail](#テストのスキップとxfail)
9. [一時ファイルとディレクトリ](#一時ファイルとディレクトリ)
10. [モックとモンキーパッチ](#モックとモンキーパッチ)
11. [テストの独立性](#テストの独立性)
12. [テストの速度とパフォーマンス](#テストの速度とパフォーマンス)
13. [決定性のあるテスト](#決定性のあるテスト)
14. [エッジケースとバウンダリーテスト](#エッジケースとバウンダリーテスト)
15. [テストカバレッジの測定](#テストカバレッジの測定)
16. [CI/CD統合](#cicd統合)
17. [設定とベストプラクティス](#設定とベストプラクティス)

---

## テストの基本構造

### Arrange-Act-Assert-Cleanupパターン

pytestにおけるテストは、以下の4つの段階で構成されるべきです：

1. **Arrange（準備）**: テストに必要なすべてを準備する
2. **Act（実行）**: テスト対象の動作を実行する
3. **Assert（検証）**: 結果が期待通りであることを検証する
4. **Cleanup（後処理）**: テスト後の片付け

```python
def test_fruit_salad(fruit_bowl):
    # Arrange: フィクスチャが準備を担当

    # Act: 動作を実行
    fruit_salad = FruitSalad(*fruit_bowl)

    # Assert: 結果を検証
    assert all(fruit.cubed for fruit in fruit_salad.fruit)

    # Cleanup: フィクスチャのyield後やfinalizer で実行
```

**ベストプラクティス:**
- 各段階を明確に分離する
- Actは単一の状態変更アクションにする
- Cleanupはフィクスチャに任せる

---

## テストディスカバリーと命名規則

### ファイルとディレクトリの命名

pytestは以下の規約に従ってテストを自動検出します：

**ファイル名:**
- `test_*.py` または `*_test.py`

**関数名:**
- `test` で始まる関数（クラス外）
- `test` で始まるメソッド（`Test` で始まるクラス内）

**クラス名:**
- `Test` で始まるクラス（`__init__` メソッドなし）

```python
# ✓ 正しい例
# test_example.py または example_test.py

def test_function():
    pass

class TestClass:
    def test_method(self):
        pass

# ✗ 検出されない例
def check_function():  # testで始まっていない
    pass

class ExampleTest:  # Testで始まっていない
    pass
```

**ベストプラクティス:**
- 一貫した命名規則を使用する
- テストの内容を明確に示す名前を付ける
- テストモジュールは、テスト対象モジュールと同じ名前に `test_` を付ける

---

## プロジェクト構造とレイアウト

### 推奨: src レイアウト

pytestでは、アプリケーションコードと分離した `src` レイアウトが推奨されます：

```
pyproject.toml
src/
    mypkg/
        __init__.py
        app.py
        view.py
tests/
    test_app.py
    test_view.py
```

**利点:**
- インストール済みパッケージに対してテストを実行できる
- ローカルコードとインストール済みコードの混同を防ぐ
- パッケージングの問題を早期に発見できる

### import mode の設定

新しいプロジェクトでは `importlib` モードを推奨：

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = ["--import-mode=importlib"]
pythonpath = ["src"]
```

**ベストプラクティス:**
- `src` レイアウトを採用する
- editable install を使用: `pip install -e .`
- `importlib` import mode を使用する
- テストディレクトリは一意の名前を維持する

---

## フィクスチャの使用

### フィクスチャの基本

フィクスチャは、テストのセットアップとティアダウンを管理する強力な仕組みです。

```python
import pytest

@pytest.fixture
def database_connection():
    # セットアップ
    db = create_database_connection()
    yield db
    # ティアダウン
    db.close()

def test_query(database_connection):
    result = database_connection.query("SELECT 1")
    assert result == 1
```

### フィクスチャのスコープ

適切なスコープを選択してパフォーマンスを最適化：

- `function` (デフォルト): テスト関数ごと
- `class`: テストクラスごと
- `module`: モジュールごと
- `package`: パッケージごと
- `session`: テストセッション全体で1回

```python
@pytest.fixture(scope="session")
def expensive_resource():
    """セッション全体で1回だけ初期化"""
    resource = setup_expensive_resource()
    yield resource
    resource.cleanup()
```

### yield フィクスチャ（推奨）

```python
@pytest.fixture
def file_resource():
    # セットアップ
    file = open("test.txt", "w")
    yield file
    # ティアダウン（必ず実行される）
    file.close()
```

### autouse フィクスチャ

すべてのテストで自動的に使用されるフィクスチャ：

```python
@pytest.fixture(autouse=True)
def reset_database():
    """各テスト前にデータベースをリセット"""
    database.reset()
```

### フィクスチャのパラメータ化

```python
@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def database(request):
    db = create_database(request.param)
    yield db
    db.close()

def test_database_query(database):
    # 3種類のデータベースすべてでテストが実行される
    result = database.query("SELECT 1")
    assert result == 1
```

**ベストプラクティス:**
- yield フィクスチャを使用する
- 適切なスコープを選択してパフォーマンスを向上させる
- フィクスチャの依存関係を明確にする
- フィクスチャは再利用可能にする
- 1つのフィクスチャで1つの状態変更のみ行う
- `conftest.py` で共有フィクスチャを定義する

---

## アサーションの書き方

### シンプルな assert 文を使用

pytestは標準のPython `assert` 文を使用し、詳細なエラーメッセージを自動生成します：

```python
def test_example():
    expected = 4
    actual = 2 + 2
    assert actual == expected
```

### 近似値の比較

浮動小数点数の比較には `pytest.approx()` を使用：

```python
def test_floats():
    assert (0.1 + 0.2) == pytest.approx(0.3)

def test_arrays():
    import numpy as np
    a = np.array([1.0, 2.0, 3.0])
    b = np.array([0.9999, 2.0001, 3.0])
    assert a == pytest.approx(b)
```

### 例外のテスト

```python
def test_zero_division():
    with pytest.raises(ZeroDivisionError):
        1 / 0

def test_exception_message():
    with pytest.raises(ValueError, match=r".*invalid.*"):
        raise ValueError("invalid value")
```

### カスタムアサーションメッセージ

```python
def test_with_message():
    x = 5
    assert x % 2 == 0, f"Expected even number, got {x}"
```

**ベストプラクティス:**
- シンプルな `assert` 文を使用する
- 複雑な比較ロジックは避ける
- 浮動小数点数の比較には `pytest.approx()` を使用
- 例外テストには `pytest.raises()` を使用
- 必要に応じてカスタムメッセージを追加

---

## テストのパラメータ化

### 基本的なパラメータ化

同じテストロジックを複数の入力で実行：

```python
@pytest.mark.parametrize("input,expected", [
    ("3+5", 8),
    ("2+4", 6),
    ("6*9", 54),
])
def test_eval(input, expected):
    assert eval(input) == expected
```

### 複数パラメータの組み合わせ

```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_foo(x, y):
    # x=0/y=2, x=1/y=2, x=0/y=3, x=1/y=3 で実行される
    pass
```

### パラメータにマーカーを付ける

```python
@pytest.mark.parametrize("input,expected", [
    ("3+5", 8),
    pytest.param("6*9", 42, marks=pytest.mark.xfail),
])
def test_eval(input, expected):
    assert eval(input) == expected
```

### フィクスチャのパラメータ化

```python
@pytest.fixture(params=["chrome", "firefox", "safari"])
def browser(request):
    driver = setup_browser(request.param)
    yield driver
    driver.quit()
```

**ベストプラクティス:**
- 同じロジックの異なる入力には必ずパラメータ化を使用
- パラメータには説明的な名前を付ける
- `ids` パラメータで読みやすいテストIDを指定
- パラメータ数が多い場合はフィクスチャのパラメータ化を検討

---

## マーカーの使用

### カスタムマーカーの登録

`pyproject.toml` でマーカーを登録：

```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "smoke: marks tests as smoke tests",
]
```

### マーカーの使用

```python
import pytest

@pytest.mark.slow
def test_heavy_computation():
    pass

@pytest.mark.integration
@pytest.mark.slow
def test_full_system():
    pass
```

### マーカーでテストを選択

```bash
# slowマーカーのテストのみ実行
pytest -m slow

# slowマーカーを除外
pytest -m "not slow"

# 複数のマーカー
pytest -m "smoke or integration"
```

### クラスやモジュールへのマーカー適用

```python
# クラス全体にマーカー
@pytest.mark.integration
class TestIntegration:
    def test_one(self):
        pass
    def test_two(self):
        pass

# モジュール全体にマーカー
pytestmark = pytest.mark.integration
```

**ベストプラクティス:**
- すべてのカスタムマーカーを登録する
- `strict_markers = true` を設定して未登録マーカーを検出
- マーカーには説明的な名前を付ける
- マーカーを使ってテストを分類（unit, integration, smoke など）

---

## テストのスキップとXFail

### テストのスキップ

条件に基づいてテストをスキップ：

```python
import sys
import pytest

@pytest.mark.skip(reason="not implemented yet")
def test_future_feature():
    pass

@pytest.mark.skipif(sys.version_info < (3, 10), reason="requires python3.10+")
def test_new_syntax():
    pass

def test_conditional():
    if not valid_config():
        pytest.skip("unsupported configuration")
```

### モジュール全体のスキップ

```python
# test_module.py
import sys
import pytest

if not sys.platform.startswith("win"):
    pytest.skip("skipping windows-only tests", allow_module_level=True)
```

### XFail（期待される失敗）

```python
@pytest.mark.xfail(reason="known bug #123")
def test_known_issue():
    assert buggy_function() == expected_value

@pytest.mark.xfail(raises=RuntimeError)
def test_expected_exception():
    raise RuntimeError("expected")

# strict mode: XPASSをエラーにする
@pytest.mark.xfail(strict=True)
def test_strict_xfail():
    pass
```

### インポートが見つからない場合のスキップ

```python
docutils = pytest.importorskip("docutils", minversion="0.3")
```

**ベストプラクティス:**
- スキップには明確な理由を記載
- プラットフォーム固有のテストには `skipif` を使用
- 既知のバグには `xfail` を使用（issueトラッカーの番号を記載）
- `strict=True` で意図しないXPASSを検出
- 一時的なスキップは最小限に

---

## 一時ファイルとディレクトリ

### tmp_path フィクスチャ（推奨）

各テスト関数に一意の一時ディレクトリを提供：

```python
def test_create_file(tmp_path):
    # tmp_pathはpathlib.Pathオブジェクト
    d = tmp_path / "sub"
    d.mkdir()
    p = d / "hello.txt"
    p.write_text("content", encoding="utf-8")

    assert p.read_text(encoding="utf-8") == "content"
    assert len(list(tmp_path.iterdir())) == 1
```

### tmp_path_factory フィクスチャ

セッションスコープで一時ディレクトリを作成：

```python
@pytest.fixture(scope="session")
def image_file(tmp_path_factory):
    img = compute_expensive_image()
    fn = tmp_path_factory.mktemp("data") / "img.png"
    img.save(fn)
    return fn

def test_histogram(image_file):
    img = load_image(image_file)
    # テスト処理
```

**ベストプラクティス:**
- 一時ファイルには必ず `tmp_path` を使用
- レガシーな `tmpdir` は避ける（`tmp_path` を使う）
- セッション全体で共有するデータには `tmp_path_factory` を使用
- 一時ディレクトリは自動的にクリーンアップされる

---

## モックとモンキーパッチ

### monkeypatch フィクスチャ

外部依存をモックしてテストを分離：

```python
def test_function(monkeypatch):
    # 関数の置き換え
    def mock_return():
        return Path("/abc")
    monkeypatch.setattr(Path, "home", mock_return)

    # 環境変数の設定
    monkeypatch.setenv("USER", "test_user")

    # 環境変数の削除
    monkeypatch.delenv("API_KEY", raising=False)

    # 辞書アイテムの設定
    monkeypatch.setitem(config, "debug", True)
```

### モッククラスの作成

```python
import requests

class MockResponse:
    @staticmethod
    def json():
        return {"mock_key": "mock_response"}

def test_api_call(monkeypatch):
    def mock_get(*args, **kwargs):
        return MockResponse()

    monkeypatch.setattr(requests, "get", mock_get)
    result = api_function()
    assert result["mock_key"] == "mock_response"
```

### フィクスチャとして再利用

```python
@pytest.fixture
def mock_response(monkeypatch):
    def mock_get(*args, **kwargs):
        return MockResponse()
    monkeypatch.setattr(requests, "get", mock_get)

def test_with_fixture(mock_response):
    result = api_function()
    assert result["mock_key"] == "mock_response"
```

### context による限定的なパッチ

```python
def test_partial(monkeypatch):
    with monkeypatch.context() as m:
        m.setattr(module, "function", mock_function)
        # ここでのみパッチが有効
    # ここではパッチが解除されている
```

**ベストプラクティス:**
- 外部依存（API、データベース、ファイルシステム）をモックする
- `monkeypatch` を使ってテストを分離する
- モックは必要最小限に
- builtin関数のパッチは避ける（問題を引き起こす可能性）
- 複雑なモックはフィクスチャ化して再利用

---

## テストの独立性

### 原則

各テストは他のテストから完全に独立している必要があります。

### 避けるべきパターン

```python
# ✗ 悪い例：グローバル状態の共有
shared_list = []

def test_one():
    shared_list.append(1)
    assert len(shared_list) == 1

def test_two():
    # test_oneの後に実行されると失敗する
    assert len(shared_list) == 0
```

### 推奨パターン

```python
# ✓ 良い例：フィクスチャで独立した状態を提供
@pytest.fixture
def empty_list():
    return []

def test_one(empty_list):
    empty_list.append(1)
    assert len(empty_list) == 1

def test_two(empty_list):
    # 常に空のリストから開始
    assert len(empty_list) == 0
```

### 実行順序への依存を避ける

```python
# ✗ 悪い例：テストの実行順序に依存
def test_setup():
    setup_database()

def test_query():
    # test_setupが先に実行されることを期待している
    result = database.query()
```

```python
# ✓ 良い例：フィクスチャで依存関係を管理
@pytest.fixture
def setup_database():
    db = create_database()
    setup_database(db)
    yield db
    db.cleanup()

def test_query(setup_database):
    result = setup_database.query()
```

**ベストプラクティス:**
- 各テストは独立して実行可能にする
- テスト間で状態を共有しない
- フィクスチャで必要な状態を提供
- テストの実行順序に依存しない
- `pytest-random-order` プラグインで独立性を検証

---

## 設定とベストプラクティス

### pyproject.toml の設定例

```toml
[tool.pytest.ini_options]
# テストディレクトリ
testpaths = ["tests"]

# import mode
addopts = [
    "--import-mode=importlib",
    "--strict-markers",
    "--strict-config",
    "-ra",  # 詳細なテスト結果サマリー
]

# Python path
pythonpath = ["src"]

# カスタムマーカー
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]

# xfail の strict mode
xfail_strict = true

# 最小バージョン
minversion = "7.0"
```

### strict mode の使用

```toml
[tool.pytest.ini_options]
strict = true  # すべてのstrict オプションを有効化
```

または個別に設定：

```toml
[tool.pytest.ini_options]
strict_config = true
strict_markers = true
xfail_strict = true
```

### conftest.py の活用

共有フィクスチャとフック関数を `conftest.py` に配置：

```python
# tests/conftest.py
import pytest

@pytest.fixture
def common_fixture():
    return "shared data"

def pytest_configure(config):
    config.addinivalue_line(
        "markers", "custom: description of custom marker"
    )
```

### flake8-pytest-style の使用

pytestのコーディングスタイルをチェック：

```bash
pip install flake8-pytest-style
flake8 tests/
```

**ベストプラクティス:**
- `pyproject.toml` で設定を一元管理
- `strict = true` でエラーを早期発見
- `src` レイアウトと `importlib` モードを使用
- `conftest.py` で共通設定とフィクスチャを管理
- `flake8-pytest-style` でコード品質を維持
- バージョン管理にテスト設定をコミット

---

## テストのベストプラクティス総まとめ

### 構造とレイアウト
- ✓ `src` レイアウトを採用
- ✓ `importlib` import mode を使用
- ✓ テストとアプリケーションコードを分離

### フィクスチャ
- ✓ `yield` フィクスチャを使用
- ✓ 適切なスコープを選択
- ✓ フィクスチャは小さく、再利用可能に
- ✓ `conftest.py` で共有フィクスチャを定義

### テストの書き方
- ✓ Arrange-Act-Assert-Cleanup パターンに従う
- ✓ シンプルな `assert` 文を使用
- ✓ テストは独立性を保つ
- ✓ 意味のあるテスト名を付ける

### パラメータ化とマーカー
- ✓ 同じロジックの異なる入力にはパラメータ化を使用
- ✓ カスタムマーカーを登録して使用
- ✓ `strict_markers = true` で未登録マーカーを検出

### モックとテストデータ
- ✓ `monkeypatch` で外部依存をモック
- ✓ `tmp_path` で一時ファイルを管理
- ✓ 実際のデータベース・APIは避ける

### 設定と品質
- ✓ `strict = true` で厳格なチェック
- ✓ `pyproject.toml` で設定を管理
- ✓ CI/CDパイプラインでテストを自動実行
- ✓ カバレッジツールで網羅性を測定

---

## テストの速度とパフォーマンス

テストの実行速度は開発サイクルに直接影響します。遅いテストは頻繁に実行されなくなり、バグの早期発見が困難になります。

### テストを高速化する原則

#### 1. 外部依存を最小化

```python
# ✗ 悪い例：実際のAPIを呼び出す
def test_api_slow():
    response = requests.get("https://api.example.com/data")
    assert response.status_code == 200

# ✓ 良い例：モックを使用
def test_api_fast(monkeypatch):
    def mock_get(*args, **kwargs):
        return MockResponse(200)
    monkeypatch.setattr(requests, "get", mock_get)

    response = api_client.get_data()
    assert response.status_code == 200
```

#### 2. データベース操作のモック

```python
import pytest
from unittest.mock import MagicMock

# インメモリデータベースを使用
@pytest.fixture
def in_memory_db():
    from sqlalchemy import create_engine
    engine = create_engine("sqlite:///:memory:")
    # テーブル作成
    yield engine
    # 自動クリーンアップ

def test_database_query(in_memory_db):
    # 高速なインメモリDBでテスト
    result = db_query(in_memory_db)
    assert len(result) > 0
```

#### 3. 適切なフィクスチャスコープの選択

```python
# セットアップコストが高いリソースはセッションスコープ
@pytest.fixture(scope="session")
def expensive_setup():
    """一度だけ初期化"""
    resource = create_expensive_resource()
    yield resource
    resource.cleanup()

# 変更可能なデータは関数スコープ
@pytest.fixture(scope="function")
def mutable_data():
    """各テストで新しいインスタンス"""
    return {"count": 0}
```

### 並列実行

pytest-xdist を使用してテストを並列実行：

```bash
# インストール
pip install pytest-xdist

# 4つのプロセスで並列実行
pytest -n 4

# CPUコア数に応じて自動調整
pytest -n auto
```

### 遅いテストの識別

```bash
# 最も遅い10個のテストを表示
pytest --durations=10

# すべてのテストの実行時間を表示
pytest --durations=0
```

### slowマーカーで分類

```python
import pytest

@pytest.mark.slow
def test_heavy_computation():
    result = expensive_calculation()
    assert result == expected

# 通常のテストのみ実行
# pytest -m "not slow"

# slowテストのみ実行
# pytest -m slow
```

**ベストプラクティス:**
- ✓ テストは1秒以内に完了することを目指す
- ✓ 外部サービス（API、データベース）はモックする
- ✓ セットアップコストが高いリソースは適切なスコープで共有
- ✓ 並列実行で全体の実行時間を短縮
- ✓ 遅いテストは `@pytest.mark.slow` でマーク
- ✓ CI/CDでは遅いテストを別パイプラインで実行

---

## 決定性のあるテスト

決定性のあるテスト（Deterministic Tests）とは、同じ入力に対して常に同じ結果を返すテストです。非決定性のテストは「フレーキー（flaky）」と呼ばれ、信頼性を損ないます。

### ランダム値の扱い

#### 悪い例：真のランダム性

```python
import random

# ✗ 悪い例：実行ごとに結果が変わる
def test_random_bad():
    value = random.randint(1, 100)
    result = process(value)
    assert result > 0  # 入力に依存して失敗する可能性
```

#### 良い例：シードの固定

```python
import random
import pytest

@pytest.fixture(autouse=True)
def reset_random():
    """各テストでランダムシードをリセット"""
    random.seed(42)
    yield

def test_random_good():
    value = random.randint(1, 100)  # 常に同じ値
    result = process(value)
    assert result == expected_value
```

### タイムスタンプと時間依存のコード

```python
from datetime import datetime, timezone
import pytest

# ✗ 悪い例：現在時刻に依存
def test_timestamp_bad():
    now = datetime.now(timezone.utc)
    result = create_record(now)
    # 時刻が変わると失敗する可能性
    assert result.created_at == now

# ✓ 良い例：固定された時刻を使用
@pytest.fixture
def fixed_time():
    return datetime(2024, 1, 1, 12, 0, 0, tzinfo=timezone.utc)

def test_timestamp_good(fixed_time, monkeypatch):
    # datetimeをモック
    class MockDatetime:
        @classmethod
        def now(cls, tz=None):
            return fixed_time

    monkeypatch.setattr("myapp.datetime", MockDatetime)
    result = create_record()
    assert result.created_at == fixed_time
```

### freezegun による時間の固定

```python
import pytest
from freezegun import freeze_time
from datetime import datetime

@freeze_time("2024-01-01 12:00:00")
def test_with_frozen_time():
    # この中では時間が固定される
    now = datetime.now()
    assert now.year == 2024
    assert now.month == 1
    assert now.day == 1
```

### 非同期処理とタイムアウト

```python
import pytest
import asyncio

# ✗ 悪い例：不確定な待機時間
async def test_async_bad():
    await asyncio.sleep(0.1)  # タイミングに依存
    result = await fetch_data()
    assert result is not None

# ✓ 良い例：明示的な待機とタイムアウト
@pytest.mark.asyncio
async def test_async_good():
    result = await asyncio.wait_for(
        fetch_data(),
        timeout=5.0  # 明示的なタイムアウト
    )
    assert result is not None
```

### UUIDと一意識別子

```python
from uuid import UUID
import pytest

@pytest.fixture
def fixed_uuid(monkeypatch):
    """UUIDを固定値に"""
    fixed_id = UUID('12345678-1234-5678-1234-567812345678')
    monkeypatch.setattr('uuid.uuid4', lambda: fixed_id)
    return fixed_id

def test_with_fixed_uuid(fixed_uuid):
    result = create_entity()
    assert result.id == fixed_uuid
```

**ベストプラクティス:**
- ✓ ランダム値には固定シードを使用
- ✓ 時刻依存のコードは `freezegun` や monkeypatch でモック
- ✓ 外部環境変数は明示的に設定
- ✓ 非同期処理には適切なタイムアウトを設定
- ✓ テストの実行順序に依存しない
- ✓ `pytest-random-order` で順序の独立性を検証

---

## エッジケースとバウンダリーテスト

エッジケースは、入力値の境界や特殊な条件でバグが発生しやすい領域です。これらを適切にテストすることで、本番環境での予期しない動作を防ぎます。

### バウンダリー値のテスト

```python
import pytest

@pytest.mark.parametrize("value,expected", [
    # 境界値
    (0, "zero"),              # 最小値
    (1, "positive"),          # 最小値 + 1
    (-1, "negative"),         # 最小値 - 1
    (100, "positive"),        # 最大値
    (99, "positive"),         # 最大値 - 1
    (101, "error"),           # 最大値 + 1
])
def test_boundary_values(value, expected):
    result = classify_number(value)
    assert result == expected
```

### 空のコレクション

```python
def test_empty_list():
    """空のリストの処理"""
    result = process_items([])
    assert result == []
    assert len(result) == 0

def test_empty_string():
    """空文字列の処理"""
    result = format_text("")
    assert result == ""

def test_empty_dict():
    """空の辞書の処理"""
    result = transform_data({})
    assert result == {}
```

### None と NULL 値

```python
import pytest

def test_none_value():
    """Noneが渡された場合"""
    result = safe_process(None)
    assert result is None

def test_none_in_list():
    """リスト内のNone"""
    data = [1, None, 3, None, 5]
    result = filter_none(data)
    assert result == [1, 3, 5]

def test_all_none():
    """すべてがNone"""
    data = [None, None, None]
    result = filter_none(data)
    assert result == []
```

### 文字列のエッジケース

```python
@pytest.mark.parametrize("input_str,expected", [
    ("", ""),                           # 空文字列
    (" ", ""),                          # スペースのみ
    ("\n\t", ""),                      # 空白文字のみ
    ("a", "a"),                         # 1文字
    ("あ", "あ"),                       # マルチバイト文字
    ("emoji 😀", "emoji 😀"),          # 絵文字
    ("line1\nline2", "line1\nline2"),  # 改行
    ("a" * 1000, "a" * 1000),          # 非常に長い文字列
])
def test_string_edge_cases(input_str, expected):
    result = process_string(input_str)
    assert result == expected
```

### 数値のエッジケース

```python
import math
import pytest

@pytest.mark.parametrize("value", [
    0,                  # ゼロ
    -0,                 # マイナスゼロ
    1,                  # 正の最小
    -1,                 # 負の最大
    float('inf'),       # 無限大
    float('-inf'),      # 負の無限大
    float('nan'),       # NaN
    2**31 - 1,         # 32bit整数の最大値
    -(2**31),          # 32bit整数の最小値
])
def test_numeric_edge_cases(value):
    # 特殊な値の処理を確認
    if math.isnan(value):
        with pytest.raises(ValueError):
            process_number(value)
    elif math.isinf(value):
        with pytest.raises(ValueError):
            process_number(value)
    else:
        result = process_number(value)
        assert isinstance(result, (int, float))
```

### 型の境界

```python
def test_type_boundaries():
    """異なる型の境界値"""
    # False と 0
    assert process_bool(False) != process_int(0)

    # None と 0 と False
    assert process_value(None) != process_value(0)
    assert process_value(0) != process_value(False)

    # 空文字列と None
    assert process_str("") != process_value(None)
```

### 例外の発生条件

```python
import pytest

def test_division_edge_cases():
    """除算のエッジケース"""
    # ゼロ除算
    with pytest.raises(ZeroDivisionError):
        result = divide(10, 0)

    # 非常に小さい数での除算
    result = divide(1, 1e-100)
    assert result > 0

def test_index_boundaries():
    """インデックスのエッジケース"""
    data = [1, 2, 3]

    # 正常なインデックス
    assert get_item(data, 0) == 1
    assert get_item(data, 2) == 3

    # 境界外
    with pytest.raises(IndexError):
        get_item(data, 3)

    with pytest.raises(IndexError):
        get_item(data, -4)
```

**ベストプラクティス:**
- ✓ 最小値、最大値、境界値±1をテスト
- ✓ 空のコレクション（空リスト、空文字列、空辞書）をテスト
- ✓ None、NULL、undefined の処理を確認
- ✓ 特殊な数値（0、無限大、NaN）をテスト
- ✓ 非常に大きい/小さい値をテスト
- ✓ マルチバイト文字、絵文字、特殊文字をテスト
- ✓ 例外が期待される条件を明示的にテスト

---

## テストカバレッジの測定

テストカバレッジは、コードのどの部分がテストで実行されたかを測定する指標です。`pytest-cov` を使用して簡単に測定できます。

### pytest-cov のインストールと基本使用

```bash
# インストール
pip install pytest-cov

# カバレッジを測定して実行
pytest --cov=myapp tests/

# カバレッジレポートをHTML形式で出力
pytest --cov=myapp --cov-report=html tests/

# 特定のディレクトリのみを対象
pytest --cov=myapp/core --cov-report=term-missing tests/
```

### pyproject.toml での設定

```toml
[tool.pytest.ini_options]
addopts = [
    "--cov=src",
    "--cov-report=html",
    "--cov-report=term-missing",
    "--cov-fail-under=80",  # 80%未満で失敗
]

[tool.coverage.run]
source = ["src"]
omit = [
    "*/tests/*",
    "*/test_*.py",
    "*/__init__.py",
    "*/migrations/*",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "class .*\\(Protocol\\):",
    "@(abc\\.)?abstractmethod",
]
```

### カバレッジから除外する

```python
def critical_function():
    """重要な関数は必ずテスト"""
    return calculate_result()

def debug_only_function():  # pragma: no cover
    """デバッグ専用の関数はカバレッジから除外"""
    print("Debug information")
    return None

if __name__ == "__main__":  # pragma: no cover
    # スクリプトとして実行される部分は除外
    main()
```

### カバレッジレポートの読み方

```bash
# ターミナル出力例
Name                      Stmts   Miss  Cover   Missing
--------------------------------------------------------
src/myapp/__init__.py         2      0   100%
src/myapp/core.py            45      5    89%   23-25, 67, 89
src/myapp/utils.py           30      0   100%
--------------------------------------------------------
TOTAL                        77      5    94%
```

- **Stmts**: 文の総数
- **Miss**: カバーされていない文の数
- **Cover**: カバレッジ率
- **Missing**: カバーされていない行番号

### カバレッジの目標値

```python
# 良好なカバレッジの目安
# ✓ 80%以上: 良好
# ✓ 90%以上: 優秀
# ✓ 100%: 理想的だが必須ではない

# プロジェクトの重要度に応じて設定
# - 金融システム: 95%以上
# - 一般的なアプリケーション: 80%以上
# - プロトタイプ: 60%以上
```

### ブランチカバレッジ

```toml
[tool.coverage.run]
branch = true  # 分岐カバレッジを有効化
```

```python
def example_function(x):
    if x > 0:  # この分岐の両方をテストする必要がある
        return "positive"
    else:
        return "non-positive"

# 両方の分岐をカバーするテスト
def test_positive():
    assert example_function(1) == "positive"

def test_non_positive():
    assert example_function(0) == "non-positive"
    assert example_function(-1) == "non-positive"
```

### CI/CDでのカバレッジチェック

```yaml
# GitHub Actions の例
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -e .[test]
          pip install pytest-cov

      - name: Run tests with coverage
        run: |
          pytest --cov=src --cov-report=xml --cov-report=term

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          fail_ci_if_error: true
```

**ベストプラクティス:**
- ✓ 80%以上のカバレッジを目標にする
- ✓ カバレッジだけでなく、テストの質も重視
- ✓ `--cov-fail-under` で最低カバレッジを強制
- ✓ ブランチカバレッジを有効化
- ✓ CI/CDでカバレッジを自動測定
- ✓ カバレッジレポートをレビューに活用
- ✓ 100%を盲目的に追求しない（テストの質が重要）

---

## CI/CD統合

CI/CDパイプラインでpytestを自動実行することで、コードの品質を継続的に保証します。

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/pyproject.toml') }}
        restore-keys: |
          ${{ runner.os }}-pip-

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -e .[test]

    - name: Run tests
      run: |
        pytest -v --cov=src --cov-report=xml --cov-report=term

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      if: matrix.python-version == '3.11'
      with:
        file: ./coverage.xml
```

### 高速化されたCI設定

```yaml
# 並列実行とキャッシュを活用
name: Fast Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cache/pip
          .pytest_cache
        key: ${{ runner.os }}-pytest-${{ hashFiles('**/pyproject.toml') }}

    - name: Install dependencies
      run: |
        pip install -e .[test]
        pip install pytest-xdist

    - name: Run tests in parallel
      run: |
        pytest -n auto --dist loadgroup
```

### マトリックス戦略（複数環境でテスト）

```yaml
name: Matrix Tests

on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false  # 1つ失敗しても他を続行
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11']
        exclude:
          # Windows + Python 3.9 の組み合わせを除外
          - os: windows-latest
            python-version: '3.9'

    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    - name: Run tests
      run: pytest -v
```

### GitLab CI

```yaml
# .gitlab-ci.yml
image: python:3.11

stages:
  - test
  - deploy

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

cache:
  paths:
    - .cache/pip
    - .pytest_cache

before_script:
  - pip install -e .[test]

test:unit:
  stage: test
  script:
    - pytest tests/unit -v --cov=src
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

test:integration:
  stage: test
  script:
    - pytest tests/integration -v
  only:
    - merge_requests
    - main
```

### ステージ別のテスト実行

```yaml
# 速いテストと遅いテストを分離
name: Staged Tests

on: [push, pull_request]

jobs:
  # ステージ1: 高速なユニットテスト（常に実行）
  unit-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest tests/unit -m "not slow" -v

  # ステージ2: 統合テスト（mainブランチのみ）
  integration-tests:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    needs: unit-tests
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest tests/integration -v

  # ステージ3: 遅いテスト（夜間実行）
  slow-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest -m slow -v
```

### テスト失敗時の通知

```yaml
name: Tests with Notifications

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest -v

    - name: Notify on failure
      if: failure()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: 'Tests failed on ${{ github.ref }}'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### pre-commit フック

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: pytest-check
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
        args: ['-v', '--tb=short', '-m', 'not slow']
```

### tox での複数環境テスト

```ini
# tox.ini
[tox]
envlist = py39,py310,py311,py312,lint

[testenv]
deps =
    pytest
    pytest-cov
    pytest-xdist
commands =
    pytest -n auto --cov=src {posargs}

[testenv:lint]
deps =
    flake8
    black
    mypy
commands =
    flake8 src tests
    black --check src tests
    mypy src
```

**ベストプラクティス:**
- ✓ すべてのプッシュ/PRでテストを自動実行
- ✓ 複数のPythonバージョンでテスト
- ✓ 依存関係をキャッシュして高速化
- ✓ 並列実行で時間を短縮
- ✓ カバレッジレポートを自動生成・保存
- ✓ 遅いテストは別パイプラインで実行
- ✓ テスト失敗時に通知
- ✓ マージ前にテストの成功を必須条件に

---

## 参考リソース

- [pytest公式ドキュメント](https://docs.pytest.org/)
- [Good Integration Practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
- [How to use fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [Anatomy of a test](https://docs.pytest.org/en/stable/explanation/anatomy.html)
