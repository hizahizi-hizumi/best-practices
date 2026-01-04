````instructions
---
description: 'mypy による Python コードの型チェックのベストプラクティス。厳密な型アノテーション、段階的な厳密化、効率的な設定管理のガイドラインを提供する。'
applyTo: '**/*.py, **/test_*.py, **/*_test.py, **/tests/**/*.py, **/conftest.py'
---

# mypy ベストプラクティス

## 基本原則

- すべての関数とメソッドに型アノテーションを追加する
- Python 3.10+ の型構文 (`|`) を使用する
- 段階的に strict モードを目指す
- テストコードも型チェックの対象に含める
- `Any` の使用を最小限に抑える

## 関数の型アノテーション

- すべての関数に引数と戻り値の型を指定する
- None を返す関数には明示的に `-> None` を記述する
- None を許容する場合は `| None` を使用する

**推奨**:
```python
def calculate_total(items: list[float], tax_rate: float) -> float:
    return sum(items) * (1 + tax_rate)

def find_user(user_id: int) -> str | None:
    return None  # | None を使用する
```

**非推奨**:
```python
def calculate_total(items, tax_rate):  # 型アノテーションなし
    return sum(items) * (1 + tax_rate)
```

## 変数の型アノテーション

- 空のコレクションには型を明示的に指定する
- 曖昧な変数には型アノテーションを追加する
- 可変長タプルには `tuple[T, ...]` を使用する

**推奨**:
```python
items: list[int] = []
config: dict[str, str] = {}
point: tuple[int, int] = (10, 20)
```

**非推奨**:
```python
items = []  # list[Any] として扱われる
```

## クラスの型アノテーション

- クラス変数には `ClassVar` を使用する
- `__init__` の戻り値の型には `-> None` を指定する
- 必要に応じて文字列リテラルで前方参照を行う

**推奨**:
```python
from typing import ClassVar

class User:
    user_count: ClassVar[int] = 0
    
    def __init__(self, name: str, age: int) -> None:
        self.name = name
        self.age = age
    
    @classmethod
    def create_guest(cls) -> "User":
        return cls(name="Guest", age=0)
```

## Union 型と Optional

- `Union[X, Y]` の代わりに `X | Y` を使用する (Python 3.10+)
- `Optional[X]` の代わりに `X | None` を使用する
- `Any` の使用を避ける

**推奨**:
```python
def process(value: int | str) -> str:
    return str(value)

def find_item(item_id: int) -> dict[str, str] | None:
    return None
```

**非推奨**:
```python
from typing import Union, Any

def process(value: Union[int, str]) -> str:  # 古い構文
    pass

def process(value: Any) -> Any:  # Any の過剰使用
    pass
```

## ジェネリック型

- `TypeVar` で型変数を定義する
- `Generic[T]` でジェネリッククラスを作成する
- `collections.abc` から `Sequence`, `Callable` をインポートする

**推奨**:
```python
from typing import TypeVar, Generic
from collections.abc import Sequence

T = TypeVar('T')

def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None

class Container(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value
    
    def get(self) -> T:
        return self.value
```

## Protocol (構造的部分型)

- 継承なしで型安全なダックタイピングを実現する
- メソッドシグネチャのみを定義する
- インターフェースとして使用する

**推奨**:
```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...
    def get_area(self) -> float: ...

def render_shapes(shapes: list[Drawable]) -> None:
    for shape in shapes:
        shape.draw()
```

## TypedDict

- 辞書の型を厳密に定義する
- オプションキーには `total=False` を使用する
- 構造化された辞書データに使用する

**推奨**:
```python
from typing import TypedDict

class UserDict(TypedDict):
    name: str
    age: int

class PartialUserDict(TypedDict, total=False):
    nickname: str
    phone: str

def create_user(user_data: UserDict) -> None:
    print(user_data['name'])
```

## 型エイリアス

- 複雑な型に `TypeAlias` で名前を付ける
- 再利用可能な型定義に使用する
- 可読性を向上させる

**推奨**:
```python
from typing import TypeAlias

UserId: TypeAlias = int
UserData: TypeAlias = dict[str, str | int]
JsonValue: TypeAlias = str | int | float | bool | None

def get_user(user_id: UserId) -> UserData:
    return {"name": "Alice", "age": 30}
```

**非推奨**:
```python
def process(data: dict[str, list[tuple[int, str]]]) -> list[tuple[int, str]]:
    pass  # 複雑な型を直接使用している
```

## Dataclass

- `@dataclass` でボイラープレートコードを削減する
- ミュータブルなデフォルト値には `field(default_factory=...)` を使用する
- イミュータブルなオブジェクトには `frozen=True` を指定する

**推奨**:
```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    is_active: bool = True
    tags: list[str] = field(default_factory=list)

@dataclass(frozen=True)
class Point:
    x: float
    y: float
```

## 型の絞り込みとキャスト

- `isinstance` で型を絞り込む
- `cast` はランタイム検証後にのみ使用する
- `cast` は型チェッカーへのヒントのみである (ランタイムチェックなし)

**推奨**:
```python
def process_value(value: int | str) -> int:
    if isinstance(value, int):
        return value * 2  # value は int である
    return len(value)  # value は str である

# ランタイム検証後の cast
from typing import cast

def get_config() -> dict[str, str]:
    raw = load_config()
    if not isinstance(raw, dict):
        raise TypeError("Config must be a dict")
    return cast(dict[str, str], raw)
```

## 一般的なパターン

### Callable
- `collections.abc.Callable` を使用する
- 形式: `Callable[[arg_types, ...], return_type]`

```python
from collections.abc import Callable

def apply(x: int, y: int, op: Callable[[int, int], int]) -> int:
    return op(x, y)
```

### 前方参照
- `from __future__ import annotations` を使用する
- 自己参照型に必要である

```python
from __future__ import annotations

class Node:
    def __init__(self, value: int, next: Node | None = None) -> None:
        self.value = value
        self.next = next
```

### コンテキストマネージャー
- `__enter__` の戻り値は `Self`
- `__exit__` の引数は `BaseException | None`

```python
from typing import Self
from types import TracebackType

class Connection:
    def __enter__(self) -> Self:
        return self
    
    def __exit__(self, exc_type: type[BaseException] | None,
                 exc_val: BaseException | None,
                 exc_tb: TracebackType | None) -> None:
        pass
```

## アンチパターン

### `Any` の過剰使用
- `Any` は型チェックを無効化する
- 具体的な型を指定する

**非推奨**:
```python
def process(data: Any) -> Any:
    return data.something()
```

### 型アノテーションの欠落
- すべての関数に型を指定する

**非推奨**:
```python
def calculate(a, b):
    return a + b
```

### `type: ignore` の過剰使用
- 正当な理由がある場合のみ使用する
- エラーコードと理由を明記する

**推奨**:
```python
result = legacy_function()  # type: ignore[no-untyped-call]  # TODO: Add types
```

### ジェネリック型パラメータの省略
- `list`, `dict` の型パラメータを指定する

**非推奨**:
```python
def process(items: list) -> dict:
    pass
```

## pyproject.toml 設定

### 段階的な厳密化

- 基本的な設定から開始する
- 段階的に strict モードを目指す
- モジュールごとに厳密度を調整する

**ステージ 1: 基本**
```toml
[tool.mypy]
python_version = "3.11"
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true

[[tool.mypy.overrides]]
module = ["untyped_lib.*"]
ignore_missing_imports = true
```

**ステージ 2: 中間**
```toml
[tool.mypy]
python_version = "3.11"
strict_equality = true
check_untyped_defs = true
disallow_any_generics = true
warn_return_any = true
no_implicit_reexport = true
files = ["src/", "tests/"]
```

**ステージ 3: 厳密モード**
```toml
[tool.mypy]
python_version = "3.11"
strict = true
files = ["src/", "tests/"]
```

## テストコードの型アノテーション

- すべてのテスト関数に `-> None` を指定する
- フィクスチャの戻り値の型をアノテーションする
- パラメータ化されたテストで型を指定する

**推奨**:
```python
import pytest

def test_calculate() -> None:
    result: float = calculate_total([1.0, 2.0], 0.1)
    assert result == 3.3

@pytest.fixture
def sample_user() -> User:
    return User(name="Test", age=25)

def test_user_creation(sample_user: User) -> None:
    assert sample_user.name == "Test"
```

## 検証方法

### ローカル実行
```bash
mypy src/
mypy src/ tests/  # テストも含める
mypy -j 4 src/  # 並列実行
dmypy run -- src/  # デーモンで高速化
```

### pre-commit 統合
```yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
        args: [--strict]
```

### CI/CD 統合
```yaml
jobs:
  mypy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install mypy
      - run: mypy src/ tests/
```

## デバッグ

### reveal_type
- 型推論を確認するために使用する

```python
def example(x: int | str) -> None:
    reveal_type(x)  # "int | str"
    if isinstance(x, int):
        reveal_type(x)  # "int"
```

### エラーコードの表示
```bash
mypy --show-error-codes src/
```

## チェックリスト

- [ ] すべての関数に型アノテーションを追加する
- [ ] 空のコレクションを明示的に型付けする
- [ ] `X | Y` 構文を使用する (Python 3.10+)
- [ ] Protocol でダックタイピングを型安全にする
- [ ] Dataclass でボイラープレートを削減する
- [ ] `Any` の使用を最小限に抑える
- [ ] テストコードを型チェックする
- [ ] pyproject.toml で段階的に strict モードを有効にする
- [ ] CI/CD で型チェックを自動化する

````
