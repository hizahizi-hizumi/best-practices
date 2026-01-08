---
description: '高品質なテストコードを作成するための pytest ベストプラクティス'
applyTo: '**/test_*.py, **/*_test.py, **/tests/**/*.py, **/conftest.py'
---

# pytest ベストプラクティス

pytest を用いた高品質な自動テストコードを作成するためのガイドライン。

## 目的と適用範囲

Python テストファイルに pytest のベストプラクティスを適用し、保守性、信頼性、高速実行を保証する。

## プロジェクト構造

### src レイアウトを使用する
```
pyproject.toml
src/
    mypkg/
        __init__.py
        app.py
tests/
    test_app.py
    conftest.py
```

**理由**: テスト時にインストールされていないパッケージを誤ってインポートすることを防ぐ。

### pyproject.toml を設定する
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

**必須マーカー**: すべてのカスタムマーカーを登録し、タイプミスを防ぐ。

### 命名規則に従う
- テストファイル: `test_*.py` または `*_test.py`
- テスト関数: `test_` で始める
- テストクラス: `Test` で始める (`__init__` メソッドは不要)
- テストメソッド: `test_` で始める

## テスト構造

### Arrange-Act-Assert パターンに従う
```python
def test_user_registration(database, email_service):
    # Arrange: テストデータとモックを準備する
    user_data = {"name": "test_user", "email": "test@example.com"}

    # Act: テスト対象の関数を実行する (1回のみ)
    result = register_user(user_data, database)

    # Assert: 結果を検証する
    assert result.success is True
    assert database.get_user("test_user") is not None
```

**ルール**:
- Act セクションは単一の関数呼び出しにする
- 同じ Act 結果を検証する場合は複数のアサーションを許可する
- クリーンアップはフィクスチャに委譲する

## フィクスチャ

### セットアップ/ティアダウンには yield フィクスチャを使用する
```python
@pytest.fixture
def database_connection():
    conn = create_connection("test.db")
    conn.initialize()
    yield conn
    conn.close()  # 常に実行される
```

**理由**: yield はテストが失敗してもクリーンアップが実行されることを保証する。

### 適切なスコープを選択する
- `function` (デフォルト): テストごとに新しいインスタンス (最も安全)
- `class`: テストクラスごとに1つのインスタンス
- `module`: モジュールごとに1つのインスタンス
- `session`: テストセッションごとに1つのインスタンス

**ルール**: ステートレスなリソースまたはコストの高い操作 (DB、Docker) の場合のみ広いスコープを使用する。

### conftest.py でフィクスチャを共有する
`tests/conftest.py` に共通フィクスチャを配置し、すべてのテストで利用可能にする。

```python
# tests/conftest.py
@pytest.fixture
def api_client():
    return APIClient(base_url="http://test")
```

### フィクスチャをパラメータ化する
```python
@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def database(request):
    db = create_database(request.param)
    yield db
    db.close()
```

**効果**: 各パラメータ値に対してテストが1回実行される。

## アサーション

### シンプルな assert 文を使用する
```python
def test_addition():
    assert 2 + 2 == 4
    assert result > 0
```

### 浮動小数点数の比較には pytest.approx を使用する
```python
assert (0.1 + 0.2) == pytest.approx(0.3)
assert 1.0001 == pytest.approx(1.0, abs=0.001)
```

**理由**: 浮動小数点の精度誤差により直接の等価比較は失敗する。

### 例外のテストには pytest.raises を使用する
```python
def test_exception():
    with pytest.raises(ValueError, match=r"invalid literal.*"):
        int("invalid")
```

**try-except は絶対に使わない**: pytest.raises はより良いエラーメッセージを提供する。

### 警告のテストには pytest.warns を使用する
```python
def test_warning():
    with pytest.warns(UserWarning, match="deprecated"):
        warnings.warn("deprecated feature", UserWarning)
```

## パラメータ化

### テストをパラメータ化して重複を削減する
```python
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 3),
    (3, 4),
])
def test_increment(input, expected):
    assert increment(input) == expected
```

### 読みやすいテスト名のために ids を使用する
```python
@pytest.mark.parametrize("input,expected", [
    ({"name": "Alice"}, True),
    ({}, False),
], ids=["valid-user", "empty-dict"])
def test_validate_user(input, expected):
    assert validate_user(input) == expected
```

### 複数のデコレータを組み合わせてデカルト積を作る
```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_combination(x, y):
    # 4つのテストを実行: (0,2), (0,3), (1,2), (1,3)
    assert x + y >= 2
```

## マーカー

### pyproject.toml でカスタムマーカーを登録する
```toml
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
]
```

**ルール**: `--strict-markers` を使用してタイプミスを防ぐ。

### テストにマーカーを適用する
```python
@pytest.mark.unit
def test_pure_function():
    assert add(1, 2) == 3

@pytest.mark.slow
@pytest.mark.integration
def test_full_workflow():
    pass
```

### マーカーでテストを実行する
```bash
pytest -m unit              # ユニットテストのみ実行
pytest -m "not slow"        # 遅いテストを除外
pytest -m "integration or slow"  # 統合テストまたは遅いテストを実行
```

## スキップと XFail

### 条件付きでテストをスキップする
```python
@pytest.mark.skipif(sys.version_info < (3, 10), reason="requires python3.10+")
def test_new_feature():
    pass

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_feature():
    pass
```

### 予想される失敗を xfail でマークする
```python
@pytest.mark.xfail(reason="known bug #123")
def test_known_issue():
    assert buggy_function() == expected_value

@pytest.mark.xfail(strict=True, reason="must fail")
def test_must_fail():
    assert False  # テストが予期せず成功した場合はエラー
```

### オプションの依存関係には importorskip を使用する
```python
numpy = pytest.importorskip("numpy", minversion="1.20")

def test_with_numpy():
    arr = numpy.array([1, 2, 3])
    assert len(arr) == 3
```

## 一時ファイルとディレクトリ

### 一時ファイルには tmp_path フィクスチャを使用する
```python
def test_write_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test content", encoding="utf-8")
    assert file_path.read_text(encoding="utf-8") == "test content"
```

**理由**: tmp_path は分離された Path オブジェクトと自動クリーンアップを提供する。

### セッションスコープの一時ファイルには tmp_path_factory を使用する
```python
@pytest.fixture(scope="session")
def shared_data_file(tmp_path_factory):
    data_dir = tmp_path_factory.mktemp("data")
    file_path = data_dir / "shared.json"
    file_path.write_text(json.dumps(data), encoding="utf-8")
    return file_path
```

## モックとモンキーパッチ

### モックには monkeypatch フィクスチャを使用する
```python
def test_environment_variable(monkeypatch):
    monkeypatch.setenv("API_KEY", "test_key")
    assert os.getenv("API_KEY") == "test_key"

def test_mock_function(monkeypatch):
    def mock_api_call(*args, **kwargs):
        return {"status": "success"}
    monkeypatch.setattr("mymodule.api_call", mock_api_call)
    result = mymodule.function_that_calls_api()
    assert result["status"] == "success"
```

**理由**: monkeypatch はテスト実行後の自動クリーンアップを提供する。

### 再利用可能なモックフィクスチャを作成する
```python
@pytest.fixture
def mock_api(monkeypatch):
    responses = []
    def mock_call(endpoint, **kwargs):
        responses.append((endpoint, kwargs))
        return {"status": "ok"}
    monkeypatch.setattr("api.call", mock_call)
    return responses

def test_with_mock_api(mock_api):
    my_function_that_uses_api()
    assert len(mock_api) == 2
    assert mock_api[0][0] == "/users"
```

## テストの独立性

### テストが任意の順序で実行できることを保証する
各テストは完全に独立しており、他のテストに依存してはならない。

**ミュータブルな状態を共有しない**:
```python
# 悪い例: 共有されたグローバル状態
shared_state = {"count": 0}

def test_increment():
    shared_state["count"] += 1
    assert shared_state["count"] == 1  # test_value の後に実行すると失敗

# 良い例: フィクスチャを使った独立した状態
@pytest.fixture
def isolated_state():
    return {"count": 0}

def test_increment(isolated_state):
    isolated_state["count"] += 1
    assert isolated_state["count"] == 1
```

**理由**: テスト順序の依存関係は不安定なテストとデバッグの困難さを引き起こす。

### 並列実行で独立性を検証する
```bash
pytest -n auto  # pytest-xdist を使用
```

並列実行でテストが失敗する場合、隠れた依存関係が存在する。

## 避けるべきアンチパターン

### 単一のテストで複数の Act を避ける
```python
# 悪い例
def test_multiple_operations():
    result1 = function1()
    assert result1 == expected1
    result2 = function2()
    assert result2 == expected2

# 良い例: 別々のテストに分割する
def test_function1():
    assert function1() == expected1

def test_function2():
    assert function2() == expected2
```

**理由**: 単一のテストは単一の動作を検証するべきである。

### テストで try-except を避ける
```python
# 悪い例
def test_exception():
    try:
        risky_function()
        assert False
    except ValueError:
        pass

# 良い例
def test_exception():
    with pytest.raises(ValueError):
        risky_function()
```

### 複雑な条件分岐を避ける
```python
# 悪い例
def test_with_condition():
    if condition:
        assert result == value1
    else:
        assert result == value2

# 良い例: パラメータ化を使用する
@pytest.mark.parametrize("condition,expected", [
    (True, value1),
    (False, value2),
])
def test_with_condition(condition, expected):
    assert function(condition) == expected
```

### パスをハードコードしない
```python
# 悪い例
def test_file():
    with open("/tmp/test.txt", "w") as f:
        f.write("test")

# 良い例
def test_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test", encoding="utf-8")
```

## パフォーマンスと速度

### テストを高速に保つ (< 1秒)
- 外部サービス (API、データベース) をモックする
- データベーステストにはインメモリ SQLite を使用する
- コストの高いリソースは広いフィクスチャスコープで共有する

### 外部依存関係をモックする
```python
# 良い例: API 呼び出しをモックする
def test_api_call(monkeypatch):
    def mock_get(*args, **kwargs):
        return MockResponse(status_code=200)
    monkeypatch.setattr(requests, "get", mock_get)
    result = api_client.fetch_data()
    assert result["status"] == "ok"

# 悪い例: 実際の API 呼び出し
def test_api_call():
    response = requests.get("https://api.example.com/data")
    assert response.status_code == 200
```

### 遅いテストをマークする
```python
@pytest.mark.slow
def test_heavy_computation():
    result = perform_complex_calculation()
    assert result == expected
```

**遅いテストなしで実行**: `pytest -m "not slow"`

### テストを並列実行する
```bash
pip install pytest-xdist
pytest -n auto  # すべての CPU コアを使用
```

### 遅いテストを特定する
```bash
pytest --durations=10
```

## 決定論的テスト

### ランダムシードを固定する
```python
@pytest.fixture(autouse=True)
def reset_random_seed():
    random.seed(42)
    yield
```

**理由**: ランダムな値は不安定なテストを引き起こす。

### 時間依存コードをモックする
```python
from freezegun import freeze_time

@freeze_time("2024-01-01 12:00:00")
def test_with_fixed_time():
    now = datetime.now(timezone.utc)
    assert now.year == 2024
```

### UUID を固定する
```python
@pytest.fixture
def fixed_uuid(monkeypatch):
    fixed_id = UUID('12345678-1234-5678-1234-567812345678')
    monkeypatch.setattr('uuid.uuid4', lambda: fixed_id)
    return fixed_id
```

## 境界値とエッジケース

### 境界値をテストする
```python
@pytest.mark.parametrize("value,expected", [
    (0, "zero"),        # 最小値
    (1, "positive"),    # 最小値 + 1
    (-1, "negative"),   # 最小値 - 1
    (100, "maximum"),   # 最大値
    (101, "error"),     # 最大値 + 1 (範囲外)
])
def test_boundary_values(value, expected):
    assert classify_number(value) == expected
```

**理由**: バグは境界で発生することが多い。

### 空のコレクションをテストする
```python
def test_empty_collections():
    assert process_items([]) == []
    assert format_text("") == ""
    assert transform_data({}) == {}
    assert safe_process(None) is None
```

### 文字列のエッジケースをテストする
```python
@pytest.mark.parametrize("input_str", [
    "",             # 空文字列
    " ",            # 空白のみ
    "a",            # 単一文字
    "あ",           # マルチバイト
    "emoji 😀",     # 絵文字
    "a" * 1000,    # 非常に長い
])
def test_string_edge_cases(input_str):
    result = process_string(input_str)
    assert isinstance(result, str)
```

### 数値の特殊ケースをテストする
```python
@pytest.mark.parametrize("value", [
    0, 1, -1,
    float('inf'), float('-inf'), float('nan'),
    2**31 - 1, -(2**31),
])
def test_numeric_edge_cases(value):
    if math.isnan(value) or math.isinf(value):
        with pytest.raises(ValueError):
            process_number(value)
    else:
        assert isinstance(process_number(value), (int, float))
```

## テストカバレッジ

### pyproject.toml でカバレッジを設定する
```toml
[tool.pytest.ini_options]
addopts = [
    "--cov=src",
    "--cov-report=html",
    "--cov-report=term-missing",
    "--cov-fail-under=80",
]

[tool.coverage.run]
branch = true
omit = ["*/tests/*", "*/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "@(abc\\.)?abstractmethod",
]
```

### デバッグコードを除外する
```python
def debug_helper():  # pragma: no cover
    print("Debug info")

if __name__ == "__main__":  # pragma: no cover
    main()
```

### カバレッジを実行する
```bash
pytest --cov=src --cov-report=html
pytest --cov=src --cov-report=term-missing
pytest --cov=src --cov-fail-under=80
```

**目標**: 一般的なアプリケーションでは80%以上、重要なシステムでは90%以上。

**ルール**: 盲目的に100%カバレッジを追求せず、品質に焦点を当てる。

## CI/CD 統合

### GitHub Actions の例
```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/pyproject.toml') }}
    
    - name: Install and test
      run: |
        pip install -e .[test]
        pytest -n auto -v --cov=src --cov-report=xml
```

### 速度別にテストをステージング化する
```yaml
jobs:
  unit:
    steps:
    - run: pytest tests/unit -m "not slow"
  
  integration:
    needs: unit
    if: github.ref == 'refs/heads/main'
    steps:
    - run: pytest tests/integration
```

**ベストプラクティス**:
- すべてのプッシュ/PR でテストを実行する
- 複数の Python バージョンでテストする
- 依存関係をキャッシュして高速化する
- `-n auto` でテストを並列実行する
- マージ前にテストの合格を要求する

## チェックリスト

テストコードをコミットする前に:

### 基本
- [ ] 命名規則に従っている (test_*.py, test_*)
- [ ] Arrange-Act-Assert パターンを使用している
- [ ] setup/teardown の代わりにフィクスチャを使用している
- [ ] テストが独立している (順序に依存しない)
- [ ] パラメータ化を使用して重複を削減している
- [ ] 適切なマーカーを適用している

### アサーションとモック
- [ ] 例外には `pytest.raises()` を使用している
- [ ] 一時ファイルには `tmp_path` を使用している
- [ ] モックには `monkeypatch` を使用している
- [ ] 浮動小数点には `pytest.approx()` を使用している

### パフォーマンスと決定論
- [ ] テストが1秒以内に完了する
- [ ] 外部 API/データベースをモックしている
- [ ] ランダムシードを固定している
- [ ] 時間依存コードをモックしている
- [ ] 遅いテストを `@pytest.mark.slow` でマークしている

### エッジケースとカバレッジ
- [ ] 境界値をテストしている (最小、最大、±1)
- [ ] 空のコレクションと None をテストしている
- [ ] 例外条件を明示的にテストしている
- [ ] カバレッジが80%以上である
- [ ] 重要なコードパスがテストされていない箇所がない

### 設定と CI
- [ ] pyproject.toml でカスタムマーカーを登録している
- [ ] `--strict-markers` と `--strict-config` を使用している
- [ ] テストに複雑なロジックがない
- [ ] CI/CD でテストが自動実行される
- [ ] 並列実行でテストが合格する

## 参考資料

- [pytest 公式ドキュメント](https://docs.pytest.org/)
- [良い統合プラクティス](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
- [フィクスチャの使い方](https://docs.pytest.org/en/stable/how-to/fixtures.html)
