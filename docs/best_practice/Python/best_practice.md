# Python ベストプラクティス

このドキュメントは、Python公式ドキュメントに基づいた、実践的なPythonコーディングのベストプラクティスをまとめたものです。

## コーディングスタイル

### PEP 8 スタイルガイド

Python Enhancement Proposal 8 (PEP 8) は、Pythonの標準的なコーディングスタイルガイドです。

#### インデントと空白

- **4スペースインデント**を使用する
  - タブは使用しない
  - 4スペースは、小さなインデント（より深いネスト可能）と大きなインデント（読みやすい）の良い妥協点
- **行の長さは79文字以内**に制限する
  - 小さなディスプレイのユーザーに配慮
  - 大きなディスプレイでは複数のコードファイルを並べて表示可能
- 関数やクラスの間、関数内の大きなコードブロックの間には**空行を挿入**する

#### コメントとドキュメント

- 可能な限り、**コメントは独立した行**に記述する
- **docstringを使用**する
- docstringの最初の行は、オブジェクトの目的を簡潔に要約する
  - 簡潔さのため、オブジェクト名や型を明示的に記述しない
  - 大文字で始まり、ピリオドで終わる
- 複数行のdocstringの場合、2行目は空白行とし、その後に詳細を記述する

```python
def my_function():
    """短い要約を記述する。
    
    2行目は空白行。
    その後に詳細な説明、呼び出し規約、副作用などを記述する。
    """
    pass
```

#### 命名規則

- **クラス名**: `UpperCamelCase`（例: `MyClass`）
- **関数名とメソッド名**: `lowercase_with_underscores`（例: `my_function`）
- メソッドの最初の引数には常に **`self`** を使用する
- 演算子の前後とコンマの後には**スペースを入れる**が、括弧の直後には入れない
  - 正しい例: `a = f(1, 2) + g(3, 4)`

#### エンコーディング

- 国際的な環境で使用するコードの場合、**UTF-8**またはプレーンASCIIを使用する
- 他言語を話す人がコードを読む可能性がある場合、識別子に非ASCII文字を使用しない

## インポートのベストプラクティス

### インポートの順序

インポートは以下の順序で記述する：

1. **標準ライブラリモジュール**（例: `sys`, `os`, `argparse`, `re`）
2. **サードパーティライブラリモジュール**（site-packagesにインストールされたもの）
3. **ローカル開発モジュール**

```python
# 標準ライブラリ
import sys
import os

# サードパーティライブラリ
import requests
import numpy as np

# ローカルモジュール
from myproject import mymodule
```

### インポートのアンチパターン

- **`from modulename import *` を避ける**
  - 名前空間を汚染する
  - 未定義名の検出が困難になる
- **モジュールはファイルの先頭でインポート**する
  - 依存関係を明確にする
  - モジュール名がスコープ内にあるかの疑問を回避
- 循環インポートの問題がある場合のみ、関数やクラス内でのインポートを検討する

## 関数のベストプラクティス

### デフォルト引数の注意点

**ミュータブルなオブジェクトをデフォルト値として使用しない：**

```python
# 悪い例
def foo(mydict={}):
    ...

# 良い例
def foo(mydict=None):
    if mydict is None:
        mydict = {}
```

デフォルト値は関数定義時に一度だけ評価されるため、ミュータブルなオブジェクト（リスト、辞書など）をデフォルト値として使用すると、予期しない動作を引き起こす可能性があります。

### 関数の引数

#### 位置引数とキーワード引数

- **位置専用パラメータ**（`/`の前）: パラメータ名がユーザーに利用可能である必要がない場合
- **キーワード専用パラメータ**（`*`の後）: 名前に意味があり、明示的にすることで関数定義が理解しやすくなる場合

```python
def f(pos1, pos2, /, pos_or_kwd, *, kwd1, kwd2):
    pass
```

### ドキュメンテーション文字列

関数の最初の文に文字列リテラルを記述することで、その関数のドキュメンテーション文字列（docstring）を定義できます。docstringを含める習慣を付けましょう。

```python
def fib(n):
    """n未満のフィボナッチ数列を出力する。"""
    a, b = 0, 1
    while a < n:
        print(a, end=' ')
        a, b = b, a+b
```

### 関数アノテーション

関数アノテーションは、ユーザー定義関数で使用される型に関する完全にオプションのメタデータ情報です。

```python
def f(ham: str, eggs: str = 'eggs') -> str:
    return ham + ' and ' + eggs
```

## エラー処理のベストプラクティス

### 例外処理の原則

1. **特定の例外を処理する**
   - 可能な限り具体的な例外型を指定する
   - `except Exception:` のような広範な例外処理は避ける

```python
try:
    f = open('myfile.txt')
    s = f.readline()
    i = int(s.strip())
except OSError as err:
    print("OS error:", err)
except ValueError:
    print("Could not convert data to an integer.")
except Exception as err:
    print(f"Unexpected {err=}, {type(err)=}")
    raise
```

2. **例外の再発生**
   - 予期しない例外は再発生させる
   - ログに記録してから再発生させることが一般的なパターン

3. **else節の活用**
   - try節で例外が発生しなかった場合にのみ実行されるコード
   - tryブロックに余分なコードを追加するより安全

```python
try:
    f = open(arg, 'r')
except OSError:
    print('cannot open', arg)
else:
    print(arg, 'has', len(f.readlines()), 'lines')
    f.close()
```

### カスタム例外

- カスタム例外は`Exception`クラスから継承する
- 例外名は"Error"で終わるのが慣例

```python
class ValidationError(Exception):
    """検証エラーの基底クラス。"""
    pass
```

### クリーンアップアクション

- **finally節**を使用してクリーンアップアクションを定義する
- `with`文を使用してリソースの自動クリーンアップを行う

```python
with open("myfile.txt") as f:
    for line in f:
        print(line, end="")
```

### 例外チェーン

明示的な例外チェーンを使用して、例外の原因を明確にする：

```python
try:
    func()
except ConnectionError as exc:
    raise RuntimeError('Failed to open database') from exc
```

## データ構造のベストプラクティス

### 変数と可変性

- **イミュータブルなオブジェクト**（str, int, tuple）は予測可能
- **ミュータブルなオブジェクト**（list, dict）の変更には注意が必要

```python
# リストの変更は参照を通じて影響する
y = x  # xとyは同じリストを参照
y.append(4)  # xも変更される
```

### オブジェクトのコピー

- **浅いコピー**: `copy.copy()`またはスライス`list[:]`
- **深いコピー**: `copy.deepcopy()`

```python
import copy

# 辞書の浅いコピー
newdict = olddict.copy()

# シーケンスのコピー
new_list = old_list[:]

# 深いコピー
deep_copied = copy.deepcopy(original)
```

### 文字列の連結

多数の文字列を連結する場合、リストに追加してから`str.join()`を使用する：

```python
# 効率的な方法
chunks = []
for s in my_strings:
    chunks.append(s)
result = ''.join(chunks)
```

## クラスとオブジェクト指向プログラミング

### クラス属性とインスタンス属性

```python
class C:
    count = 0  # クラス属性
    
    def __init__(self):
        C.count = C.count + 1  # クラス名を明示的に使用
    
    def getcount(self):
        return C.count
```

### 静的メソッドとクラスメソッド

```python
class C:
    @staticmethod
    def static_method(arg1, arg2):
        # selfパラメータなし
        pass
    
    @classmethod
    def class_method(cls, arg1):
        # 最初のパラメータはクラス自体
        pass
```

### 継承とインスタンスチェック

- `isinstance(obj, cls)`を使用してインスタンスをチェックする
- 複数のクラスをタプルで指定可能: `isinstance(obj, (class1, class2))`

## パフォーマンスのベストプラクティス

### 一般原則

1. **アルゴリズムの最適化を優先**
   - マイクロ最適化よりも効率的なアルゴリズムを選択
2. **適切なデータ構造を使用**
   - `collections`モジュールの活用
3. **組み込み関数を活用**
   - Pythonの組み込み機能は高速（特にC実装のもの）
4. **プロファイリング**
   - `profile`モジュールでホットスポットを特定
   - `timeit`モジュールで実行時間を測定
5. **過度な抽象化を避ける**
   - 小さな関数やメソッドの過度な使用は逆効果

### 具体的なテクニック

- `list.sort()`または`sorted()`を使用してソート
- 標準ライブラリのプリミティブを優先的に使用
- 必要に応じてCythonやC拡張を検討

## モジュールとパッケージ

### グローバル変数の共有

単一プログラム内でモジュール間で情報を共有するには、特別なモジュール（configまたはcfgと呼ばれることが多い）を作成：

```python
# config.py
x = 0

# mod.py
import config
config.x = 1

# main.py
import config
import mod
print(config.x)  # 1
```

### 循環インポートの回避

1. `from <module> import ...`の使用を避ける
2. インポート文の前にエクスポート（グローバル、関数、クラス）を配置
3. 可能であれば、循環インポートが不要になるようにコードを再構築

## テストとデバッグ

### ユニットテスト

- `unittest`モジュールを使用
- `doctest`でドキュメント内の例をテスト

### デバッグツール

- **pdb**: Python標準デバッガー
- `breakpoint()`関数でデバッガーに入る

### 静的解析

- **Pylint**: 基本的なチェック
- **Pyflakes**: 軽量な静的解析
- **Mypy**: 型チェッカー

## アノテーションのベストプラクティス

### アクセス方法

Python 3.10以降では`inspect.get_annotations()`を使用：

```python
import inspect

annotations = inspect.get_annotations(my_function)
```

### 一般的なルール

1. `__annotations__`への直接代入を避ける
2. 常に`dict`オブジェクトを設定
3. `__annotations__`を直接アクセスせず、`inspect.get_annotations()`を使用
4. `__annotations__`辞書の変更を避ける

## セキュリティのベストプラクティス

1. **ユーザー入力の検証**
   - `eval()`の使用を避ける（特にユーザー入力に対して）
   - 代わりに`int()`, `float()`などの型コンストラクタを使用

2. **機密情報の管理**
   - `secrets`モジュールを使用して安全な乱数を生成
   - `hashlib`と`hmac`でセキュアなハッシュを使用

3. **ファイルパスの処理**
   - `pathlib`モジュールを使用してパスを安全に操作

## ロギングのベストプラクティス

### loggingモジュールの使用

`print()`の代わりに`logging`モジュールを使用する：

```python
import logging

# ロガーの設定
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ログの記録
logger.debug('デバッグ情報')
logger.info('情報メッセージ')
logger.warning('警告メッセージ')
logger.error('エラーメッセージ')
logger.critical('重大なエラー')
```

### ログレベル

適切なログレベルを使用する：

- **DEBUG**: 詳細な診断情報
- **INFO**: 一般的な情報メッセージ
- **WARNING**: 警告（プログラムは動作するが注意が必要）
- **ERROR**: エラー（一部の機能が動作しない）
- **CRITICAL**: 重大なエラー（プログラムの継続が困難）

## 仮想環境のベストプラクティス

### venvの使用

プロジェクトごとに仮想環境を作成する：

```bash
# 仮想環境の作成
python -m venv myenv

# 仮想環境のアクティブ化（Linux/Mac）
source myenv/bin/activate

# 仮想環境のアクティブ化（Windows）
myenv\Scripts\activate

# パッケージのインストール
pip install requests

# requirements.txtの生成
pip freeze > requirements.txt
```

### 依存関係の管理

- `requirements.txt`でプロジェクトの依存関係を管理
- バージョンを明示的に指定する（例: `requests==2.28.0`）
- 開発用と本番用の依存関係を分ける（`requirements-dev.txt`）

## コード品質ツール

### リンターとフォーマッター

- **Pylint**: コード品質チェック
- **Pyflakes**: 構文エラーの検出
- **Black**: 自動コードフォーマッター
- **isort**: インポート文の自動整理
- **flake8**: PEP 8準拠チェック

```bash
# Blackでコードをフォーマット
black myfile.py

# isortでインポートを整理
isort myfile.py

# flake8でコードをチェック
flake8 myfile.py
```

### 型チェック

- **Mypy**: 静的型チェック
- **Pyre**: Facebook製の型チェッカー
- **Pytype**: Google製の型チェッカー

```python
# 型ヒントの使用例
from typing import List, Dict, Optional

def process_data(items: List[str]) -> Dict[str, int]:
    """文字列のリストを処理して辞書を返す。"""
    return {item: len(item) for item in items}

def find_user(user_id: int) -> Optional[str]:
    """ユーザーIDからユーザー名を検索する。見つからない場合はNoneを返す。"""
    users = {1: "Alice", 2: "Bob"}
    return users.get(user_id)
```

## モジュール化とコードの再利用

### DRY原則（Don't Repeat Yourself）

- コードの重複を避ける
- 再利用可能な関数やクラスを作成する
- 共通ロジックをユーティリティモジュールに抽出する

```python
# 悪い例：コードの重複
def calculate_price_with_tax_domestic(price):
    return price * 1.1

def calculate_price_with_tax_export(price):
    return price * 1.08

# 良い例：再利用可能な関数
def calculate_price_with_tax(price, tax_rate):
    return price * (1 + tax_rate)

domestic_price = calculate_price_with_tax(1000, 0.1)
export_price = calculate_price_with_tax(1000, 0.08)
```

### SOLID原則

- **単一責任の原則**: クラスは1つの責任のみを持つ
- **開放閉鎖の原則**: 拡張に開いていて、修正に閉じている
- **リスコフの置換原則**: 派生クラスは基底クラスと置換可能
- **インターフェース分離の原則**: クライアントは使用しないメソッドに依存しない
- **依存性逆転の原則**: 抽象に依存し、具象に依存しない

## まとめ

このベストプラクティスガイドは、Python公式ドキュメントとコミュニティの標準に基づいて作成されました。これらのガイドラインに従うことで：

- **可読性の高いコード**を書くことができる
- **保守しやすいコード**を作成できる
- **効率的なコード**を実装できる
- **安全なコード**を開発できる
- **チームでの協業が容易**になる

### 継続的な学習

- PEP 8やその他の関連PEPの更新を常に確認する
- コミュニティのベストプラクティスに従う
- コードレビューを活用して品質を向上させる
- 静的解析ツールとテストを CI/CD パイプラインに組み込む

Pythonは常に進化しているため、最新のベストプラクティスを学び続けることが重要です。
