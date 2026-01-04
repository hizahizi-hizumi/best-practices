````instructions
---
description: 'PEP 8 準拠の Python コーディング規約とベストプラクティス'
applyTo: '**/*.py'
---

# Python ベストプラクティス

## プロジェクトコンテキスト

- 対象言語: Python 3.10 以降
- スタイルガイド: PEP 8
- 型チェッカー: Mypy
- フォーマッター: Black
- インポート整理: isort

## コーディングスタイル

### インデントと空白

- 4スペースインデントを使用する (タブは使用しない)
- 行の長さは 79 文字以内に制限する
- 関数とクラスの間には 2 行の空行を挿入する
- クラス内のメソッド間には 1 行の空行を挿入する

```python
# 良い例
def first_function():
    """最初の関数。"""
    pass


def second_function():
    """2番目の関数。"""
    pass


class MyClass:
    """サンプルクラス。"""

    def method_one(self):
        """最初のメソッド。"""
        pass

    def method_two(self):
        """2番目のメソッド。"""
        pass
```

### 演算子とスペース

- 演算子の前後とカンマの後にスペースを挿入する
- 開き括弧の直後にスペースを挿入しない

```python
# 推奨
result = function(arg1, arg2) + another_function(arg3, arg4)

# 非推奨
result=function( arg1,arg2 )+another_function( arg3,arg4 )
```

## 命名規則

| 対象 | 規則 | 例 |
|------|------|-----|
| クラス名 | `UpperCamelCase` | `MyClass`, `DataProcessor` |
| 関数/メソッド名 | `lowercase_with_underscores` | `my_function`, `calculate_total` |
| 変数名 | `lowercase_with_underscores` | `user_name`, `total_count` |
| 定数 | `UPPER_CASE_WITH_UNDERSCORES` | `MAX_SIZE`, `DEFAULT_TIMEOUT` |
| モジュール名 | `lowercase` (アンダースコア可) | `mymodule`, `data_parser` |
| プライベート属性 | `_leading_underscore` | `_internal_value` |

```python
# 推奨
class UserManager:
    """ユーザー管理クラス。"""

    MAX_USERS = 1000  # 定数

    def __init__(self):
        self._users = []  # プライベート属性

    def add_user(self, user_name: str) -> bool:
        """ユーザーを追加する。"""
        if len(self._users) < self.MAX_USERS:
            self._users.append(user_name)
            return True
        return False
```

### メソッド引数名

- インスタンスメソッドの第1引数には `self` を使用する
- クラスメソッドの第1引数には `cls` を使用する

## インポートのベストプラクティス

### インポート順序

1. 標準ライブラリのインポート
2. サードパーティライブラリのインポート
3. ローカルモジュールのインポート

各グループ間に空行を挿入する。

```python
# 推奨
import os
from typing import List

import numpy as np
import requests

from myproject.core import database
```

### インポートの制約

- ワイルドカードインポート (`from module import *`) は使用しない

**理由**: 名前空間の汚染、未定義名の検出困難、依存関係の不明確化

### インポートの配置

- モジュールの先頭にインポートを配置する
- 循環依存を回避する場合のみ、関数内やクラス内でインポートする

## 関数とドキュメント

### docstring の記述

- すべてのパブリック関数、メソッド、クラスに docstring を記述する

```python
def calculate_sum(numbers: List[int]) -> int:
    """数値リストの合計を計算する。

    Args:
        numbers: 合計する整数のリスト

    Returns:
        すべての数値の合計

    Raises:
        ValueError: リストが空の場合
    """
    if not numbers:
        raise ValueError("List is empty")
    return sum(numbers)
```

### docstring の構造

- 1行目: 簡潔な要約 (大文字で始め、ピリオドで終わる)
- 2行目: 空行
- 3行目以降: Args、Returns、Raises を記述する

### 型ヒント

- 型ヒントを積極的に使用する

```python
from typing import List, Dict, Optional, Union

def process_users(
    user_ids: List[int],
    options: Optional[Dict[str, str]] = None
) -> Dict[int, str]:
    """ユーザーIDを処理してユーザー情報を返す。

    Args:
        user_ids: 処理するユーザーIDのリスト
        options: オプションのパラメータ辞書

    Returns:
        ユーザーIDをキー、ユーザー名を値とする辞書
    """
    if options is None:
        options = {}

    return {user_id: f"User_{user_id}" for user_id in user_ids}
```

### デフォルト引数

- ミュータブルなオブジェクト (リスト、辞書など) をデフォルト値として使用しない

```python
# 非推奨
def append_to_list(item, my_list=[]):
    my_list.append(item)
    return my_list

# 推奨
def append_to_list(item, my_list=None):
    if my_list is None:
        my_list = []
    my_list.append(item)
    return my_list
```

**理由**: デフォルト値は定義時に一度だけ評価されるため、予期しない動作を引き起こす可能性がある

## エラーハンドリング

### 特定の例外を処理する

- 特定の例外クラスをキャッチする
- 広範な `Exception` キャッチを避ける

```python
# 推奨
try:
    with open('config.json', 'r') as f:
        config = json.load(f)
except FileNotFoundError:
    logger.error("構成ファイルが見つかりません")
    config = get_default_config()
except json.JSONDecodeError as e:
    logger.error(f"JSON解析エラー: {e}")
    raise
except Exception as e:
    logger.error(f"予期しないエラー: {e}")
    raise
```

### else 節と finally 節

- else 節は例外が発生しなかった場合の処理を記述する
- finally 節はクリーンアップ処理を必ず実行する

### with 文によるリソース管理

- ファイル、データベース接続、ロックなどのリソースには with 文を使用する

```python
# 推奨
with open('file.txt', 'r') as f:
    content = f.read()
    process(content)
```

### カスタム例外

- `Exception` を継承する
- 名前は "Error" で終わる

```python
class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User ID {user_id} not found")
```

### 例外チェーン

- 例外チェーンには `from e` 構文を使用して原因を明確にする

## データ構造とアルゴリズム

### 文字列連結

- 複数の文字列を連結するには `str.join()` を使用する
- ループ内での `+=` 連結を避ける

```python
# 推奨
result = ', '.join(str(item) for item in items)

# 非推奨
result = ''
for item in items:
    result += str(item) + ', '
```

### リスト内包表記とジェネレータ式

- リスト内包表記を使用して簡潔なループを記述する
- 大量のデータにはジェネレータ式を使用する
- 辞書内包表記とセット内包表記を活用する

```python
squares = [x**2 for x in range(10)]
squares_gen = (x**2 for x in range(10))  # メモリ効率的
even_squares = [x**2 for x in range(10) if x % 2 == 0]
```

### 適切なデータ構造の選択

| データ構造 | 用途 | 特性 |
|------------|------|------|
| `list` | 順序付きコレクション | ミュータブル、インデックスアクセス |
| `tuple` | イミュータブルな順序付きコレクション | イミュータブル |
| `set` | 一意な要素のコレクション | 高速検索、順序なし |
| `dict` | キーと値のペア | 高速検索 |
| `collections.deque` | 両端キュー | 両端での高速操作 |
| `collections.defaultdict` | デフォルト値を持つ辞書 | キーが存在しなくても安全 |
| `collections.Counter` | カウンター | 要素の出現回数を集計 |

```python
from collections import defaultdict, Counter, deque

user_groups = defaultdict(list)
user_groups['admin'].append('Alice')

word_counts = Counter(['apple', 'banana', 'apple'])

queue = deque([1, 2, 3])
queue.append(4)
```

## クラスとオブジェクト指向プログラミング

### クラス属性とインスタンス属性

- クラス属性は `ClassName.attribute` で明示的に更新する
- インスタンス属性は `__init__` 内で `self.attribute` として定義する

### プロパティの使用

- `@property` デコレータを使用して属性アクセスを制御する

```python
class User:
    """ユーザークラス。"""

    def __init__(self, name: str, age: int):
        self._name = name
        self._age = age

    @property
    def name(self) -> str:
        """ユーザー名を取得する。"""
        return self._name

    @property
    def age(self) -> int:
        """年齢を取得する。"""
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        """年齢を設定する (検証付き)。"""
        if value < 0:
            raise ValueError("Age must be 0 or greater")
        self._age = value
```

### 継承とインスタンスチェック

- 型チェックには `isinstance()` を使用する
- サブクラスチェックには `issubclass()` を使用する

## パフォーマンスと最適化

### 一般原則

- アルゴリズムの最適化を優先する
- 適切なデータ構造を選択する (`collections` モジュールを活用)
- 組み込み関数を活用する (高速な C 実装)
- プロファイリングでボトルネックを特定する (`cProfile`, `timeit`)
- 過度な抽象化を避ける

```python
# 推奨
total = sum(numbers)
squares = [x**2 for x in range(1000)]
```

### 測定とプロファイリング

- 実行時間の測定には `timeit` を使用する
- ボトルネックの特定には `cProfile` を使用する

## ログ記録

- `print()` の代わりに `logging` モジュールを使用する

```python
import logging

# ロガーの設定
logger = logging.getLogger(__name__)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# ログの記録
logger.debug('デバッグ情報')
logger.info('処理を開始する')
logger.warning('警告: 構成が見つかりません')
logger.error('エラーが発生しました')
logger.critical('致命的なエラー')
```

### ログレベルの使用

| レベル | 用途 |
|--------|------|
| DEBUG | 詳細な診断情報 (開発中) |
| INFO | 一般的な情報メッセージ |
| WARNING | 警告メッセージ (動作するが注意が必要) |
| ERROR | エラーメッセージ (一部の機能が動作しない) |
| CRITICAL | 致命的なエラー (プログラムの継続が困難) |

## セキュリティ

### 入力検証

- `eval()` を使用しない (セキュリティリスク)
- リテラル評価には `ast.literal_eval()` を使用する

```python
# 推奨
import ast
user_input = "[1, 2, 3]"
safe_value = ast.literal_eval(user_input)
```

### 機密情報の管理

- 乱数生成には `secrets` モジュールを使用する
- パスワードハッシュには `hashlib.pbkdf2_hmac` を使用する

### 安全なファイルパス処理

- パスの構築には `pathlib` を使用する
- `Path.is_relative_to()` でパストラバーサルを防ぐ

## コード品質ツール

### 推奨ツール

- Black: コードの自動フォーマット
- isort: インポートの自動整理
- flake8: PEP 8 準拠チェック
- mypy: 静的型チェック

### pre-commit フック設定

- `.pre-commit-config.yaml` で Black、isort、flake8 を設定する

## チェックリスト

### スタイルとフォーマット
- [ ] 4スペースインデントを使用する
- [ ] 行の長さを 79 文字以内に制限する
- [ ] 命名規則に従う
- [ ] 正しいインポート順序
- [ ] ワイルドカードインポートを使用しない

### ドキュメント
- [ ] パブリック関数、メソッド、クラスに docstring を記述する
- [ ] docstring に Args、Returns、Raises を記述する
- [ ] 型ヒントを使用する

### エラーハンドリング
- [ ] 特定の例外処理
- [ ] with 文によるリソース管理
- [ ] 適切な例外処理と再送出

### コード品質
- [ ] デフォルト引数にミュータブルなオブジェクトを使用しない
- [ ] logging モジュールを使用する
- [ ] `eval()` を使用しない
- [ ] 適切なデータ構造を選択する

## 参考リソース

- [PEP 8 -- Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Python 公式ドキュメント](https://docs.python.org/ja/3/)
- [The Zen of Python (PEP 20)](https://peps.python.org/pep-0020/)
- [Type Hints (PEP 484)](https://peps.python.org/pep-0484/)

````
