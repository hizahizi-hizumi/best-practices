# mypy ベストプラクティス

mypy を使用したコード品質管理のベストプラクティスガイド

最終更新: 2026-01-03

---

## 目次

1. [導入戦略](#導入戦略)
2. [設定ファイル](#設定ファイル)
3. [型アノテーション](#型アノテーション)
4. [CI/CD統合](#cicd統合)
5. [よくある問題と解決方法](#よくある問題と解決方法)
6. [段階的な厳密化](#段階的な厳密化)
7. [パフォーマンス最適化](#パフォーマンス最適化)
8. [チーム開発での運用](#チーム開発での運用)

---

## 導入戦略

### 既存コードベースへの導入

#### 小さく始める

- **5,000〜50,000行程度のサブセットから開始**する
- 1〜2日で mypy が成功するようにする
- 早期に成果を得ることで継続的な改善につなげる

```bash
# 特定のモジュールだけをチェック
mypy src/core src/models
```

#### 段階的なアプローチ

1. **エラーの少ないモジュールから開始**
   - ユーティリティ関数やヘルパーモジュール
   - 新しく書かれたコード

2. **広く使われているモジュールを優先**
   - モデルクラス
   - 基底クラス
   - 共通ユーティリティ

3. **エラーを抑制しながら進める**
   ```python
   # type: ignore コメントで一時的にエラーを抑制
   import legacy_module  # type: ignore
   ```

### アノテーションの追加方針

#### 新規コードには必ず型アノテーションを追加

```python
# Good
def calculate_total(items: list[float], tax_rate: float) -> float:
    subtotal = sum(items)
    return subtotal * (1 + tax_rate)

# Bad - アノテーションなし
def calculate_total(items, tax_rate):
    subtotal = sum(items)
    return subtotal * (1 + tax_rate)
```

#### 既存コードの修正時にアノテーションを追加

- コードを変更する際に、その関数・クラスに型アノテーションを追加
- 段階的にカバレッジを向上させる

#### 自動アノテーションツールの活用

- **MonkeyType**: 実行時の型情報を収集してアノテーションを生成
- **autotyping**: 静的解析ベースの自動アノテーション
- **PyAnnotate**: Facebook製の型収集ツール

```bash
# MonkeyType の使用例
monkeytype run my_script.py
monkeytype apply my_module
```

---

## 設定ファイル

### 設定ファイルの選択

mypy は以下の設定ファイルを優先順位順に探索します:

1. `mypy.ini`
2. `.mypy.ini`
3. `pyproject.toml` (推奨)
4. `setup.cfg`

### 推奨設定（初期段階）

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true

# エラーを特定のモジュールで無視
[[tool.mypy.overrides]]
module = "legacy_module.*"
ignore_errors = true

[[tool.mypy.overrides]]
module = "third_party_lib.*"
ignore_missing_imports = true
```

### 推奨設定（中級段階）

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
```

### 推奨設定（strict モード）

```toml
[tool.mypy]
python_version = "3.11"
strict = true

# 必要に応じて個別に緩和
# warn_return_any = false
```

### モジュール単位の設定

```toml
# 新しいモジュールは厳密に
[[tool.mypy.overrides]]
module = "myapp.new_feature.*"
disallow_untyped_defs = true
disallow_incomplete_defs = true

# レガシーモジュールは緩く
[[tool.mypy.overrides]]
module = "myapp.legacy.*"
check_untyped_defs = true

# 型情報のないサードパーティライブラリ
[[tool.mypy.overrides]]
module = [
    "untyped_lib.*",
    "another_lib.*",
]
ignore_missing_imports = true
```

---

## 型アノテーション

### 基本的な型アノテーション

#### 変数の型アノテーション

```python
# 基本型
age: int = 25
name: str = "Alice"
is_active: bool = True
price: float = 99.99

# コレクション型
numbers: list[int] = [1, 2, 3]
names: set[str] = {"Alice", "Bob"}
config: dict[str, int] = {"timeout": 30}

# タプル（固定長）
point: tuple[int, int] = (10, 20)
# タプル（可変長）
coords: tuple[int, ...] = (1, 2, 3, 4)
```

#### 関数の型アノテーション

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

def process_items(items: list[int], threshold: int = 10) -> list[int]:
    return [item for item in items if item > threshold]

# 戻り値なし
def log_message(message: str) -> None:
    print(message)
```

#### クラスの型アノテーション

```python
from typing import ClassVar

class User:
    # クラス変数
    user_count: ClassVar[int] = 0

    def __init__(self, name: str, age: int) -> None:
        self.name = name  # 型は自動推論される
        self.age = age
        User.user_count += 1

    def get_info(self) -> str:
        return f"{self.name} ({self.age})"
```

### 高度な型アノテーション

#### Union型とOptional型

```python
from typing import Union, Optional

# Python 3.10+
def process(value: int | str) -> str:
    if isinstance(value, int):
        return str(value)
    return value

# Optional は None を許容
def find_user(user_id: int) -> Optional[str]:
    # None を返す可能性がある
    return None

# Python 3.10+ では | None を使用
def find_user_new(user_id: int) -> str | None:
    return None
```

#### ジェネリック型

```python
from typing import TypeVar, Generic
from collections.abc import Sequence

T = TypeVar('T')

def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None

# ジェネリッククラス
class Container(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value

    def get(self) -> T:
        return self.value
```

#### Protocol（構造的部分型）

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

# Protocol を使用することで、
# 継承なしでダックタイピングを型安全に実現
def render(shape: Drawable) -> None:
    shape.draw()

render(Circle())  # OK
render(Square())  # OK
```

#### TypedDict

```python
from typing import TypedDict

class UserDict(TypedDict):
    name: str
    age: int
    email: str

# 部分的にオプション
class PartialUserDict(TypedDict, total=False):
    nickname: str
    phone: str

def create_user(user_data: UserDict) -> None:
    print(f"Creating user: {user_data['name']}")
```

#### Callable

```python
from collections.abc import Callable

def apply_operation(
    x: int,
    y: int,
    operation: Callable[[int, int], int]
) -> int:
    return operation(x, y)

def add(a: int, b: int) -> int:
    return a + b

result = apply_operation(5, 3, add)  # OK
```

### 型アノテーションのベストプラクティス

#### 1. 明示的な型アノテーションを使用

```python
# Good - 明示的
items: list[int] = []

# Bad - 型推論に依存（空のコレクションの場合）
items = []  # list[Any] と推論される可能性
```

#### 2. 前方参照の処理

```python
from __future__ import annotations

class Node:
    def __init__(self, value: int, next: Node | None = None) -> None:
        self.value = value
        self.next = next
```

#### 3. Any型の使用を最小限に

```python
from typing import Any

# Bad - Any は型チェックを無効化
def process(data: Any) -> Any:
    return data.something()

# Good - 具体的な型を使用
def process(data: dict[str, int]) -> int:
    return sum(data.values())
```

#### 4. 型エイリアスの活用

```python
from typing import TypeAlias

# 複雑な型を簡潔に
UserId: TypeAlias = int
UserData: TypeAlias = dict[str, str | int]

def get_user(user_id: UserId) -> UserData:
    return {"name": "Alice", "age": 30}
```

#### 5. typing.cast の適切な使用

```python
from typing import cast

# mypy が型を正しく推論できない場合のみ使用
def process_data(data: list[object]) -> list[str]:
    # isinstance チェック後、型が確定している
    result: list[str] = []
    for item in data:
        if isinstance(item, str):
            result.append(item)
    return result

# 複雑な型変換で mypy が理解できない場合
def get_config() -> dict[str, str]:
    raw_config = load_config()  # returns Any
    # cast を使用して型を明示
    return cast(dict[str, str], raw_config)
```

**注意**: cast は実行時のチェックを行わないため、慎重に使用してください。

#### 6. Dataclasses のサポート

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    email: str
    is_active: bool = True  # デフォルト値

# mypy は自動的に __init__, __repr__ などの型をチェック
user = User(name="Alice", age=30, email="alice@example.com")

# frozen dataclass で不変性を保証
@dataclass(frozen=True)
class Point:
    x: float
    y: float
```

#### 7. 読みやすい型ヒントを保つ

```python
# Bad - 長すぎる型定義
def process(data: dict[str, list[tuple[int, str, dict[str, float]]]]) -> None:
    pass

# Good - 型エイリアスで読みやすく
DataPoint: TypeAlias = tuple[int, str, dict[str, float]]
ProcessData: TypeAlias = dict[str, list[DataPoint]]

def process(data: ProcessData) -> None:
    pass
```

#### 8. テストコードも型チェックする

```python
# テストコードにも型アノテーションを追加
import pytest

def test_calculate_total() -> None:
    result: float = calculate_total([1.0, 2.0, 3.0], 0.1)
    assert result == 6.6

# Fixture にも型を追加
@pytest.fixture
def sample_user() -> User:
    return User(name="Test", age=25, email="test@example.com")
```

---

## CI/CD統合

### GitHub Actions での設定例

```yaml
name: Type Check

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

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
          python -m pip install --upgrade pip
          pip install mypy
          pip install types-requests types-PyYAML

      - name: Run mypy
        run: mypy src/
```

### GitLab CI での設定例

```yaml
mypy:
  image: python:3.11
  stage: test
  script:
    - pip install mypy
    - mypy src/
  only:
    - merge_requests
    - main
```

### pre-commit での設定

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests, types-PyYAML]
        args: [--strict, --ignore-missing-imports]
```

### 段階的な導入

```bash
# 1. 新規エラーのみを検出
mypy --show-error-codes src/ > baseline.txt

# 2. ベースラインとの差分をチェック
mypy src/ | diff baseline.txt -
```

---

## よくある問題と解決方法

### インポートエラー

#### 問題: モジュールが見つからない

```
error: Cannot find implementation or library stub for module named 'some_library'
```

#### 解決方法

1. **スタブパッケージのインストール**
   ```bash
   pip install types-requests types-PyYAML
   ```

2. **個別にインポートを無視**
   ```python
   import some_library  # type: ignore
   ```

3. **設定ファイルでモジュール全体を無視**
   ```toml
   [[tool.mypy.overrides]]
   module = "some_library.*"
   ignore_missing_imports = true
   ```

### 型エラーの抑制

#### ファイル単位での無視

```python
# ファイル先頭に記述（import文の前）
# mypy: ignore-errors
```

#### 行単位での無視

```python
result = complex_function()  # type: ignore[return-value]
```

#### 特定のエラーコードのみ無視

```python
x = some_value  # type: ignore[assignment]
```

### 型の絞り込み

```python
def process(value: int | str) -> str:
    if isinstance(value, int):
        # この中では value は int として扱われる
        return str(value * 2)
    else:
        # この中では value は str として扱われる
        return value.upper()
```

### Anyからの漏洩を防ぐ

```python
# Bad - Any が広がる
def bad_function(x):  # x は Any
    return x + 1  # 戻り値も Any

# Good - 明示的な型
def good_function(x: int) -> int:
    return x + 1
```

### reveal_type を使ったデバッグ

```python
from typing import reveal_type

def example(x: int | str) -> None:
    reveal_type(x)  # Revealed type is "int | str"

    if isinstance(x, int):
        reveal_type(x)  # Revealed type is "int"
```

---

## 段階的な厳密化

### レベル1: 基本設定

```toml
[tool.mypy]
python_version = "3.11"
warn_unused_configs = true
```

### レベル2: 基本的な警告を有効化

```toml
[tool.mypy]
python_version = "3.11"
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_return_any = true
strict_equality = true
```

### レベル3: 型チェックを強化

```toml
[tool.mypy]
python_version = "3.11"
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_return_any = true
strict_equality = true

# 追加の厳密化
check_untyped_defs = true
disallow_any_generics = true
disallow_subclassing_any = true
```

### レベル4: 完全なstrictモード

```toml
[tool.mypy]
python_version = "3.11"
strict = true
```

**strictモードで有効になるオプション:**
- `warn_unused_configs`
- `disallow_any_generics`
- `disallow_subclassing_any`
- `disallow_untyped_calls`
- `disallow_untyped_defs`
- `disallow_incomplete_defs`
- `check_untyped_defs`
- `disallow_untyped_decorators`
- `warn_redundant_casts`
- `warn_unused_ignores`
- `warn_return_any`
- `warn_unreachable`
- `strict_equality`
- `strict_concatenate`
- `no_implicit_reexport`

---

## パフォーマンス最適化

### mypy daemon (dmypy) の使用

大規模プロジェクトでは、mypy daemon を使用することで **10倍以上の高速化** が可能です。

```bash
# daemon の起動
dmypy start

# 型チェック実行
dmypy check src/

# 再起動
dmypy restart

# 停止
dmypy stop
```

### リモートキャッシュの活用

チーム開発では、リモートキャッシュを使用して CI/CD での型チェックを高速化できます。

```toml
[tool.mypy]
cache_dir = ".mypy_cache"
sqlite_cache = true
```

### インクリメンタルモードの活用

```toml
[tool.mypy]
incremental = true
cache_dir = ".mypy_cache"
```

### 並列実行

```bash
# 複数のプロセスで並列実行
mypy -j 4 src/
```

### faster-cache の使用

```bash
# orjson を使用した高速キャッシュ
pip install mypy[faster-cache]
```

---

## チーム開発での運用

### 統一されたmypy実行環境

#### 1. 同じ設定ファイルを使用

```toml
# pyproject.toml をリポジトリにコミット
[tool.mypy]
python_version = "3.11"
strict = true
```

#### 2. 同じバージョンのmypyを使用

```txt
# requirements-dev.txt
mypy==1.8.0
types-requests==2.31.0.10
types-PyYAML==6.0.12.12
```

#### 3. 実行スクリプトの統一

```bash
#!/bin/bash
# scripts/run_mypy.sh
set -e

echo "Running mypy..."
python -m mypy src/

echo "✓ mypy check passed!"
```

### CI/CD での継続的なチェック

- プルリクエストごとに mypy を実行
- エラーがある場合はマージをブロック
- 新しいエラーの導入を防止
- **テストコードも含めて型チェック**を実行

### 型チェックの範囲

```toml
# テストコードも含める
[tool.mypy]
files = ["src/", "tests/"]

# または明示的に除外
exclude = [
    "tests/legacy/",
    "scripts/",
]
```

### レビュープロセス

- 型アノテーションの品質をコードレビューでチェック
- `# type: ignore` の使用を最小限に
- 正当な理由がある場合は、コメントで説明

```python
# type: ignore を使用する場合は理由を明記
result = legacy_function()  # type: ignore  # TODO: Fix legacy_function types
```

### ドキュメント化

- 型アノテーションのガイドラインをREADMEに記載
- チーム内で型アノテーションの方針を共有
- 定期的な学習セッションの開催

---

## まとめ

### 導入のステップ

1. **小さく始める**: 一部のモジュールから開始
2. **CI/CD統合**: 早期に自動チェックを導入
3. **段階的に厳密化**: strict モードを目指す
4. **チーム全体で取り組む**: 統一されたプラクティスを確立

### 重要なポイント

- ✅ 新規コードには必ず型アノテーションを追加
- ✅ 既存コード修正時にアノテーションを追加
- ✅ 設定ファイルで段階的に厳密化
- ✅ CI/CD でリグレッションを防止
- ✅ mypy daemon でパフォーマンス向上
- ✅ チーム全体で統一された環境を維持
- ✅ **具体的な型を使用** (Any の使用を最小限に)
- ✅ **Protocol を活用**してダックタイピングを型安全に
- ✅ **型エイリアス**で複雑な型を読みやすく
- ✅ **Dataclasses** で定型コードを削減
- ✅ **テストコード**も型チェック対象に含める

### さらに学ぶために

- [mypy公式ドキュメント](https://mypy.readthedocs.io/)
- [PEP 484 – Type Hints](https://peps.python.org/pep-0484/)
- [PEP 561 – Distributing and Packaging Type Information](https://peps.python.org/pep-0561/)
- [Python typing モジュール](https://docs.python.org/3/library/typing.html)

---

## 参考リンク

詳細な参考リンクは [url.md](url.md) を参照してください。
