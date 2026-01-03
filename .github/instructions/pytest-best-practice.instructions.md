---
description: 'pytest ベストプラクティスに従ったテストコード作成ガイドライン'
applyTo: '**/test_*.py, **/*_test.py, **/tests/**/*.py, **/conftest.py'
---

# pytest ベストプラクティス

pytest を使用した高品質な自動テストコードを作成するための包括的なガイドライン。

## プロジェクト構造とレイアウト

### src レイアウトを採用（推奨）

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
    conftest.py
```

### pyproject.toml の必須設定

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "--import-mode=importlib",
    "--strict-markers",
    "--strict-config",
    "-ra",
]
pythonpath = ["src"]
minversion = "7.0"
xfail_strict = true

markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

### 命名規則

**必須:**
- テストファイル: `test_*.py` または `*_test.py`
- テスト関数: `test_` で始める
- テストクラス: `Test` で始める（`__init__` メソッドなし）
- テストメソッド: `test_` で始める

**例:**
```python
# test_example.py
def test_function():
    pass

class TestExample:
    def test_method(self):
        pass
```

## テストの基本構造

### Arrange-Act-Assert-Cleanup パターン（必須）

テストは以下の4段階で構成する：

```python
def test_user_registration(database, email_service):
    # Arrange（準備）: テストに必要なデータとモックを準備
    user_data = {
        "name": "test_user",
        "email": "test@example.com"
    }

    # Act（実行）: テスト対象の関数を1回だけ実行
    result = register_user(user_data, database)

    # Assert（検証）: 結果を検証
    assert result.success is True
    assert database.get_user("test_user") is not None
    assert email_service.sent_count == 1

    # Cleanup（後処理）: フィクスチャが自動的に処理
```

**ルール:**
- Act セクションは単一の関数呼び出しまたは状態変更のみ
- Assert は複数可能だが、すべて同じ Act の結果を検証
- Cleanup はフィクスチャに委ねる

## フィクスチャの作成と使用

### yield フィクスチャ（推奨）

```python
import pytest

@pytest.fixture
def database_connection():
    # セットアップ
    conn = create_connection("test.db")
    conn.initialize()

    yield conn  # テストに提供

    # ティアダウン（必ず実行される）
    conn.close()
    conn.cleanup()

def test_query(database_connection):
    result = database_connection.query("SELECT 1")
    assert result == 1
```

### フィクスチャのスコープ選択

```python
# 関数スコープ（デフォルト）: 各テスト関数で新規作成
@pytest.fixture
def user():
    return User("test")

# クラススコープ: テストクラスで1回
@pytest.fixture(scope="class")
def shared_resource():
    return SharedResource()

# モジュールスコープ: モジュールで1回
@pytest.fixture(scope="module")
def database():
    db = create_database()
    yield db
    db.drop()

# セッションスコープ: テストセッション全体で1回
@pytest.fixture(scope="session")
def docker_container():
    container = start_docker()
    yield container
    container.stop()
```

**スコープ選択のガイドライン:**
- デフォルトは `function`（最も安全）
- 高コストなリソース（DB、Docker）には `session` または `module`
- 状態を持たないリソースのみ広いスコープを使用

### フィクスチャの依存関係

```python
@pytest.fixture
def database():
    return create_database()

@pytest.fixture
def user_repository(database):
    # database フィクスチャに依存
    return UserRepository(database)

@pytest.fixture
def user_service(user_repository):
    # user_repository フィクスチャに依存
    return UserService(user_repository)

def test_create_user(user_service):
    # 依存関係は自動的に解決される
    user = user_service.create("test")
    assert user.name == "test"
```

### autouse フィクスチャ

```python
@pytest.fixture(autouse=True)
def reset_state():
    """各テスト前に自動的に実行"""
    global_cache.clear()
    database.reset()

@pytest.fixture(autouse=True, scope="module")
def setup_logging():
    """モジュール開始時に自動実行"""
    logging.basicConfig(level=logging.DEBUG)
```

### フィクスチャのパラメータ化

```python
@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def database(request):
    """3種類のDBすべてでテストが実行される"""
    db = create_database(request.param)
    yield db
    db.close()

def test_database_operations(database):
    # sqlite, postgres, mysql それぞれでテストされる
    assert database.query("SELECT 1") == 1
```

### conftest.py での共有フィクスチャ

```python
# tests/conftest.py
import pytest

@pytest.fixture
def api_client():
    """すべてのテストで使用可能"""
    return APIClient(base_url="http://test")

@pytest.fixture
def authenticated_client(api_client):
    """認証済みクライアント"""
    api_client.login("test", "password")
    return api_client
```

## アサーションの書き方

### シンプルな assert 文（必須）

```python
def test_addition():
    result = 2 + 2
    assert result == 4
    assert result > 0
    assert result != 5
```

### 浮動小数点数の比較

```python
import pytest

def test_float_comparison():
    # ✓ 正しい
    assert (0.1 + 0.2) == pytest.approx(0.3)
    assert 1.0001 == pytest.approx(1.0, abs=0.001)

    # ✗ 避ける（浮動小数点誤差で失敗する可能性）
    assert (0.1 + 0.2) == 0.3
```

### コレクションの比較

```python
def test_collections():
    # リスト
    assert [1, 2, 3] == [1, 2, 3]

    # セット
    assert {1, 2, 3} == {3, 2, 1}

    # 辞書
    result = {"name": "test", "age": 30}
    assert result["name"] == "test"
    assert result == {"name": "test", "age": 30}

    # 部分一致
    assert "name" in result
    assert result.get("age") == 30
```

### 例外のテスト

```python
import pytest

def test_exception():
    # 例外が発生することを確認
    with pytest.raises(ValueError):
        int("invalid")

    # 例外メッセージの確認
    with pytest.raises(ValueError, match=r"invalid literal.*"):
        int("invalid")

    # 例外オブジェクトの詳細確認
    with pytest.raises(ValueError) as exc_info:
        raise ValueError("test error")
    assert "test error" in str(exc_info.value)
```

### 警告のテスト

```python
import warnings
import pytest

def test_warning():
    with pytest.warns(UserWarning, match="deprecated"):
        warnings.warn("deprecated feature", UserWarning)
```

### カスタムエラーメッセージ

```python
def test_with_message():
    x = 5
    assert x % 2 == 0, f"Expected even number, got {x}"

    result = complex_calculation()
    assert result > 0, f"Result must be positive: {result}"
```

## テストのパラメータ化

### 基本的なパラメータ化

```python
import pytest

@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 3),
    (3, 4),
])
def test_increment(input, expected):
    assert increment(input) == expected
```

### 複数のパラメータセット

```python
@pytest.mark.parametrize("input,expected", [
    ("3+5", 8),
    ("2+4", 6),
    ("6*9", 54),
    ("12/3", 4),
])
def test_calculator(input, expected):
    assert eval(input) == expected
```

### ids による読みやすいテストID

```python
@pytest.mark.parametrize("input,expected", [
    ({"name": "Alice"}, True),
    ({}, False),
    ({"age": 30}, False),
], ids=["valid-user", "empty-dict", "missing-name"])
def test_validate_user(input, expected):
    assert validate_user(input) == expected
```

### 複数デコレータでの直積

```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_combination(x, y):
    # 4つのテスト: (0,2), (0,3), (1,2), (1,3)
    assert x + y >= 2
```

### マーカー付きパラメータ

```python
@pytest.mark.parametrize("input,expected", [
    (10, 20),
    pytest.param(100, 200, marks=pytest.mark.slow),
    pytest.param(1000, 2000, marks=pytest.mark.xfail(reason="known issue")),
])
def test_process(input, expected):
    assert process(input) == expected
```

## マーカーの使用

### カスタムマーカーの定義と使用

```python
# pyproject.toml に登録必須
# markers = [
#     "slow: marks tests as slow",
#     "integration: marks tests as integration tests",
#     "unit: marks tests as unit tests",
# ]

import pytest

@pytest.mark.unit
def test_pure_function():
    assert add(1, 2) == 3

@pytest.mark.integration
def test_database_integration():
    db = connect_to_database()
    assert db.is_connected()

@pytest.mark.slow
@pytest.mark.integration
def test_full_workflow():
    # 複数のマーカーを付与可能
    result = execute_long_process()
    assert result.success
```

### クラスとモジュールへのマーカー

```python
# クラス全体にマーカー
@pytest.mark.integration
class TestDatabaseOperations:
    def test_insert(self):
        pass

    def test_update(self):
        pass

# モジュール全体にマーカー（ファイルの先頭に記述）
pytestmark = pytest.mark.integration

# または複数マーカー
pytestmark = [pytest.mark.integration, pytest.mark.slow]
```

### 実行コマンド

```bash
# unit マーカーのテストのみ実行
pytest -m unit

# slow 以外のテスト実行
pytest -m "not slow"

# integration または slow のテスト実行
pytest -m "integration or slow"

# integration かつ slow でないテスト実行
pytest -m "integration and not slow"
```

## テストのスキップと XFail

### 条件付きスキップ

```python
import sys
import pytest

@pytest.mark.skipif(sys.version_info < (3, 10), reason="requires python3.10+")
def test_new_feature():
    # Python 3.10以降でのみ実行
    pass

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_feature():
    pass

# 動的スキップ
def test_conditional():
    if not has_required_dependency():
        pytest.skip("required dependency not available")
    # テストコード
```

### XFail（期待される失敗）

```python
@pytest.mark.xfail(reason="known bug #123")
def test_known_issue():
    # 現在失敗するが、将来修正される予定
    assert buggy_function() == expected_value

@pytest.mark.xfail(raises=RuntimeError, reason="feature not implemented")
def test_future_feature():
    new_feature()  # NotImplementedError が発生する

# strict mode: 予期せず成功した場合エラーにする
@pytest.mark.xfail(strict=True, reason="must fail")
def test_must_fail():
    assert False
```

### importorskip

```python
# 依存パッケージがない場合スキップ
numpy = pytest.importorskip("numpy", minversion="1.20")
pandas = pytest.importorskip("pandas")

def test_with_numpy():
    arr = numpy.array([1, 2, 3])
    assert len(arr) == 3
```

## 一時ファイルとディレクトリ

### tmp_path フィクスチャ（推奨）

```python
from pathlib import Path

def test_write_file(tmp_path):
    # tmp_path は pathlib.Path オブジェクト
    file_path = tmp_path / "test.txt"
    file_path.write_text("test content", encoding="utf-8")

    assert file_path.read_text(encoding="utf-8") == "test content"
    assert file_path.exists()

def test_directory_structure(tmp_path):
    # ディレクトリ構造の作成
    sub_dir = tmp_path / "sub" / "nested"
    sub_dir.mkdir(parents=True)

    file1 = sub_dir / "file1.txt"
    file1.write_text("content1", encoding="utf-8")

    assert file1.exists()
    assert len(list(tmp_path.rglob("*.txt"))) == 1
```

### tmp_path_factory フィクスチャ

```python
@pytest.fixture(scope="session")
def shared_data_file(tmp_path_factory):
    """セッション全体で共有される一時ファイル"""
    data_dir = tmp_path_factory.mktemp("data")
    file_path = data_dir / "shared.json"

    # 高コストなデータ生成（1回のみ）
    data = generate_expensive_data()
    file_path.write_text(json.dumps(data), encoding="utf-8")

    return file_path

def test_read_shared_data(shared_data_file):
    data = json.loads(shared_data_file.read_text(encoding="utf-8"))
    assert "key" in data
```

## モックとモンキーパッチ

### monkeypatch フィクスチャ

```python
def test_environment_variable(monkeypatch):
    # 環境変数の設定
    monkeypatch.setenv("API_KEY", "test_key")
    assert os.getenv("API_KEY") == "test_key"

    # 環境変数の削除
    monkeypatch.delenv("API_KEY", raising=False)

def test_mock_function(monkeypatch):
    # 関数の置き換え
    def mock_api_call(*args, **kwargs):
        return {"status": "success", "data": "mocked"}

    monkeypatch.setattr("mymodule.api_call", mock_api_call)
    result = mymodule.function_that_calls_api()
    assert result["status"] == "success"

def test_mock_method(monkeypatch):
    # メソッドの置き換え
    class MockResponse:
        status_code = 200
        def json(self):
            return {"mocked": True}

    def mock_get(*args, **kwargs):
        return MockResponse()

    monkeypatch.setattr("requests.get", mock_get)
```

### 辞書とクラス属性のパッチ

```python
def test_dict_patch(monkeypatch):
    # 辞書アイテムの設定
    config = {"debug": False}
    monkeypatch.setitem(config, "debug", True)
    assert config["debug"] is True

def test_class_attribute(monkeypatch):
    # クラス属性の置き換え
    monkeypatch.setattr(MyClass, "class_var", "new_value")
    assert MyClass.class_var == "new_value"
```

### context による限定的なパッチ

```python
def test_context_patch(monkeypatch):
    original_value = module.value

    with monkeypatch.context() as m:
        m.setattr(module, "value", "patched")
        assert module.value == "patched"

    # context 外では元に戻る
    assert module.value == original_value
```

### モックをフィクスチャ化（推奨）

```python
@pytest.fixture
def mock_api(monkeypatch):
    """再利用可能なAPIモック"""
    responses = []

    def mock_call(endpoint, **kwargs):
        responses.append((endpoint, kwargs))
        return {"status": "ok"}

    monkeypatch.setattr("api.call", mock_call)
    return responses

def test_with_mock_api(mock_api):
    result = my_function_that_uses_api()
    assert len(mock_api) == 2  # 2回呼ばれた
    assert mock_api[0][0] == "/users"  # 最初の呼び出しのエンドポイント
```

## テストの独立性（重要）

### 原則

各テストは他のテストから完全に独立し、任意の順序で実行可能でなければならない。

### 避けるべきパターン（NG）

```python
# ✗ 悪い例: グローバル状態の共有
shared_state = {"count": 0}

def test_increment():
    shared_state["count"] += 1
    assert shared_state["count"] == 1  # 実行順序に依存

def test_value():
    assert shared_state["count"] == 0  # test_increment の後だと失敗

# ✗ 悪い例: ファイルシステムの共有
def test_create_file():
    with open("test.txt", "w") as f:
        f.write("test")

def test_read_file():
    # test_create_file が先に実行されることを期待
    with open("test.txt", "r") as f:
        assert f.read() == "test"
```

### 推奨パターン（OK）

```python
# ✓ 良い例: フィクスチャで独立した状態を提供
@pytest.fixture
def isolated_state():
    return {"count": 0}

def test_increment(isolated_state):
    isolated_state["count"] += 1
    assert isolated_state["count"] == 1

def test_value(isolated_state):
    assert isolated_state["count"] == 0  # 常に独立した状態

# ✓ 良い例: tmp_path で独立したファイル
def test_create_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test", encoding="utf-8")
    assert file_path.exists()

def test_read_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test", encoding="utf-8")
    assert file_path.read_text(encoding="utf-8") == "test"
```

### 依存関係はフィクスチャで明示

```python
@pytest.fixture
def database():
    db = create_test_database()
    db.setup()
    yield db
    db.teardown()

@pytest.fixture
def user(database):
    # database フィクスチャに依存することを明示
    user = database.create_user("test")
    return user

def test_user_operations(user):
    # 依存関係が自動的に解決される
    assert user.name == "test"
```

## 避けるべきアンチパターン

### 複数の Act を含むテスト（NG）

```python
# ✗ 悪い例
def test_multiple_operations():
    result1 = function1()
    assert result1 == expected1

    result2 = function2()
    assert result2 == expected2

# ✓ 良い例: テストを分離
def test_function1():
    result = function1()
    assert result == expected1

def test_function2():
    result = function2()
    assert result == expected2
```

### try-except の使用（NG）

```python
# ✗ 悪い例
def test_exception():
    try:
        risky_function()
        assert False, "Exception not raised"
    except ValueError:
        pass  # 期待通り

# ✓ 良い例
def test_exception():
    with pytest.raises(ValueError):
        risky_function()
```

### 複雑な条件分岐（NG）

```python
# ✗ 悪い例
def test_with_condition():
    result = function()
    if condition:
        assert result == value1
    else:
        assert result == value2

# ✓ 良い例: パラメータ化で分離
@pytest.mark.parametrize("condition,expected", [
    (True, value1),
    (False, value2),
])
def test_with_condition(condition, expected):
    result = function(condition)
    assert result == expected
```

### ハードコードされたパス（NG）

```python
# ✗ 悪い例
def test_file():
    with open("/tmp/test.txt", "w") as f:
        f.write("test")

# ✓ 良い例
def test_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test", encoding="utf-8")
```

## プロジェクト固有のベストプラクティス

### ディレクトリ構造の推奨事項

```
tests/
    unit/           # 単体テスト
        test_models.py
        test_utils.py
    integration/    # 統合テスト
        test_api.py
        test_database.py
    conftest.py     # 共有フィクスチャ
    pytest.ini または pyproject.toml
```

### conftest.py の階層化

```python
# tests/conftest.py - すべてのテストで共有
@pytest.fixture
def app_config():
    return load_config("test")

# tests/unit/conftest.py - unit テストで共有
@pytest.fixture
def mock_database():
    return MockDatabase()

# tests/integration/conftest.py - integration テストで共有
@pytest.fixture
def real_database():
    db = connect_to_test_database()
    yield db
    db.cleanup()
```

### テストヘルパー関数

```python
# tests/helpers.py
def assert_valid_user(user):
    """再利用可能なアサーションヘルパー"""
    assert user.name is not None
    assert "@" in user.email
    assert user.created_at is not None

def create_test_user(**kwargs):
    """テストユーザー作成ヘルパー"""
    defaults = {
        "name": "test_user",
        "email": "test@example.com",
    }
    defaults.update(kwargs)
    return User(**defaults)

# tests/test_user.py
from tests.helpers import assert_valid_user, create_test_user

def test_user_creation():
    user = create_test_user(name="Alice")
    assert_valid_user(user)
    assert user.name == "Alice"
```

## コード例: 完全なテストファイル

```python
"""
ユーザー管理機能のテスト
"""
import pytest
from myapp.models import User
from myapp.services import UserService


# フィクスチャ定義
@pytest.fixture
def user_data():
    """テスト用ユーザーデータ"""
    return {
        "name": "test_user",
        "email": "test@example.com",
        "age": 25,
    }


@pytest.fixture
def database(tmp_path):
    """一時的なテスト用データベース"""
    db_path = tmp_path / "test.db"
    db = Database(db_path)
    db.initialize()
    yield db
    db.close()


@pytest.fixture
def user_service(database):
    """UserService インスタンス"""
    return UserService(database)


# 単体テスト
class TestUserModel:
    """User モデルのテスト"""

    def test_create_user(self, user_data):
        # Arrange
        # user_data フィクスチャが準備を担当

        # Act
        user = User(**user_data)

        # Assert
        assert user.name == "test_user"
        assert user.email == "test@example.com"
        assert user.age == 25

    @pytest.mark.parametrize("age,is_adult", [
        (17, False),
        (18, True),
        (20, True),
        (100, True),
    ], ids=["minor", "exactly-18", "adult", "senior"])
    def test_is_adult(self, user_data, age, is_adult):
        # Arrange
        user_data["age"] = age
        user = User(**user_data)

        # Act
        result = user.is_adult()

        # Assert
        assert result == is_adult

    def test_invalid_email(self, user_data):
        # Arrange
        user_data["email"] = "invalid"

        # Act & Assert
        with pytest.raises(ValueError, match="Invalid email"):
            User(**user_data)


# 統合テスト
@pytest.mark.integration
class TestUserService:
    """UserService の統合テスト"""

    def test_create_and_retrieve_user(self, user_service, user_data):
        # Arrange
        # フィクスチャが準備を担当

        # Act
        created_user = user_service.create_user(user_data)
        retrieved_user = user_service.get_user(created_user.id)

        # Assert
        assert retrieved_user is not None
        assert retrieved_user.name == user_data["name"]
        assert retrieved_user.email == user_data["email"]

    def test_duplicate_email(self, user_service, user_data):
        # Arrange
        user_service.create_user(user_data)

        # Act & Assert
        with pytest.raises(ValueError, match="Email already exists"):
            user_service.create_user(user_data)

    @pytest.mark.slow
    def test_bulk_user_creation(self, user_service):
        # Arrange
        users_data = [
            {"name": f"user{i}", "email": f"user{i}@example.com", "age": 20 + i}
            for i in range(1000)
        ]

        # Act
        created_users = user_service.bulk_create(users_data)

        # Assert
        assert len(created_users) == 1000
        assert user_service.count() == 1000


# モックを使用したテスト
class TestUserServiceWithMocks:
    """外部依存をモックしたテスト"""

    def test_send_welcome_email(self, user_service, user_data, monkeypatch):
        # Arrange
        email_sent = []

        def mock_send_email(to, subject, body):
            email_sent.append({"to": to, "subject": subject})

        monkeypatch.setattr("myapp.email.send_email", mock_send_email)

        # Act
        user = user_service.create_user(user_data)

        # Assert
        assert len(email_sent) == 1
        assert email_sent[0]["to"] == user.email
        assert "Welcome" in email_sent[0]["subject"]
```

## 実行とレポート

### 基本的な実行コマンド

```bash
# すべてのテストを実行
pytest

# 詳細な出力
pytest -v

# 失敗したテストのみ再実行
pytest --lf

# 最初の失敗で停止
pytest -x

# 並列実行（pytest-xdist が必要）
pytest -n auto

# カバレッジ測定
pytest --cov=myapp --cov-report=html
```

### マーカーによる選択的実行

```bash
# 単体テストのみ
pytest -m unit

# 統合テストを除外
pytest -m "not integration"

# 高速なテストのみ
pytest -m "not slow"
```

## テストの速度とパフォーマンス

### 高速なテストを書く

```python
# ✓ 良い例: 外部依存をモック
def test_api_call_fast(monkeypatch):
    def mock_get(*args, **kwargs):
        return MockResponse(status_code=200, data={"result": "success"})
    
    monkeypatch.setattr(requests, "get", mock_get)
    result = api_client.fetch_data()
    assert result["result"] == "success"

# ✗ 避ける: 実際のAPIを呼び出す
def test_api_call_slow():
    response = requests.get("https://api.example.com/data")
    assert response.status_code == 200
```

### データベーステストの高速化

```python
# インメモリSQLiteを使用
@pytest.fixture(scope="session")
def db_engine():
    from sqlalchemy import create_engine
    engine = create_engine("sqlite:///:memory:")
    # テーブル作成
    Base.metadata.create_all(engine)
    return engine
```

### 適切なスコープで高コストなリソースを共有

```python
# セッションスコープで一度だけ初期化
@pytest.fixture(scope="session")
def expensive_resource():
    resource = setup_expensive_resource()  # 時間がかかる
    yield resource
    resource.cleanup()

# 関数スコープで毎回新しいインスタンス
@pytest.fixture
def mutable_data():
    return {"counter": 0}
```

### 遅いテストにマーカーを付ける

```python
import pytest

@pytest.mark.slow
def test_heavy_computation():
    result = perform_complex_calculation()
    assert result == expected

# 通常のテスト実行では遅いテストを除外
# pytest -m "not slow"
```

### 並列実行の設定

```bash
# pytest-xdist をインストール
pip install pytest-xdist

# CPUコア数に応じて並列実行
pytest -n auto

# 4プロセスで実行
pytest -n 4

# 遅いテストの識別
pytest --durations=10
```

**重要:**
- テストは1秒以内に完了することを目指す
- 外部サービス（API、実DB）は常にモックする
- 並列実行でテストの独立性を検証できる

## 決定性のあるテスト（非フレーキーテスト）

### ランダム値の固定

```python
import random
import pytest

# ✓ 良い例: シードを固定
@pytest.fixture(autouse=True)
def reset_random_seed():
    random.seed(42)
    yield

def test_with_random():
    value = random.randint(1, 100)  # 常に同じ値
    assert process(value) == expected_result

# ✗ 避ける: 真のランダム性
def test_random_bad():
    value = random.randint(1, 100)  # 実行ごとに異なる
    assert process(value) > 0  # 不安定
```

### 時間依存コードのモック

```python
from datetime import datetime, timezone
import pytest

# freezegun を使用
from freezegun import freeze_time

@freeze_time("2024-01-01 12:00:00")
def test_with_fixed_time():
    now = datetime.now(timezone.utc)
    assert now.year == 2024
    assert now.month == 1

# または monkeypatch を使用
@pytest.fixture
def fixed_datetime(monkeypatch):
    fixed_time = datetime(2024, 1, 1, 12, 0, 0, tzinfo=timezone.utc)
    
    class MockDatetime:
        @classmethod
        def now(cls, tz=None):
            return fixed_time
    
    monkeypatch.setattr("myapp.datetime", MockDatetime)
    return fixed_time
```

### UUID の固定

```python
from uuid import UUID
import pytest

@pytest.fixture
def fixed_uuid(monkeypatch):
    fixed_id = UUID('12345678-1234-5678-1234-567812345678')
    monkeypatch.setattr('uuid.uuid4', lambda: fixed_id)
    return fixed_id

def test_entity_creation(fixed_uuid):
    entity = create_entity()
    assert entity.id == fixed_uuid
```

### 非同期処理のタイムアウト

```python
import pytest
import asyncio

@pytest.mark.asyncio
async def test_async_with_timeout():
    # 明示的なタイムアウトを設定
    result = await asyncio.wait_for(
        fetch_data(),
        timeout=5.0
    )
    assert result is not None
```

**重要:**
- ランダム値には固定シードを使用
- 時間依存のコードは freezegun や monkeypatch でモック
- 非同期処理には適切なタイムアウトを設定
- `pytest-random-order` で実行順序の独立性を検証

## エッジケースとバウンダリーテスト

### バウンダリー値を必ずテスト

```python
import pytest

@pytest.mark.parametrize("value,expected", [
    # 境界値のテスト
    (0, "zero"),              # 最小値
    (1, "positive"),          # 最小値 + 1
    (-1, "negative"),         # 最小値 - 1
    (100, "maximum"),         # 最大値
    (99, "positive"),         # 最大値 - 1
    (101, "error"),           # 最大値 + 1（範囲外）
])
def test_boundary_values(value, expected):
    result = classify_number(value)
    assert result == expected
```

### 空のコレクションをテスト

```python
def test_empty_collections():
    # 空リスト
    assert process_items([]) == []
    
    # 空文字列
    assert format_text("") == ""
    
    # 空辞書
    assert transform_data({}) == {}
    
    # None
    assert safe_process(None) is None
```

### 文字列のエッジケース

```python
@pytest.mark.parametrize("input_str,expected", [
    ("", ""),                           # 空文字列
    (" ", ""),                          # スペースのみ
    ("\n\t", ""),                      # 空白文字のみ
    ("a", "a"),                         # 1文字
    ("あ", "あ"),                       # マルチバイト
    ("emoji 😀", "emoji 😀"),          # 絵文字
    ("line1\nline2", "line1\nline2"),  # 改行
    ("a" * 1000, "a" * 1000),          # 非常に長い文字列
])
def test_string_edge_cases(input_str, expected):
    result = process_string(input_str)
    assert result == expected
```

### 数値の特殊ケース

```python
import math
import pytest

@pytest.mark.parametrize("value", [
    0,                  # ゼロ
    1,                  # 正の最小
    -1,                 # 負の最大
    float('inf'),       # 無限大
    float('-inf'),      # 負の無限大
    float('nan'),       # NaN
    2**31 - 1,         # 32bit整数の最大値
    -(2**31),          # 32bit整数の最小値
])
def test_numeric_edge_cases(value):
    if math.isnan(value) or math.isinf(value):
        with pytest.raises(ValueError):
            process_number(value)
    else:
        result = process_number(value)
        assert isinstance(result, (int, float))
```

### 例外の発生条件

```python
def test_exception_edge_cases():
    # ゼロ除算
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)
    
    # インデックス範囲外
    data = [1, 2, 3]
    with pytest.raises(IndexError):
        get_item(data, 10)
    
    # 型エラー
    with pytest.raises(TypeError):
        add_numbers("string", 5)
```

**重要:**
- 最小値、最大値、境界値±1を必ずテスト
- 空のコレクション、None、特殊文字を確認
- 無限大、NaN などの特殊な数値をテスト
- 例外が期待される条件を明示的にテスト

## テストカバレッジの測定

### pytest-cov の設定

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = [
    "--cov=src",
    "--cov-report=html",
    "--cov-report=term-missing",
    "--cov-fail-under=80",  # 80%未満で失敗
]

[tool.coverage.run]
source = ["src"]
branch = true  # ブランチカバレッジを有効化
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
    "@(abc\\.)?abstractmethod",
]
```

### カバレッジから除外

```python
def critical_function():
    """重要な関数は必ずテスト"""
    return calculate_result()

def debug_helper():  # pragma: no cover
    """デバッグ用の関数は除外"""
    print("Debug info")

if __name__ == "__main__":  # pragma: no cover
    # スクリプト実行部分は除外
    main()
```

### カバレッジコマンド

```bash
# 基本的なカバレッジ測定
pytest --cov=src

# HTML レポート生成
pytest --cov=src --cov-report=html

# 未カバーの行を表示
pytest --cov=src --cov-report=term-missing

# 80%未満で失敗
pytest --cov=src --cov-fail-under=80

# ブランチカバレッジ
pytest --cov=src --cov-branch
```

**カバレッジの目標:**
- 一般的なアプリケーション: 80%以上
- 重要なシステム（金融など）: 90-95%以上
- 100%を盲目的に追求しない（質が重要）

## CI/CD 統合

### GitHub Actions の設定例

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/pyproject.toml') }}
    
    - name: Install dependencies
      run: |
        pip install -e .[test]
        pip install pytest-xdist pytest-cov
    
    - name: Run tests
      run: |
        pytest -n auto -v --cov=src --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      if: matrix.python-version == '3.11'
```

### 高速化されたCI設定

```yaml
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
    
    - name: Cache
      uses: actions/cache@v3
      with:
        path: |
          ~/.cache/pip
          .pytest_cache
        key: ${{ runner.os }}-pytest-${{ hashFiles('**/pyproject.toml') }}
    
    - name: Install and test
      run: |
        pip install -e .[test] pytest-xdist
        pytest -n auto --dist loadgroup
```

### ステージ別のテスト実行

```yaml
name: Staged Tests

jobs:
  # 高速なユニットテスト（常に実行）
  unit:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest tests/unit -m "not slow"
  
  # 統合テスト（mainブランチのみ）
  integration:
    if: github.ref == 'refs/heads/main'
    needs: unit
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest tests/integration
  
  # 遅いテスト（スケジュール実行）
  slow:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
    - run: pip install -e .[test]
    - run: pytest -m slow
```

### pre-commit での自動テスト

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: pytest
        name: Run pytest
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
        args: ['-v', '-m', 'not slow']
```

**CI/CD のベストプラクティス:**
- 全てのプッシュ/PRでテストを自動実行
- 複数のPythonバージョンでテスト
- 依存関係をキャッシュして高速化
- 並列実行で時間を短縮
- カバレッジレポートを自動生成
- 遅いテストは別パイプラインで実行
- マージ前にテストの成功を必須に

## チェックリスト

テストコード作成時の確認事項：

### 基本
- [ ] テストファイルと関数の命名規則に従っているか
- [ ] Arrange-Act-Assert-Cleanup パターンに従っているか
- [ ] フィクスチャを適切に使用しているか
- [ ] テストは独立しているか（実行順序に依存していないか）
- [ ] パラメータ化で重複を削減しているか
- [ ] 適切なマーカーを付けているか

### アサーションとモック
- [ ] 例外テストに `pytest.raises()` を使用しているか
- [ ] 一時ファイルに `tmp_path` を使用しているか
- [ ] 外部依存を `monkeypatch` でモックしているか
- [ ] 浮動小数点数の比較に `pytest.approx()` を使用しているか

### パフォーマンスと決定性
- [ ] テストは1秒以内に完了するか
- [ ] 外部API・DBをモックしているか
- [ ] ランダム値に固定シードを使用しているか
- [ ] 時間依存のコードをモックしているか
- [ ] 遅いテストに `@pytest.mark.slow` を付けているか

### エッジケースとカバレッジ
- [ ] 境界値（最小値、最大値、±1）をテストしているか
- [ ] 空のコレクション、None をテストしているか
- [ ] 例外の発生条件を明示的にテストしているか
- [ ] テストカバレッジが80%以上あるか
- [ ] 未カバーの重要なコードパスはないか

### 設定とCI/CD
- [ ] カスタムマーカーを `pyproject.toml` に登録しているか
- [ ] `strict = true` を設定しているか
- [ ] 複雑なロジックをテストに含めていないか
- [ ] CI/CDでテストが自動実行されるか
- [ ] 並列実行でテストの独立性が保たれているか

## 参考リソース

- [pytest 公式ドキュメント](https://docs.pytest.org/)
- [Good Integration Practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
- [How to use fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
