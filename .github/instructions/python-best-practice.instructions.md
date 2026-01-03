---
description: 'PEP 8準拠のPythonコーディング規約とベストプラクティス'
applyTo: '**/*.py'
---

# Python ベストプラクティス

Python公式ドキュメントとPEP 8に基づいた、実践的なPythonコーディングのベストプラクティスです。このガイドラインに従うことで、可読性が高く、保守しやすい高品質なPythonコードを作成できます。

## プロジェクトコンテキスト

- 対象言語: Python 3.10以降
- スタイルガイド: PEP 8 (Python Enhancement Proposal 8)
- 型チェック: Mypy推奨
- フォーマッター: Black推奨
- インポート整理: isort推奨

## コーディングスタイル

### インデントと空白

- **4スペースインデント**を使用する（タブは使用しない）
- 行の長さは**79文字以内**に制限する
- 関数やクラスの間には**2行の空行**を挿入する
- クラス内のメソッド間には**1行の空行**を挿入する
- 関数内の論理的なセクション間には空行を適度に使用する

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

演算子の前後とコンマの後にスペースを入れるが、括弧の直後には入れない：

```python
# 良い例
result = function(arg1, arg2) + another_function(arg3, arg4)
x = 1
y = 2
total = x + y

# 悪い例
result=function( arg1,arg2 )+another_function( arg3,arg4 )
```

## 命名規則

| 対象 | 規則 | 例 |
|------|------|-----|
| クラス名 | `UpperCamelCase` | `MyClass`, `DataProcessor` |
| 関数・メソッド名 | `lowercase_with_underscores` | `my_function`, `calculate_total` |
| 変数名 | `lowercase_with_underscores` | `user_name`, `total_count` |
| 定数 | `UPPER_CASE_WITH_UNDERSCORES` | `MAX_SIZE`, `DEFAULT_TIMEOUT` |
| モジュール名 | `lowercase` (アンダースコア可) | `mymodule`, `data_parser` |
| プライベート属性 | `_leading_underscore` | `_internal_value` |

```python
# 良い例
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

### メソッドの特殊な引数名

- インスタンスメソッドの最初の引数: **`self`**
- クラスメソッドの最初の引数: **`cls`**

```python
class MyClass:
    def instance_method(self, arg):
        """インスタンスメソッド。"""
        pass
    
    @classmethod
    def class_method(cls, arg):
        """クラスメソッド。"""
        pass
    
    @staticmethod
    def static_method(arg):
        """静的メソッド。"""
        pass
```

## インポートのベストプラクティス

### インポートの順序

以下の順序でインポートを記述し、各グループ間に空行を挿入する：

1. 標準ライブラリモジュール
2. サードパーティライブラリモジュール
3. ローカル開発モジュール

```python
# 良い例
# 標準ライブラリ
import os
import sys
from pathlib import Path
from typing import List, Optional

# サードパーティライブラリ
import numpy as np
import requests
from flask import Flask, request

# ローカルモジュール
from myproject.core import database
from myproject.utils import helpers
```

### インポートの禁止事項

```python
# 悪い例：ワイルドカードインポート（使用禁止）
from module import *

# 良い例：明示的なインポート
from module import function1, function2
```

**理由**:
- 名前空間を汚染する
- 未定義名の検出が困難になる
- コードの依存関係が不明確になる

### インポートの配置

- モジュールの**先頭**でインポートを行う
- 関数やクラス内でのインポートは、循環インポートの回避が必要な場合のみ許可

## 関数とドキュメンテーション

### docstringの記述

すべての公開関数、メソッド、クラスに**docstring**を記述する：

```python
def calculate_sum(numbers: List[int]) -> int:
    """数値のリストの合計を計算する。
    
    Args:
        numbers: 合計を計算する整数のリスト
    
    Returns:
        すべての数値の合計
    
    Raises:
        ValueError: リストが空の場合
    """
    if not numbers:
        raise ValueError("リストが空です")
    return sum(numbers)
```

### docstringの構造

- **1行目**: 大文字で始まり、ピリオドで終わる簡潔な要約
- **2行目**: 空行
- **3行目以降**: 詳細な説明、引数、戻り値、例外などを記述

### 型ヒント

Python 3.10以降では型ヒントを積極的に使用する：

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

### デフォルト引数の注意点

**ミュータブルなオブジェクト（リスト、辞書など）をデフォルト値として使用しない**：

```python
# 悪い例
def append_to_list(item, my_list=[]):
    my_list.append(item)
    return my_list

# 良い例
def append_to_list(item, my_list=None):
    """アイテムをリストに追加する。"""
    if my_list is None:
        my_list = []
    my_list.append(item)
    return my_list
```

**理由**: デフォルト値は関数定義時に一度だけ評価されるため、ミュータブルなオブジェクトを使用すると予期しない動作を引き起こす。

## エラー処理

### 具体的な例外を処理

```python
# 良い例：特定の例外を処理
try:
    with open('config.json', 'r') as f:
        config = json.load(f)
except FileNotFoundError:
    logger.error("設定ファイルが見つかりません")
    config = get_default_config()
except json.JSONDecodeError as e:
    logger.error(f"JSONパースエラー: {e}")
    raise
except Exception as e:
    logger.error(f"予期しないエラー: {e}")
    raise

# 悪い例：広範な例外処理
try:
    do_something()
except Exception:  # 避けるべき
    pass
```

### else節とfinally節の活用

```python
# else節：例外が発生しなかった場合のみ実行
try:
    f = open('data.txt', 'r')
except FileNotFoundError:
    logger.error("ファイルが見つかりません")
else:
    # 例外が発生しなかった場合のみ実行
    data = f.read()
    f.close()
    process_data(data)

# finally節：必ず実行されるクリーンアップ
try:
    connection = create_connection()
    execute_query(connection)
except DatabaseError as e:
    logger.error(f"データベースエラー: {e}")
    raise
finally:
    # 必ず実行される
    if connection:
        connection.close()
```

### with文によるリソース管理

ファイル、データベース接続、ロックなどのリソースには**with文**を使用する：

```python
# 良い例：with文を使用
with open('file.txt', 'r') as f:
    content = f.read()
    process(content)

# 複数のリソース
with open('input.txt', 'r') as infile, open('output.txt', 'w') as outfile:
    for line in infile:
        outfile.write(process_line(line))
```

### カスタム例外

プロジェクト固有の例外を定義する際は、`Exception`を継承し、名前を"Error"で終わらせる：

```python
class ValidationError(Exception):
    """検証エラーの基底クラス。"""
    pass


class UserNotFoundError(Exception):
    """ユーザーが見つからない場合の例外。"""
    
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"ユーザーID {user_id} が見つかりません")
```

### 例外チェーン

例外の原因を明確にするため、例外チェーンを使用する：

```python
try:
    connect_to_database()
except ConnectionError as e:
    raise DatabaseConnectionError('データベース接続に失敗しました') from e
```

## データ構造とアルゴリズム

### 文字列の連結

複数の文字列を連結する場合は、リストに追加してから`str.join()`を使用する：

```python
# 良い例：効率的
parts = []
for item in items:
    parts.append(str(item))
result = ', '.join(parts)

# さらに良い例：リスト内包表記
result = ', '.join(str(item) for item in items)

# 悪い例：非効率
result = ''
for item in items:
    result += str(item) + ', '
```

### リスト内包表記とジェネレータ式

```python
# リスト内包表記：メモリにリストを作成
squares = [x**2 for x in range(10)]

# ジェネレータ式：メモリ効率が良い
squares_gen = (x**2 for x in range(10))

# 条件付き内包表記
even_squares = [x**2 for x in range(10) if x % 2 == 0]

# 辞書内包表記
user_dict = {user.id: user.name for user in users}
```

### 適切なデータ構造の選択

| データ構造 | 用途 | 特徴 |
|-----------|------|------|
| `list` | 順序付きコレクション | 可変、インデックスアクセス |
| `tuple` | 不変な順序付きコレクション | イミュータブル |
| `set` | 一意な要素のコレクション | 高速な検索、順序なし |
| `dict` | キー・バリューペア | 高速な検索 |
| `collections.deque` | 両端キュー | 高速な両端操作 |
| `collections.defaultdict` | デフォルト値付き辞書 | キーがない場合も安全 |
| `collections.Counter` | カウンター | 要素の出現回数を集計 |

```python
from collections import defaultdict, Counter, deque

# defaultdict：キーがない場合もエラーにならない
user_groups = defaultdict(list)
user_groups['admin'].append('Alice')

# Counter：要素の出現回数を数える
word_counts = Counter(['apple', 'banana', 'apple', 'orange'])

# deque：両端での高速な追加・削除
queue = deque([1, 2, 3])
queue.append(4)  # 右端に追加
queue.appendleft(0)  # 左端に追加
```

## クラスとオブジェクト指向

### クラス属性とインスタンス属性

```python
class MyClass:
    """サンプルクラス。"""
    
    class_variable = 0  # クラス属性（全インスタンスで共有）
    
    def __init__(self, value):
        self.instance_variable = value  # インスタンス属性（各インスタンス固有）
        MyClass.class_variable += 1  # クラス属性の更新は明示的に
```

### プロパティの使用

属性へのアクセスを制御する場合は、`@property`デコレータを使用する：

```python
class User:
    """ユーザークラス。"""
    
    def __init__(self, name: str, age: int):
        self._name = name
        self._age = age
    
    @property
    def name(self) -> str:
        """ユーザー名を取得。"""
        return self._name
    
    @property
    def age(self) -> int:
        """年齢を取得。"""
        return self._age
    
    @age.setter
    def age(self, value: int) -> None:
        """年齢を設定（検証付き）。"""
        if value < 0:
            raise ValueError("年齢は0以上である必要があります")
        self._age = value
```

### 継承とインスタンスチェック

```python
# インスタンスチェック
if isinstance(obj, MyClass):
    process(obj)

# 複数のクラスをチェック
if isinstance(obj, (str, bytes)):
    handle_text(obj)

# サブクラスチェック
if issubclass(MyClass, BaseClass):
    do_something()
```

## パフォーマンスと最適化

### 一般原則

1. **アルゴリズムの最適化を優先**する（マイクロ最適化よりも重要）
2. **適切なデータ構造**を選択する（`collections`モジュールを活用）
3. **組み込み関数**を活用する（C実装で高速）
4. **プロファイリング**してボトルネックを特定する（`cProfile`, `timeit`）
5. **過度な抽象化**を避ける

```python
# 良い例：組み込み関数を使用
numbers = [1, 2, 3, 4, 5]
total = sum(numbers)
maximum = max(numbers)

# 良い例：リスト内包表記（高速）
squares = [x**2 for x in range(1000)]

# 悪い例：不要なループ
squares = []
for x in range(1000):
    squares.append(x**2)
```

### 計測とプロファイリング

```python
import timeit
import cProfile

# 実行時間の計測
execution_time = timeit.timeit('sum(range(100))', number=10000)

# プロファイリング
cProfile.run('my_function()')
```

## ロギング

`print()`の代わりに**loggingモジュール**を使用する：

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
logger.info('処理を開始します')
logger.warning('警告：設定が見つかりません')
logger.error('エラーが発生しました')
logger.critical('重大なエラー')
```

### ログレベルの使い分け

| レベル | 用途 |
|--------|------|
| DEBUG | 詳細な診断情報（開発時） |
| INFO | 一般的な情報メッセージ |
| WARNING | 警告（動作するが注意が必要） |
| ERROR | エラー（一部機能が動作しない） |
| CRITICAL | 重大なエラー（プログラム継続困難） |

## セキュリティ

### 入力の検証

```python
# 悪い例：eval()の使用（危険）
user_input = "os.system('rm -rf /')"
eval(user_input)  # 絶対に使用しない

# 良い例：型変換を使用
user_input = "42"
value = int(user_input)

# 良い例：安全な評価
import ast
user_input = "[1, 2, 3]"
safe_value = ast.literal_eval(user_input)
```

### 機密情報の管理

```python
import secrets
import hashlib

# 安全な乱数生成
token = secrets.token_hex(16)

# セキュアなハッシュ
password = "user_password"
salt = secrets.token_bytes(32)
hashed = hashlib.pbkdf2_hmac('sha256', password.encode(), salt, 100000)
```

### ファイルパスの安全な処理

```python
from pathlib import Path

# 良い例：pathlibを使用
base_dir = Path('/safe/directory')
user_file = base_dir / 'user_input.txt'  # パストラバーサル対策

# パスの検証
if user_file.resolve().is_relative_to(base_dir):
    process_file(user_file)
```

## コード品質ツール

### 推奨ツール

プロジェクトに以下のツールを導入する：

```bash
# フォーマッター
black .  # コードを自動フォーマット
isort .  # インポートを自動整理

# リンター
flake8 .  # PEP 8準拠チェック
pylint mymodule.py  # コード品質チェック

# 型チェック
mypy mymodule.py  # 静的型チェック
```

### pre-commitフックの設定

`.pre-commit-config.yaml`を設定して自動チェックを有効化：

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
  
  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
  
  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
```

## チェックリスト

コード作成時に以下を確認する：

### スタイルとフォーマット
- [ ] 4スペースインデントを使用している
- [ ] 行の長さが79文字以内である
- [ ] 命名規則に従っている（クラス名、関数名、変数名）
- [ ] インポートが正しい順序で記述されている
- [ ] `from module import *` を使用していない

### ドキュメンテーション
- [ ] すべての公開関数・メソッド・クラスにdocstringがある
- [ ] docstringに引数、戻り値、例外が記述されている
- [ ] 型ヒントを使用している

### エラー処理
- [ ] 具体的な例外を処理している
- [ ] リソース管理にwith文を使用している
- [ ] 例外を適切に処理または再発生させている

### コード品質
- [ ] デフォルト引数にミュータブルなオブジェクトを使用していない
- [ ] ロギングにloggingモジュールを使用している
- [ ] `eval()`などの危険な関数を使用していない
- [ ] 適切なデータ構造を選択している

### テスト
- [ ] ユニットテストを作成している
- [ ] 境界条件をテストしている
- [ ] 例外処理をテストしている

### パフォーマンス
- [ ] 適切なアルゴリズムとデータ構造を選択している
- [ ] 不要なループや処理を避けている
- [ ] 組み込み関数を活用している

## 参考リソース

- [PEP 8 -- Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Python公式ドキュメント](https://docs.python.org/ja/3/)
- [The Zen of Python (PEP 20)](https://peps.python.org/pep-0020/)
- [Type Hints (PEP 484)](https://peps.python.org/pep-0484/)
