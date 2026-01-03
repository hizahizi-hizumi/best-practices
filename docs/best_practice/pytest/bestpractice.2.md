# pytest ベストプラクティス集

pytestで単体テストを作成する際のベストプラクティスについて、主要なウェブサイトから収集した情報をまとめました。

## 目次

1. [テストの基本原則](#テストの基本原則)
2. [テストの構造と命名規則](#テストの構造と命名規則)
3. [フィクスチャの活用](#フィクスチャの活用)
4. [パラメータ化テスト](#パラメータ化テスト)
5. [モッキング](#モッキング)
6. [テストの整理と構成](#テストの整理と構成)
7. [テストカバレッジ](#テストカバレッジ)
8. [CI/CD統合](#cicd統合)
9. [その他のベストプラクティス](#その他のベストプラクティス)
10. [参考リソース](#参考リソース)

## テストの基本原則

### 1. テストは高速であるべき

**理由:**
- 高速なテストは頻繁な実行を促進し、バグの早期発見につながる
- 開発サイクルの迅速化
- CI環境での効率的な実行

**実践方法:**
- 外部依存を最小限に抑える
- モックを活用して外部リソースへの依存を削減
- 複雑なデータ構造を避ける

```python
# 良い例: 軽量で高速なテスト
def test_addition():
    assert add(2, 3) == 5

# 避けるべき例: 遅い外部API呼び出し
# def test_slow_api():
#     response = requests.get("https://api.example.com")  # 遅い
#     assert response.status_code == 200
```

### 2. テストは独立している必要がある

**理由:**
- テストの実行順序に依存しない
- 並列実行が可能
- デバッグが容易

**実践方法:**
- 各テストで新しいデータセットを使用
- Setup/Teardownでテスト環境を整える
- Fixtureのスコープを適切に設定

```python
@pytest.fixture
def fresh_dataset():
    # セットアップ
    dataset = create_dataset()
    yield dataset
    # クリーンアップ
    dataset.cleanup()

def test_sample(fresh_dataset):
    assert some_condition(fresh_dataset) == expected_result
```

### 3. 1つのテストは1つのことをテストする

**理由:**
- エラーの原因を特定しやすい
- テストの可読性向上
- メンテナンスが容易

```python
# 悪い例: 複数の機能を1つのテストで検証
def test_user_creation_and_email_sending():
    user = create_user("test@example.com")
    assert user.email == "test@example.com"
    send_welcome_email(user)
    assert email_service.last_sent_email == "test@example.com"

# 良い例: 機能ごとに分割
def test_user_creation():
    user = create_user("test@example.com")
    assert user.email == "test@example.com"

def test_welcome_email_sending():
    user = User("test@example.com")
    send_welcome_email(user)
    assert email_service.last_sent_email == "test@example.com"
```

### 4. テストは決定的である必要がある

**理由:**
- 一貫した結果を提供
- 信頼性の向上
- デバッグの容易さ

```python
# 良い例: 決定的なテスト
def test_addition():
    assert 2 + 2 == 4

# 悪い例: 非決定的なテスト（ランダム性を含む）
def test_random_number_is_even():
    assert random.randint(1, 100) % 2 == 0  # 時々失敗する

# 修正例: 固定シードを使用
def test_random_number_is_even_fixed():
    random.seed(42)
    assert random.randint(1, 100) % 2 == 0
```

### 5. テストは読みやすくメンテナンス可能である必要がある

**実践方法:**
- 説明的な関数名を使用
- AAAパターン（Arrange, Act, Assert）に従う
- テストを小さく保つ

```python
# 悪い例
def test1():
    assert len([]) == 0

# 良い例
def test_empty_list_has_zero_length():
    # Arrange
    empty_list = []

    # Act
    length = len(empty_list)

    # Assert
    assert length == 0
```

## テストの構造と命名規則

### ファイル構成

```
project/
├── src/
│   ├── models/
│   │   └── user.py
│   └── services/
│       └── payment_service.py
└── tests/
    ├── unit/
    │   ├── test_user.py
    │   └── test_payment_service.py
    ├── integration/
    │   └── test_api_endpoints.py
    ├── e2e/
    │   └── test_user_journey.py
    ├── fixtures/
    │   ├── fixtures_db.py
    │   └── fixtures_api.py
    ├── conftest.py
    └── pytest.ini
```

### 命名規則

**テストファイル:**
- `test_*.py` または `*_test.py`
- 説明的な名前を使用（例: `test_user_authentication.py`）

**テスト関数:**
- `test_` で始める
- 何をテストしているか明確に表現

```python
# 良い命名例
def test_user_login_with_valid_credentials():
    pass

def test_user_login_with_invalid_credentials():
    pass

def test_factorial_of_large_number():
    pass
```

**テストクラス:**
- `Test` で始める（大文字）
- 関連するテストをグループ化

```python
class TestUserAuthentication:
    def test_successful_login(self):
        pass

    def test_failed_login_invalid_password(self):
        pass
```

## フィクスチャの活用

### フィクスチャの基本

フィクスチャはテストに必要なリソースのセットアップとクリーンアップを行う強力な機能です。

```python
import pytest

@pytest.fixture
def sample_data():
    return {"name": "Alice", "age": 30}

def test_person_data(sample_data):
    assert sample_data["name"] == "Alice"
    assert sample_data["age"] == 30
```

### フィクスチャのスコープ

```python
# Function スコープ（デフォルト）: 各テスト関数ごとに実行
@pytest.fixture(scope="function")
def temp_file():
    with open("temp.txt", "w") as f:
        yield f

# Class スコープ: テストクラスごとに1回実行
@pytest.fixture(scope="class")
def sample_data():
    return {"key": "value"}

# Module スコープ: モジュールごとに1回実行
@pytest.fixture(scope="module")
def db_connection():
    conn = connect_to_db()
    yield conn
    conn.close()

# Session スコープ: テストセッション全体で1回実行
@pytest.fixture(scope="session")
def app_config():
    return load_config()
```

### フィクスチャの整理

**集中管理 vs ローカル配置:**

```
# ハイブリッドアプローチ（推奨）
tests/
├── fixtures/
│   ├── fixtures_db.py      # 共有データベースフィクスチャ
│   ├── fixtures_api.py     # 共有APIフィクスチャ
│   └── fixtures_auth.py    # 共有認証フィクスチャ
├── unit/
│   ├── conftest.py         # ユニットテスト固有のフィクスチャ
│   └── test_models.py
├── integration/
│   ├── conftest.py         # 統合テスト固有のフィクスチャ
│   └── test_api.py
└── conftest.py             # グローバルフィクスチャ
```

**フィクスチャ値を変更しない:**

```python
# 悪い例
@pytest.fixture
def user_data():
    return {"name": "John", "age": 30}

def test_modify_user(user_data):
    user_data["age"] = 31  # フィクスチャを変更すべきでない
    assert user_data["age"] == 31

# 良い例
@pytest.fixture
def user_data():
    return {"name": "John", "age": 30}

def test_user_age(user_data):
    modified_data = user_data.copy()
    modified_data["age"] = 31
    assert modified_data["age"] == 31
```

## パラメータ化テスト

### 基本的なパラメータ化

```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (-10, 5, -5),
    (0, 0, 0),
    (100, -50, 50)
])
def test_add_numbers(a, b, expected):
    result = add_numbers(a, b)
    assert result == expected
```

### カスタムIDを使用したパラメータ化

```python
@pytest.mark.parametrize("size_bytes, expected_result", [
    pytest.param(0, "0B", id="zero_bytes"),
    pytest.param(1, "1.00 B", id="one_byte"),
    pytest.param(1024, "1.00 KB", id="one_kilobyte"),
    pytest.param(1024**2, "1.00 MB", id="one_megabyte"),
])
def test_format_file_size(size_bytes, expected_result):
    assert format_file_size(size_bytes) == expected_result
```

### データクラスを使用したパラメータ化

```python
from dataclasses import dataclass, field

@dataclass
class FileSizeTestCase:
    size_bytes: int
    expected_result: str
    id: str = field(init=False)

    def __post_init__(self):
        self.id = f"test_{self.size_bytes}_bytes"

test_cases = [
    FileSizeTestCase(0, "0B"),
    FileSizeTestCase(1024, "1.00 KB"),
    FileSizeTestCase(1024**2, "1.00 MB"),
]

@pytest.mark.parametrize("test_case", test_cases, ids=lambda tc: tc.id)
def test_format_file_size(test_case):
    assert format_file_size(test_case.size_bytes) == test_case.expected_result
```

## モッキング

### mockerとpytest-mock

```python
# mocker over mock を選択
def test_make_a_dict(mocker):
    """
    make_a_dict関数が期待される辞書を返すことをテスト
    """
    mocker.patch(
        "__main__.some_calculation",
        return_value=5,
        autospec=True  # 常にautospec=Trueを使用
    )

    my_dict = make_a_dict(2, 3)
    expected_dict = {"a": 2, "b": 3, "result": 5}

    assert my_dict == expected_dict
```

### autospec=True の重要性

```python
# 良い例: autospec=True を使用
def test_with_autospec(mocker):
    mocker.patch(
        "__main__.some_function",
        return_value=42,
        autospec=True  # 関数のシグネチャを検証
    )
```

### テストしたくないものをモック化

```python
# 外部APIの呼び出しをモック化
@patch("requests.get")
def test_fetch_data(mock_get):
    mock_get.return_value.json.return_value = {"status": "success"}
    assert fetch_data("https://api.example.com") == {"status": "success"}
```

## テストの整理と構成

### テストピラミッドに基づく構成

```
tests/
├── unit/                    # 多数の高速なユニットテスト
│   ├── test_models.py
│   └── test_services.py
├── integration/             # 中程度の数の統合テスト
│   ├── test_api_endpoints.py
│   └── test_database_interactions.py
└── e2e/                     # 少数のE2Eテスト
    └── test_user_journey.py
```

### アプリケーションコードをミラーリング

```
# アプリケーション構造
src/
├── models/
│   ├── user.py
│   └── order.py
├── services/
│   └── payment_service.py
└── controllers/
    └── order_controller.py

# テスト構造（ミラーリング）
tests/
├── models/
│   ├── test_user.py
│   └── test_order.py
├── services/
│   └── test_payment_service.py
└── controllers/
    └── test_order_controller.py
```

### マーカーを使用したテストの分類

```python
import pytest

@pytest.mark.slow
def test_slow_function():
    import time
    time.sleep(5)
    assert True

@pytest.mark.fast
def test_fast_function():
    assert True

# pytest.ini で登録
# [pytest]
# markers =
#     slow: marks tests as slow
#     fast: marks tests as fast
```

**実行例:**

```bash
# 遅いテストのみ実行
pytest -m slow

# 速いテストのみ実行
pytest -m fast

# 遅いテストを除外
pytest -m "not slow"
```

## テストカバレッジ

### カバレッジ目標の設定

- 通常70〜80%を目標とする
- プロジェクトとチームで議論して決定
- CI/CDパイプラインで強制することも可能

### pytest-covの使用

```bash
# インストール
pip install pytest-cov

# カバレッジレポートの生成
pytest --cov=my_project

# HTMLレポートの生成
pytest --cov=my_project --cov-report=html
```

### カバレッジの分析

```python
# coverage.py を使用
pip install coverage

# テスト実行
coverage run -m pytest

# レポート表示
coverage report

# HTMLレポート生成
coverage html
```

**重要:** カバレッジが高いことは、コードがテストされていることを意味しますが、バグがないことを保証するものではありません。エッジケースや様々なテストデータでのテストも重要です。

## CI/CD統合

### GitHub Actionsの例

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.8'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pytest pytest-cov
        pip install -r requirements.txt

    - name: Run tests
      run: |
        pytest --cov=src --cov-report=xml

    - name: Upload coverage
      uses: codecov/codecov-action@v2
```

### Jenkinsパイプラインの例

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo/your-project.git'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'pip install pytest pytest-cov'
                sh 'pip install -r requirements.txt'
            }
        }
        stage('Run Tests') {
            steps {
                sh 'pytest --cov=src --junitxml=report.xml'
            }
        }
    }
}
```

## その他のベストプラクティス

### 1. 実装詳細ではなく、動作をテストする

```python
# 悪い例: 実装詳細をテスト
def test_add():
    assert 2 + 3 == 5  # 実装詳細を含む

# 良い例: 動作をテスト
def test_add():
    assert add(2, 3) == 5  # 実装から独立

# より良い例: ユーザーの観点から動作をテスト
def test_added_user_can_be_retrieved_by_username():
    user = User(username="johndoe")
    repository = InMemoryUserRepository()
    repository.add(user)

    assert user == repository.get_by_username(user.username)
```

### 2. フレームワークをテストしない

```python
# 悪い例: Django/Flaskのフレームワーク機能をテスト
@pytest.mark.django_db
def test_django_authentication():
    user = User.objects.create_user(username='testuser', password='testpassword')
    authenticated = user.check_password('testpassword')
    assert authenticated  # Djangoの機能をテストしている

# 良い例: 自分のビジネスロジックをテスト
def test_user_service_creates_user():
    user_service = UserService()
    user = user_service.create_user("testuser", "testpassword")
    assert user.username == "testuser"
```

### 3. DRYの原則（繰り返しを避ける）

**フィクスチャの活用:**

```python
# 悪い例: コードの重複
def test_person_is_adult():
    person = Person("Emi")
    person.age = 19
    assert person.is_adult()

def test_person_is_not_adult():
    person = Person("Emi")
    person.age = 10
    assert not person.is_adult()

# 良い例: フィクスチャで共通部分を抽出
@pytest.fixture
def person():
    return Person("Emi")

def test_person_is_adult(person):
    person.age = 19
    assert person.is_adult()

def test_person_is_not_adult(person):
    person.age = 10
    assert not person.is_adult()
```

### 4. 例外処理のテスト

```python
def test_format_file_size_negative_size():
    with pytest.raises(ValueError, match="Size cannot be negative"):
        format_file_size(-1)

# 複数の例外ケースをパラメータ化
@pytest.mark.parametrize("invalid_input,error_message", [
    (-1, "Size cannot be negative"),
    ("invalid", "Invalid input type"),
    (None, "Input cannot be None"),
])
def test_format_file_size_invalid_input(invalid_input, error_message):
    with pytest.raises(ValueError, match=error_message):
        format_file_size(invalid_input)
```

### 5. テストの依存関係注入

```python
# 悪い例: テストが困難
class DataManager:
    def __init__(self):
        self.db = DatabaseConnector()

    def data_operation(self):
        return self.db.connect() + " - Data operation done"

# 良い例: 依存関係注入を使用
class DataManager:
    def __init__(self, db_connector):
        self.db = db_connector

    def data_operation(self):
        return self.db.connect() + " - Data operation done"

# テスト
class MockDatabaseConnector:
    def connect(self):
        return "Mock database connection"

def test_data_operation():
    mock_db = MockDatabaseConnector()
    data_manager = DataManager(mock_db)
    result = data_manager.data_operation()
    assert result == "Mock database connection - Data operation done"
```

### 6. セキュリティテスト

```python
def test_sql_injection_vulnerability():
    malicious_input = "1; DROP TABLE users"
    with pytest.raises(SecurityException):
        get_user_details(malicious_input)
```

### 7. 偽データ/データベースの使用

```python
# 偽データの使用
def test_calculate_average_age():
    fake_data = [{'age': 25}, {'age': 35}, {'age': 45}]
    average_age = calculate_average_age(fake_data)
    assert average_age == 35

# モックデータベースの使用
from unittest.mock import MagicMock

def test_user_creation():
    db = MagicMock()
    db.insert_user.return_value = True
    user_service = UserService(db)
    result = user_service.create_user("John Doe", "john@example.com")
    assert result is True
```

### 8. テストの定期的なレビューとリファクタリング

- 冗長性を削除
- コード変更に応じてテストを更新
- テストの構成と命名を改善
- テストスイートを最新の状態に保つ

### 9. フレイキーテスト（不安定なテスト）への対処

```bash
# pytest-rerunfailuresを使用
pip install pytest-rerunfailures
pytest --reruns 3 --reruns-delay 5
```

**フレイキーテストの根本原因:**
- タイミングの問題
- 外部システムへの依存
- ランダム性
- テスト実行順序への依存

### 10. Test Driven Development (TDD)

1. **Red**: 失敗するテストを書く
2. **Green**: テストを通過させる最小限のコードを書く
3. **Refactor**: コードをリファクタリング

```python
# Step 1: 失敗するテストを書く
def test_add():
    assert add(2, 3) == 5

# Step 2: 実装
def add(a, b):
    return a + b

# Step 3: リファクタリング（必要に応じて）
```

## 参考リソース

### 主要リンク

1. **BetterStack - A Beginner's Guide to Unit Testing with Pytest**
   https://betterstack.com/community/guides/testing/pytest-guide/
   - 初心者向けの包括的なガイド
   - Fixturesとパラメータ化の詳細な説明

2. **pytest-with-eric.com - Python Unit Testing Best Practices**
   https://pytest-with-eric.com/introduction/python-unit-testing-best-practices/
   - テストの速度、独立性、単一責任について
   - 実践的なコード例

3. **emimartin.me - Pytest Best Practices**
   https://emimartin.me/pytest_best_practices
   - 良いテストと悪いテストの具体例
   - モッキングとパラメータ化の実践

4. **dev.to - Python Testing – Unit Tests, Pytest, and Best Practices**
   https://dev.to/nkpydev/python-testing-unit-tests-pytest-and-best-practices-45gl
   - 基本的な単体テストの書き方
   - Fixturesとパラメータ化テスト

5. **Medium - Detailed Guide to Writing Efficient Unit Tests in Python with Pytest**
   https://medium.com/@ydmarinb/detailed-guide-to-writing-efficient-unit-tests-in-python-with-pytest-2e56b33cb7dc
   - 効率的な単体テストの書き方
   - パラメータ化と例外処理

6. **irislogic.com - Best Practices and Advanced Techniques with Pytest**
   https://irislogic.com/best-practices-and-advanced-techniques-with-pytest/
   - Fixturesの高度な使用方法
   - テストデータの作成とセットアップ

7. **pytest-with-eric.com - 5 Best Practices For Organizing Tests**
   https://pytest-with-eric.com/pytest-best-practices/pytest-organize-tests/
   - テスト構造の最適化
   - アプリケーションコードをミラーリングする配置

8. **namastedev.com - Implementing Unit Testing in Python with pytest**
   https://namastedev.com/blog/implementing-unit-testing-in-python-with-pytest-a-quick-start-guide/
   - pytestの基本から実践的な使用方法
   - ベストプラクティス

9. **Medium - Testing Best Practices with Pytest**
   https://medium.com/@ngattai.lam/testing-best-practices-with-pytest-a2079d5e842b
   - テストカバレッジの重要性（70%以上推奨）
   - フレームワークの選択

10. **Dev Genius - Pytest in 2025: A Complete Guide for Python Developers**
    https://blog.devgenius.io/pytest-in-2025-a-complete-guide-for-python-developers-9b15ae0fe07e
    - 2025年版の最新ガイド
    - 頻繁なテスト実行とFixtures活用

### 追加の有用なリソース

- **Reddit - What are best practices with Pytest?**
  https://www.reddit.com/r/Python/comments/o2pcj1/what_are_best_practices_with_pytest/

- **Towards Data Science - Pytest with Marking, Mocking, and Fixtures in 10 Minutes**
  https://towardsdatascience.com/pytest-with-marking-mocking-and-fixtures-in-10-minutes-678d7ccd2f70

- **Inspired Python - Five Advanced Pytest Fixture Patterns**
  https://www.inspiredpython.com/article/five-advanced-pytest-fixture-patterns

- **DataCamp - pytest-mock Tutorial**
  https://www.datacamp.com/tutorial/pytest-mock

### 公式ドキュメント

- **Pytest公式ドキュメント**: https://docs.pytest.org/
- **Pytest Good Integration Practices**: https://docs.pytest.org/en/7.1.x/explanation/goodpractices.html
- **Pytest Plugin List**: https://docs.pytest.org/en/7.1.x/reference/plugin_list.html

## まとめ

pytestでの単体テスト作成における主要なベストプラクティス:

1. **テストの基本原則を守る**: 高速、独立、決定的、読みやすい
2. **適切な構造化**: テストピラミッド、ファイル構成、命名規則
3. **フィクスチャの効果的活用**: スコープの理解、集中管理とローカル配置のバランス
4. **パラメータ化でDRY**: 重複を避け、多様なケースをカバー
5. **適切なモッキング**: autospec=Trueの使用、外部依存の分離
6. **CI/CD統合**: 自動テスト実行、継続的な品質保証
7. **カバレッジ目標**: 70-80%を目標に、エッジケースにも注意
8. **継続的改善**: 定期的なレビュー、リファクタリング、TDDの実践

これらのベストプラクティスを実践することで、保守性が高く、スケーラブルで、信頼性の高いテストスイートを構築できます。
