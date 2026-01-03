---
description: 'mypy を使用した Python コードの型チェックのベストプラクティス。厳密な型アノテーション、段階的な厳密化、効率的な設定管理の指針を提供します。'
applyTo: '**/*.py, **/test_*.py, **/*_test.py, **/tests/**/*.py, **/conftest.py'
---

# mypy ベストプラクティス

Python コードに型アノテーションを追加し、mypy で型チェックを行う際のベストプラクティス。strict モードを目指す段階的なアプローチを採用します。

## 基本方針

- すべての新規コードに型アノテーションを追加
- 既存コード修正時にもアノテーションを追加
- Python 3.9+ を想定（3.10+ の新しい型構文を推奨）
- strict モードを最終目標とする段階的な厳密化
- テストコードも型チェック対象に含める

## 型アノテーションの基本

### 関数の型アノテーション

すべての関数に引数と戻り値の型を明記します。

**Good Example**:
```python
def calculate_total(items: list[float], tax_rate: float) -> float:
    """合計金額を計算（税込み）"""
    subtotal = sum(items)
    return subtotal * (1 + tax_rate)

def log_message(message: str) -> None:
    """ログメッセージを出力（戻り値なし）"""
    print(f"[LOG] {message}")

def find_user(user_id: int) -> str | None:
    """ユーザーを検索（見つからない場合は None）"""
    # Python 3.10+ では | None を使用
    return None
```

**Bad Example**:
```python
# アノテーションなし
def calculate_total(items, tax_rate):
    subtotal = sum(items)
    return subtotal * (1 + tax_rate)

# 不完全なアノテーション
def find_user(user_id: int):
    return None
```

### 変数の型アノテーション

空のコレクションや型が曖昧な場合は明示的に型を指定します。

**Good Example**:
```python
# 空のコレクションは型を明示
items: list[int] = []
config: dict[str, str] = {}
unique_ids: set[int] = set()

# 複雑な型も明示
user_data: dict[str, int | str] = {"name": "Alice", "age": 30}

# タプルの型指定
point: tuple[int, int] = (10, 20)
coords: tuple[int, ...] = (1, 2, 3, 4)  # 可変長
```

**Bad Example**:
```python
# 型が推論できない
items = []  # list[Any] として扱われる可能性
config = {}  # dict[Any, Any]
```

### クラスの型アノテーション

**Good Example**:
```python
from typing import ClassVar

class User:
    # クラス変数には ClassVar を使用
    user_count: ClassVar[int] = 0
    
    def __init__(self, name: str, age: int, email: str) -> None:
        self.name = name
        self.age = age
        self.email = email
        User.user_count += 1
    
    def get_info(self) -> str:
        return f"{self.name} ({self.age})"
    
    @classmethod
    def create_guest(cls) -> "User":
        return cls(name="Guest", age=0, email="guest@example.com")
```

**Bad Example**:
```python
class User:
    user_count = 0  # ClassVar が明示されていない
    
    def __init__(self, name, age, email):  # 型アノテーションなし
        self.name = name
        self.age = age
```

## 高度な型アノテーション

### Union型とOptional

**Good Example**:
```python
# Python 3.10+ では | を使用
def process(value: int | str) -> str:
    if isinstance(value, int):
        return str(value * 2)
    return value.upper()

# None を許容する場合
def find_item(item_id: int) -> dict[str, str] | None:
    return None

# 複数の型を許容
def parse_input(data: str | int | float) -> float:
    return float(data)
```

**Bad Example**:
```python
from typing import Optional, Union

# 古い構文（Python 3.9 以下では許容）
def process(value: Union[int, str]) -> str:
    pass

# 不明瞭な Any の使用
from typing import Any
def process(value: Any) -> Any:
    pass
```

### ジェネリック型

**Good Example**:
```python
from typing import TypeVar, Generic
from collections.abc import Sequence, Callable

T = TypeVar('T')

def first(items: Sequence[T]) -> T | None:
    """シーケンスの最初の要素を返す"""
    return items[0] if items else None

def apply_twice(func: Callable[[T], T], value: T) -> T:
    """関数を2回適用"""
    return func(func(value))

# ジェネリッククラス
class Container(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value
    
    def get(self) -> T:
        return self.value
    
    def set(self, value: T) -> None:
        self.value = value
```

### Protocol（構造的部分型）

**Good Example**:
```python
from typing import Protocol

class Drawable(Protocol):
    """描画可能なオブジェクトのプロトコル"""
    def draw(self) -> None: ...
    def get_area(self) -> float: ...

class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius
    
    def draw(self) -> None:
        print("Drawing circle")
    
    def get_area(self) -> float:
        return 3.14159 * self.radius ** 2

# Protocol を使用することで継承なしでダックタイピングを型安全に実現
def render_shapes(shapes: list[Drawable]) -> None:
    for shape in shapes:
        shape.draw()
        print(f"Area: {shape.get_area()}")
```

### TypedDict

**Good Example**:
```python
from typing import TypedDict

class UserDict(TypedDict):
    """ユーザー情報の辞書型"""
    name: str
    age: int
    email: str

class PartialUserDict(TypedDict, total=False):
    """オプショナルなユーザー情報"""
    nickname: str
    phone: str

def create_user(user_data: UserDict) -> None:
    print(f"Creating user: {user_data['name']}")

def update_user(user_id: int, updates: PartialUserDict) -> None:
    """部分的な更新を適用"""
    if 'nickname' in updates:
        print(f"Updating nickname: {updates['nickname']}")
```

### 型エイリアス

複雑な型は型エイリアスで読みやすくします。

**Good Example**:
```python
from typing import TypeAlias

# 型エイリアスで可読性向上
UserId: TypeAlias = int
UserData: TypeAlias = dict[str, str | int]
JsonDict: TypeAlias = dict[str, "JsonValue"]
JsonValue: TypeAlias = str | int | float | bool | None | JsonDict | list["JsonValue"]

def get_user(user_id: UserId) -> UserData:
    return {"name": "Alice", "age": 30}

def parse_json(data: str) -> JsonDict:
    import json
    return json.loads(data)
```

**Bad Example**:
```python
# 複雑な型をそのまま使用（読みにくい）
def process_data(
    data: dict[str, list[tuple[int, str, dict[str, float]]]]
) -> list[tuple[int, dict[str, float]]]:
    pass
```

### Dataclasses

**Good Example**:
```python
from dataclasses import dataclass, field

@dataclass
class User:
    """ユーザー情報を保持するデータクラス"""
    name: str
    age: int
    email: str
    is_active: bool = True
    tags: list[str] = field(default_factory=list)
    
    def display_name(self) -> str:
        return f"{self.name} ({self.age})"

@dataclass(frozen=True)
class Point:
    """不変な座標点"""
    x: float
    y: float
    
    def distance_from_origin(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5
```

**Bad Example**:
```python
# 通常のクラスで定型コードを記述（dataclass を使うべき）
class User:
    def __init__(self, name, age, email, is_active=True):
        self.name = name
        self.age = age
        self.email = email
        self.is_active = is_active
    
    def __repr__(self):
        return f"User({self.name}, {self.age})"
```

## 型の絞り込みとキャスト

### isinstance による型の絞り込み

**Good Example**:
```python
def process_value(value: int | str | list[int]) -> int:
    if isinstance(value, int):
        # この中では value は int として扱われる
        return value * 2
    elif isinstance(value, str):
        # この中では value は str として扱われる
        return len(value)
    else:
        # この中では value は list[int] として扱われる
        return sum(value)
```

### cast の使用（慎重に）

**Good Example**:
```python
from typing import cast

def get_config() -> dict[str, str]:
    # 外部ライブラリが Any を返す場合など、やむを得ない場合のみ
    raw_config = load_config()  # type: Any
    
    # 実行時にバリデーションを行う
    if not isinstance(raw_config, dict):
        raise TypeError("Config must be a dict")
    
    # cast は型チェッカーへのヒントのみ（実行時チェックはしない）
    return cast(dict[str, str], raw_config)
```

**Bad Example**:
```python
# cast を安易に使用（実行時エラーの原因になる）
def bad_cast(value: Any) -> str:
    return cast(str, value)  # 実行時に検証していない
```

## よくあるパターン

### Callable の使用

**Good Example**:
```python
from collections.abc import Callable

def apply_operation(
    x: int,
    y: int,
    operation: Callable[[int, int], int],
) -> int:
    """2つの整数に演算を適用"""
    return operation(x, y)

def add(a: int, b: int) -> int:
    return a + b

def multiply(a: int, b: int) -> int:
    return a * b

result = apply_operation(5, 3, add)  # OK
```

### 前方参照

**Good Example**:
```python
from __future__ import annotations

class Node:
    """自己参照型のノード"""
    def __init__(self, value: int, next: Node | None = None) -> None:
        self.value = value
        self.next = next
    
    def append(self, node: Node) -> None:
        current = self
        while current.next:
            current = current.next
        current.next = node
```

### コンテキストマネージャー

**Good Example**:
```python
from typing import Self
from types import TracebackType

class DatabaseConnection:
    def __init__(self, url: str) -> None:
        self.url = url
    
    def __enter__(self) -> Self:
        # 接続を開く処理
        return self
    
    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> None:
        # 接続を閉じる処理
        pass
```

## 避けるべきアンチパターン

### 1. Any の濫用

**Bad Example**:
```python
from typing import Any

def process(data: Any) -> Any:  # 型チェックが無効化される
    return data.something()
```

**Good Example**:
```python
def process(data: dict[str, int]) -> int:
    return sum(data.values())
```

### 2. 型アノテーションの欠如

**Bad Example**:
```python
def calculate(a, b):  # 型が不明
    return a + b
```

**Good Example**:
```python
def calculate(a: int, b: int) -> int:
    return a + b
```

### 3. type: ignore の乱用

**Bad Example**:
```python
result = some_function()  # type: ignore
value = another_function()  # type: ignore
data = third_function()  # type: ignore
```

**Good Example**:
```python
# 正当な理由がある場合のみ使用し、理由を明記
result = legacy_function()  # type: ignore[no-untyped-call]  # TODO: Add types to legacy_function
```

### 4. 不適切な型の使用

**Bad Example**:
```python
# list や dict を直接型として使用（Python 3.9+）
def process(items: list) -> dict:
    pass
```

**Good Example**:
```python
# ジェネリック型を明示
def process(items: list[str]) -> dict[str, int]:
    pass
```

## 設定ファイル

### pyproject.toml での設定（推奨）

**段階1: 基本設定**
```toml
[tool.mypy]
python_version = "3.11"
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true

# サードパーティライブラリの型情報がない場合
[[tool.mypy.overrides]]
module = ["untyped_lib.*", "another_lib.*"]
ignore_missing_imports = true
```

**段階2: 中級設定**
```toml
[tool.mypy]
python_version = "3.11"
strict_equality = true
check_untyped_defs = true
disallow_subclassing_any = true
disallow_untyped_decorators = true
disallow_any_generics = true

warn_return_any = true
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true

no_implicit_reexport = true

# テストコードも含める
files = ["src/", "tests/"]
```

**段階3: strict モード（最終目標）**
```toml
[tool.mypy]
python_version = "3.11"
strict = true

files = ["src/", "tests/"]

# 必要に応じて個別に緩和
# [[tool.mypy.overrides]]
# module = "tests.*"
# disallow_untyped_defs = false
```

### モジュール単位の設定

```toml
# 新しいモジュールは厳密に
[[tool.mypy.overrides]]
module = "myapp.new_feature.*"
disallow_untyped_defs = true
disallow_incomplete_defs = true

# レガシーモジュールは緩く（段階的に厳密化）
[[tool.mypy.overrides]]
module = "myapp.legacy.*"
check_untyped_defs = true
ignore_errors = false
```

## テストコードの型アノテーション

テストコードにも型アノテーションを追加します。

**Good Example**:
```python
import pytest

def test_calculate_total() -> None:
    """合計計算のテスト"""
    result: float = calculate_total([1.0, 2.0, 3.0], 0.1)
    assert result == 6.6

@pytest.fixture
def sample_user() -> User:
    """テスト用のユーザーフィクスチャ"""
    return User(name="Test", age=25, email="test@example.com")

def test_user_creation(sample_user: User) -> None:
    """ユーザー作成のテスト"""
    assert sample_user.name == "Test"
    assert sample_user.age == 25

@pytest.mark.parametrize(
    "input_value,expected",
    [
        (1, "1"),
        (2, "2"),
        (3, "3"),
    ],
)
def test_convert_to_string(input_value: int, expected: str) -> None:
    """整数から文字列への変換テスト"""
    assert str(input_value) == expected
```

## 検証方法

### ローカルでの実行

```bash
# 基本的な実行
mypy src/

# テストコードも含める
mypy src/ tests/

# 特定のファイルのみ
mypy src/main.py

# 並列実行（大規模プロジェクト）
mypy -j 4 src/

# mypy daemon を使用（高速化）
dmypy run -- src/
```

### pre-commit での自動チェック

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests, types-PyYAML]
        args: [--strict]
        files: ^(src/|tests/)
```

### CI/CD での統合

```yaml
# GitHub Actions での例
name: Type Check

on: [push, pull_request]

jobs:
  mypy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install mypy types-requests
      
      - name: Run mypy
        run: mypy src/ tests/
```

## デバッグツール

### reveal_type の使用

```python
def example(x: int | str) -> None:
    reveal_type(x)  # Revealed type is "int | str"
    
    if isinstance(x, int):
        reveal_type(x)  # Revealed type is "int"
        print(x * 2)
```

### エラーコードの表示

```bash
# エラーコードを表示して原因を特定
mypy --show-error-codes src/
```

## まとめ

- すべての関数・メソッドに型アノテーションを追加
- 空のコレクションや曖昧な型は明示的に指定
- Python 3.10+ の新しい型構文（`|`）を使用
- Protocol を活用してダックタイピングを型安全に
- Dataclasses で定型コードを削減
- Any の使用を最小限に抑える
- テストコードも型チェック対象に含める
- 段階的に strict モードを目指す
- CI/CD で型チェックを自動化

型アノテーションは開発者の意図を明確にし、バグを早期に発見するための強力なツールです。一貫性のある型アノテーションを維持することで、コードの品質と保守性が向上します。
