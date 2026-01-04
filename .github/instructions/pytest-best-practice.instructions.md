---
description: 'pytest best practices for high-quality test code'
applyTo: '**/test_*.py, **/*_test.py, **/tests/**/*.py, **/conftest.py'
---

# pytest Best Practices

Guidelines for creating high-quality automated test code with pytest.

## Purpose and Scope

Apply pytest best practices to Python test files to ensure maintainability, reliability, and fast execution.

## Project Structure

### Use src Layout
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

**Rationale**: Prevents accidental imports of uninstalled packages during testing.

### Configure pyproject.toml
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

**Required markers**: Register all custom markers to prevent typos.

### Follow Naming Conventions
- Test files: `test_*.py` or `*_test.py`
- Test functions: Start with `test_`
- Test classes: Start with `Test` (no `__init__` method)
- Test methods: Start with `test_`

## Test Structure

### Follow Arrange-Act-Assert Pattern
```python
def test_user_registration(database, email_service):
    # Arrange: Prepare test data and mocks
    user_data = {"name": "test_user", "email": "test@example.com"}

    # Act: Execute the function under test (only once)
    result = register_user(user_data, database)

    # Assert: Verify results
    assert result.success is True
    assert database.get_user("test_user") is not None
```

**Rules**:
- Keep Act section to single function call
- Allow multiple assertions if they verify the same Act result
- Delegate cleanup to fixtures

## Fixtures

### Use yield Fixtures for Setup/Teardown
```python
@pytest.fixture
def database_connection():
    conn = create_connection("test.db")
    conn.initialize()
    yield conn
    conn.close()  # Always executed
```

**Rationale**: yield ensures cleanup runs even if test fails.

### Choose Appropriate Scope
- `function` (default): New instance per test (safest)
- `class`: One instance per test class
- `module`: One instance per module
- `session`: One instance per test session

**Rule**: Use broader scope only for stateless resources or high-cost operations (DB, Docker).

### Share Fixtures via conftest.py
Place common fixtures in `tests/conftest.py` to make them available to all tests.

```python
# tests/conftest.py
@pytest.fixture
def api_client():
    return APIClient(base_url="http://test")
```

### Parameterize Fixtures
```python
@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def database(request):
    db = create_database(request.param)
    yield db
    db.close()
```

**Effect**: Test runs once for each parameter value.

## Assertions

### Use Simple assert Statements
```python
def test_addition():
    assert 2 + 2 == 4
    assert result > 0
```

### Compare Floating-Point Numbers with pytest.approx
```python
assert (0.1 + 0.2) == pytest.approx(0.3)
assert 1.0001 == pytest.approx(1.0, abs=0.001)
```

**Rationale**: Direct equality fails due to floating-point precision errors.

### Test Exceptions with pytest.raises
```python
def test_exception():
    with pytest.raises(ValueError, match=r"invalid literal.*"):
        int("invalid")
```

**Never use try-except**: pytest.raises provides better error messages.

### Test Warnings with pytest.warns
```python
def test_warning():
    with pytest.warns(UserWarning, match="deprecated"):
        warnings.warn("deprecated feature", UserWarning)
```

## Parameterization

### Parameterize Tests to Reduce Duplication
```python
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 3),
    (3, 4),
])
def test_increment(input, expected):
    assert increment(input) == expected
```

### Use ids for Readable Test Names
```python
@pytest.mark.parametrize("input,expected", [
    ({"name": "Alice"}, True),
    ({}, False),
], ids=["valid-user", "empty-dict"])
def test_validate_user(input, expected):
    assert validate_user(input) == expected
```

### Combine Multiple Decorators for Cartesian Product
```python
@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_combination(x, y):
    # Runs 4 tests: (0,2), (0,3), (1,2), (1,3)
    assert x + y >= 2
```

## Markers

### Register Custom Markers in pyproject.toml
```toml
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
]
```

**Rule**: Use `--strict-markers` to prevent typos.

### Apply Markers to Tests
```python
@pytest.mark.unit
def test_pure_function():
    assert add(1, 2) == 3

@pytest.mark.slow
@pytest.mark.integration
def test_full_workflow():
    pass
```

### Run Tests by Marker
```bash
pytest -m unit              # Run only unit tests
pytest -m "not slow"        # Exclude slow tests
pytest -m "integration or slow"  # Run integration or slow
```

## Skipping and XFail

### Skip Tests Conditionally
```python
@pytest.mark.skipif(sys.version_info < (3, 10), reason="requires python3.10+")
def test_new_feature():
    pass

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_feature():
    pass
```

### Mark Expected Failures with xfail
```python
@pytest.mark.xfail(reason="known bug #123")
def test_known_issue():
    assert buggy_function() == expected_value

@pytest.mark.xfail(strict=True, reason="must fail")
def test_must_fail():
    assert False  # Error if test passes unexpectedly
```

### Use importorskip for Optional Dependencies
```python
numpy = pytest.importorskip("numpy", minversion="1.20")

def test_with_numpy():
    arr = numpy.array([1, 2, 3])
    assert len(arr) == 3
```

## Temporary Files and Directories

### Use tmp_path Fixture for Temporary Files
```python
def test_write_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test content", encoding="utf-8")
    assert file_path.read_text(encoding="utf-8") == "test content"
```

**Rationale**: tmp_path provides isolated Path objects and automatic cleanup.

### Use tmp_path_factory for Session-Scoped Temp Files
```python
@pytest.fixture(scope="session")
def shared_data_file(tmp_path_factory):
    data_dir = tmp_path_factory.mktemp("data")
    file_path = data_dir / "shared.json"
    file_path.write_text(json.dumps(data), encoding="utf-8")
    return file_path
```

## Mocking and Monkeypatching

### Use monkeypatch Fixture for Mocking
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

**Rationale**: monkeypatch provides automatic cleanup after test execution.

### Create Reusable Mock Fixtures
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

## Test Independence

### Ensure Tests Run in Any Order
Each test must be completely independent and not rely on other tests.

**Never share mutable state**:
```python
# Bad: Shared global state
shared_state = {"count": 0}

def test_increment():
    shared_state["count"] += 1
    assert shared_state["count"] == 1  # Fails if run after test_value

# Good: Independent state via fixtures
@pytest.fixture
def isolated_state():
    return {"count": 0}

def test_increment(isolated_state):
    isolated_state["count"] += 1
    assert isolated_state["count"] == 1
```

**Rationale**: Test order dependency causes flaky tests and debugging difficulties.

### Verify Independence with Parallel Execution
```bash
pytest -n auto  # Uses pytest-xdist
```

If tests fail with parallel execution, they have hidden dependencies.

## Anti-Patterns to Avoid

### Avoid Multiple Acts in Single Test
```python
# Bad
def test_multiple_operations():
    result1 = function1()
    assert result1 == expected1
    result2 = function2()
    assert result2 == expected2

# Good: Split into separate tests
def test_function1():
    assert function1() == expected1

def test_function2():
    assert function2() == expected2
```

**Rationale**: Single test should verify single behavior.

### Avoid try-except in Tests
```python
# Bad
def test_exception():
    try:
        risky_function()
        assert False
    except ValueError:
        pass

# Good
def test_exception():
    with pytest.raises(ValueError):
        risky_function()
```

### Avoid Complex Conditionals
```python
# Bad
def test_with_condition():
    if condition:
        assert result == value1
    else:
        assert result == value2

# Good: Use parametrization
@pytest.mark.parametrize("condition,expected", [
    (True, value1),
    (False, value2),
])
def test_with_condition(condition, expected):
    assert function(condition) == expected
```

### Never Hardcode Paths
```python
# Bad
def test_file():
    with open("/tmp/test.txt", "w") as f:
        f.write("test")

# Good
def test_file(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("test", encoding="utf-8")
```

## Performance and Speed

### Keep Tests Fast (< 1 second)
- Mock external services (APIs, databases)
- Use in-memory SQLite for database tests
- Share expensive resources with broader fixture scopes

### Mock External Dependencies
```python
# Good: Mock API calls
def test_api_call(monkeypatch):
    def mock_get(*args, **kwargs):
        return MockResponse(status_code=200)
    monkeypatch.setattr(requests, "get", mock_get)
    result = api_client.fetch_data()
    assert result["status"] == "ok"

# Bad: Real API calls
def test_api_call():
    response = requests.get("https://api.example.com/data")
    assert response.status_code == 200
```

### Mark Slow Tests
```python
@pytest.mark.slow
def test_heavy_computation():
    result = perform_complex_calculation()
    assert result == expected
```

**Run without slow tests**: `pytest -m "not slow"`

### Run Tests in Parallel
```bash
pip install pytest-xdist
pytest -n auto  # Uses all CPU cores
```

### Identify Slow Tests
```bash
pytest --durations=10
```

## Deterministic Tests

### Fix Random Seeds
```python
@pytest.fixture(autouse=True)
def reset_random_seed():
    random.seed(42)
    yield
```

**Rationale**: Random values cause flaky tests.

### Mock Time-Dependent Code
```python
from freezegun import freeze_time

@freeze_time("2024-01-01 12:00:00")
def test_with_fixed_time():
    now = datetime.now(timezone.utc)
    assert now.year == 2024
```

### Fix UUIDs
```python
@pytest.fixture
def fixed_uuid(monkeypatch):
    fixed_id = UUID('12345678-1234-5678-1234-567812345678')
    monkeypatch.setattr('uuid.uuid4', lambda: fixed_id)
    return fixed_id
```

## Boundary and Edge Cases

### Test Boundary Values
```python
@pytest.mark.parametrize("value,expected", [
    (0, "zero"),        # Minimum
    (1, "positive"),    # Minimum + 1
    (-1, "negative"),   # Minimum - 1
    (100, "maximum"),   # Maximum
    (101, "error"),     # Maximum + 1 (out of range)
])
def test_boundary_values(value, expected):
    assert classify_number(value) == expected
```

**Rationale**: Bugs often occur at boundaries.

### Test Empty Collections
```python
def test_empty_collections():
    assert process_items([]) == []
    assert format_text("") == ""
    assert transform_data({}) == {}
    assert safe_process(None) is None
```

### Test String Edge Cases
```python
@pytest.mark.parametrize("input_str", [
    "",             # Empty
    " ",            # Whitespace only
    "a",            # Single char
    "あ",           # Multibyte
    "emoji 😀",     # Emoji
    "a" * 1000,    # Very long
])
def test_string_edge_cases(input_str):
    result = process_string(input_str)
    assert isinstance(result, str)
```

### Test Numeric Special Cases
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

## Test Coverage

### Configure Coverage in pyproject.toml
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

### Exclude Debug Code
```python
def debug_helper():  # pragma: no cover
    print("Debug info")

if __name__ == "__main__":  # pragma: no cover
    main()
```

### Run Coverage
```bash
pytest --cov=src --cov-report=html
pytest --cov=src --cov-report=term-missing
pytest --cov=src --cov-fail-under=80
```

**Target**: 80%+ for general applications, 90%+ for critical systems.

**Rule**: Do not blindly pursue 100% coverage; focus on quality.

## CI/CD Integration

### GitHub Actions Example
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

### Stage Tests by Speed
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

**Best practices**:
- Run tests on all pushes/PRs
- Test multiple Python versions
- Cache dependencies for speed
- Run tests in parallel with `-n auto`
- Require tests to pass before merge

## Checklist

Before committing test code:

### Basic
- [ ] Follow naming conventions (test_*.py, test_*)
- [ ] Use Arrange-Act-Assert pattern
- [ ] Use fixtures instead of setup/teardown
- [ ] Tests are independent (order-agnostic)
- [ ] Use parametrization to reduce duplication
- [ ] Apply appropriate markers

### Assertions and Mocks
- [ ] Use `pytest.raises()` for exceptions
- [ ] Use `tmp_path` for temporary files
- [ ] Use `monkeypatch` for mocking
- [ ] Use `pytest.approx()` for floats

### Performance and Determinism
- [ ] Tests complete in < 1 second
- [ ] Mock external APIs/databases
- [ ] Fix random seeds
- [ ] Mock time-dependent code
- [ ] Mark slow tests with `@pytest.mark.slow`

### Edge Cases and Coverage
- [ ] Test boundary values (min, max, ±1)
- [ ] Test empty collections and None
- [ ] Test exception conditions explicitly
- [ ] Coverage is 80%+
- [ ] No untested critical code paths

### Configuration and CI
- [ ] Custom markers registered in pyproject.toml
- [ ] Use `--strict-markers` and `--strict-config`
- [ ] No complex logic in tests
- [ ] Tests run automatically in CI/CD
- [ ] Tests pass with parallel execution

## References

- [pytest Official Documentation](https://docs.pytest.org/)
- [Good Integration Practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
- [How to use fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
