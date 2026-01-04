---
description: 'Best practices for type checking Python code with mypy. Provides guidelines for strict type annotations, gradual strictness, and efficient configuration management.'
applyTo: '**/*.py, **/test_*.py, **/*_test.py, **/tests/**/*.py, **/conftest.py'
---

# mypy Best Practices

## Basic Principles

- Add type annotations to all functions and methods
- Use Python 3.10+ type syntax (`|`)
- Aim for strict mode gradually
- Include test code in type checking
- Minimize the use of `Any`

## Function Type Annotations

- Specify argument and return types for all functions
- Explicitly mark functions returning None with `-> None`
- Use `| None` when allowing None

**Recommended**:
```python
def calculate_total(items: list[float], tax_rate: float) -> float:
    return sum(items) * (1 + tax_rate)

def find_user(user_id: int) -> str | None:
    return None  # Use | None
```

**Not Recommended**:
```python
def calculate_total(items, tax_rate):  # No type annotations
    return sum(items) * (1 + tax_rate)
```

## Variable Type Annotations

- Explicitly annotate empty collections with types
- Add type annotations to ambiguous variables
- Use `tuple[T, ...]` for variable-length tuples

**Recommended**:
```python
items: list[int] = []
config: dict[str, str] = {}
point: tuple[int, int] = (10, 20)
```

**Not Recommended**:
```python
items = []  # Treated as list[Any]
```

## Class Type Annotations

- Use `ClassVar` for class variables
- Specify `-> None` as the return type for `__init__`
- Use string literals for forward references when necessary

**Recommended**:
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

## Union Types and Optional

- Use `X | Y` instead of `Union[X, Y]` (Python 3.10+)
- Use `X | None` instead of `Optional[X]`
- Avoid using `Any`

**Recommended**:
```python
def process(value: int | str) -> str:
    return str(value)

def find_item(item_id: int) -> dict[str, str] | None:
    return None
```

**Not Recommended**:
```python
from typing import Union, Any

def process(value: Union[int, str]) -> str:  # Old syntax
    pass

def process(value: Any) -> Any:  # Overuse of Any
    pass
```

## Generic Types

- Define type variables with `TypeVar`
- Create generic classes with `Generic[T]`
- Import `Sequence`, `Callable` from `collections.abc`

**Recommended**:
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

## Protocol (Structural Subtyping)

- Achieve type-safe duck typing without inheritance
- Define only method signatures
- Use as interfaces

**Recommended**:
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

- Strictly define dictionary types
- Use `total=False` to specify optional keys
- Use for structured dictionary data

**Recommended**:
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

## Type Aliases

- Name complex types with `TypeAlias`
- Use for reusable type definitions
- Improve readability

**Recommended**:
```python
from typing import TypeAlias

UserId: TypeAlias = int
UserData: TypeAlias = dict[str, str | int]
JsonValue: TypeAlias = str | int | float | bool | None

def get_user(user_id: UserId) -> UserData:
    return {"name": "Alice", "age": 30}
```

**Not Recommended**:
```python
def process(data: dict[str, list[tuple[int, str]]]) -> list[tuple[int, str]]:
    pass  # Using complex types directly
```

## Dataclasses

- Reduce boilerplate code with `@dataclass`
- Use `field(default_factory=...)` for mutable default values
- Specify `frozen=True` for immutable objects

**Recommended**:
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

## Type Narrowing and Casting

- Narrow types with `isinstance`
- Use `cast` only after runtime validation
- `cast` is only a hint to type checkers (no runtime check)

**Recommended**:
```python
def process_value(value: int | str) -> int:
    if isinstance(value, int):
        return value * 2  # value is int
    return len(value)  # value is str

# cast after runtime validation
from typing import cast

def get_config() -> dict[str, str]:
    raw = load_config()
    if not isinstance(raw, dict):
        raise TypeError("Config must be a dict")
    return cast(dict[str, str], raw)
```

## Common Patterns

### Callable
- Use `collections.abc.Callable`
- Format: `Callable[[arg_types, ...], return_type]`

```python
from collections.abc import Callable

def apply(x: int, y: int, op: Callable[[int, int], int]) -> int:
    return op(x, y)
```

### Forward References
- Use `from __future__ import annotations`
- Required for self-referencing types

```python
from __future__ import annotations

class Node:
    def __init__(self, value: int, next: Node | None = None) -> None:
        self.value = value
        self.next = next
```

### Context Managers
- `__enter__` return value is `Self`
- `__exit__` arguments are `BaseException | None`

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

## Anti-Patterns

### Overuse of `Any`
- `Any` disables type checking
- Specify concrete types

**Not Recommended**:
```python
def process(data: Any) -> Any:
    return data.something()
```

### Missing Type Annotations
- Specify types for all functions

**Not Recommended**:
```python
def calculate(a, b):
    return a + b
```

### Overuse of `type: ignore`
- Use only when there's a valid reason
- Specify error codes and reasons

**Recommended**:
```python
result = legacy_function()  # type: ignore[no-untyped-call]  # TODO: Add types
```

### Omitting Generic Type Parameters
- Specify type parameters for `list`, `dict`

**Not Recommended**:
```python
def process(items: list) -> dict:
    pass
```

## pyproject.toml Configuration

### Gradual Strictness

- Start with basic configuration
- Gradually aim for strict mode
- Adjust strictness per module

**Stage 1: Basic**
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

**Stage 2: Intermediate**
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

**Stage 3: Strict Mode**
```toml
[tool.mypy]
python_version = "3.11"
strict = true
files = ["src/", "tests/"]
```

## Type Annotations for Test Code

- Specify `-> None` for all test functions
- Annotate fixture return types
- Specify types in parameterized tests

**Recommended**:
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

## Validation Methods

### Local Execution
```bash
mypy src/
mypy src/ tests/  # Include tests
mypy -j 4 src/  # Parallel execution
dmypy run -- src/  # Speed up with daemon
```

### pre-commit Integration
```yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
        args: [--strict]
```

### CI/CD Integration
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

## Debugging

### reveal_type
- Use to check type inference

```python
def example(x: int | str) -> None:
    reveal_type(x)  # "int | str"
    if isinstance(x, int):
        reveal_type(x)  # "int"
```

### Display Error Codes
```bash
mypy --show-error-codes src/
```

## Checklist

- [ ] Add type annotations to all functions
- [ ] Explicitly type empty collections
- [ ] Use `X | Y` syntax (Python 3.10+)
- [ ] Make duck typing type-safe with Protocol
- [ ] Reduce boilerplate with Dataclasses
- [ ] Minimize use of `Any`
- [ ] Type check test code
- [ ] Gradually enable strict mode in pyproject.toml
- [ ] Automate type checking in CI/CD
