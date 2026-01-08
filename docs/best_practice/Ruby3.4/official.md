# Ruby 3.4 公式ベストプラクティス

本ドキュメントは、Ruby 3.4 の公式ドキュメントから収集したベストプラクティスをまとめたものです。

## 概要

Ruby はオブジェクト指向スクリプト言語であり、以下の特徴を持ちます：

- **インタプリタ言語**: コンパイル不要で即座に実行可能
- **動的型付け**: 変数に型宣言が不要で柔軟なプログラミングが可能
- **純粋なオブジェクト指向**: 整数を含むすべてのデータがオブジェクト
- **ガーベージコレクション**: メモリ管理が自動化
- **強力な文字列操作**: Perl をお手本とした正規表現サポート
- **例外処理機能**: 堅牢なエラーハンドリング機構

## クラスとメソッド定義のベストプラクティス

### クラス定義の基本

```ruby
class ClassName < SuperClass
  def initialize(param)
    @instance_var = param
  end
  
  def method_name
    # メソッド本体
  end
end
```

**推奨事項**:

- クラス名は必ずアルファベットの大文字で始める（定数として扱われる）
- スーパークラスを省略した場合は自動的に `Object` クラスを継承
- インスタンス変数は `@` で始まる名前を使用
- クラス定義内では `rescue`/`ensure` 節を使用可能

### メソッド定義のベストプラクティス

```ruby
# 基本的なメソッド定義
def method_name(required_param, optional_param = default_value)
  # メソッド本体
end

# 可変長引数
def method_with_rest(*args)
  args.each { |arg| puts arg }
end

# キーワード引数
def method_with_keywords(name:, age: 18)
  puts "#{name} is #{age} years old"
end

# ブロック引数
def method_with_block(&block)
  block.call if block
end
```

**注意点**:

- デフォルト値を持つ引数は連続した区間に配置する必要がある
- デフォルト式は呼び出し時に評価される
- 仮引数の順序: 必須引数 → デフォルト引数 → `*`引数 → 後置必須引数 → キーワード引数 → `**`引数 → `&`引数

### イテレータの定義

```ruby
# yield を使用する方法（推奨）
def each_item
  return unless block_given?
  yield item1
  yield item2
end

# Proc.new を使用する方法
def each_with_proc
  return unless block_given?
  Proc.new.call(item)
end

# ブロック引数を使用する方法
def each_with_block(&block)
  return unless block
  block.call(item)
end
```

## メソッドの可視性（呼び出し制限）

### 可視性の種類

- **public**: 制限なしに呼び出し可能（デフォルト）
- **private**: 関数形式でのみ呼び出し可能（`self.method` 形式は Ruby 2.7+ で許可）
- **protected**: そのメソッドを持つオブジェクトが `self` であるコンテキストでのみ呼び出し可能

```ruby
class Example
  def public_method
    # 誰でも呼び出せる
  end
  
  def protected_method
    # 同じクラスのインスタンスから呼び出せる
  end
  protected :protected_method
  
  def private_method
    # このオブジェクトの内部からのみ呼び出せる
  end
  private :private_method
  
  # または、以下のようにまとめて指定
  private
  
  def another_private_method
    # これも private
  end
end
```

**ベストプラクティス**:

- `initialize` と `initialize_copy` は自動的に `private` になる
- 外部から呼ぶ必要のないヘルパーメソッドは `private` にする
- 同じクラスのインスタンス間で比較演算などを行う場合は `protected` を使用

## クラスメソッドの定義

```ruby
class MyClass
  # 方法1: 特異メソッド方式（クラス名を明示）
  def MyClass.class_method1
    'class method 1'
  end
  
  # 方法2: self を使用（推奨: クラス名の変更に強い）
  def self.class_method2
    'class method 2'
  end
  
  # 方法3: 特異クラス方式（複数定義時に便利）
  class << self
    def class_method3
      'class method 3'
    end
    
    def class_method4
      'class method 4'
    end
  end
end
```

## モジュールと Mix-in

### モジュールの基本

```ruby
module MyModule
  def module_method
    'module method'
  end
  
  module_function :module_method
end

# モジュールメソッドとして呼び出し
MyModule.module_method

# Mix-in として使用
class MyClass
  include MyModule  # インスタンスメソッドとして追加
end

class AnotherClass
  extend MyModule   # クラスメソッドとして追加
end
```

**推奨パターン**:

- 関連するクラスをまとめる名前空間として使用
- 共通機能を複数のクラスで共有する Mix-in として使用
- 定数をグループ化するコンテナとして使用

## 制御構造のベストプラクティス

### 条件分岐

```ruby
# if 文
if condition
  # 処理
elsif another_condition
  # 処理
else
  # 処理
end

# if 修飾子（1行で書く場合）
do_something if condition

# unless（否定条件）
unless error_occurred
  continue_processing
end

# case 文
case value
when condition1
  # 処理1
when condition2, condition3  # 複数条件
  # 処理2
else
  # デフォルト処理
end
```

**注意点**:

- Ruby では `false` と `nil` のみが偽、それ以外はすべて真
- `0` や空文字列 `""` も真として扱われる
- `case` 文は `===` 演算子を使用して判定

### 繰り返し

```ruby
# while ループ
while condition
  # 処理
end

# until ループ
until condition
  # 処理
end

# for ループ（イテレータの each を使うことが多い）
for item in collection
  # 処理
end

# イテレータ（推奨）
collection.each do |item|
  # 処理
end
```

**ループ制御**:

- `break`: ループを脱出（値を指定可能）
- `next`: 次の繰り返しへジャンプ
- `redo`: 同じ繰り返しをやり直す（条件チェックなし）
- `retry`: `rescue` 節内で `begin` からやり直す

## 例外処理のベストプラクティス

### 基本的な例外処理

```ruby
begin
  # 例外が発生する可能性のある処理
  risky_operation
rescue StandardError => e
  # StandardError とそのサブクラスを捕捉
  handle_error(e)
rescue SpecificError => e
  # 特定の例外を捕捉
  handle_specific_error(e)
else
  # 例外が発生しなかった場合の処理
  success_operation
ensure
  # 必ず実行される処理（リソース解放など）
  cleanup
end
```

**推奨事項**:

- `rescue` で例外クラスを省略すると `StandardError` のサブクラスをすべて捕捉
- 具体的な例外クラスを指定して、適切なエラーハンドリングを行う
- `ensure` 節はリソース解放に使用（ファイルクローズ、接続クローズなど）
- `ensure` 節の戻り値は無視される

### メソッド定義での例外処理

```ruby
def method_with_rescue
  # 処理
rescue => e
  # エラー処理
ensure
  # クリーンアップ
end
```

### rescue 修飾子

```ruby
# 簡潔な例外処理（StandardError のみ捕捉）
result = risky_operation rescue default_value

# ただし、優先順位に注意が必要
p((open("file") rescue false))  # 二重括弧が必要な場合がある
```

### 例外の発生

```ruby
# RuntimeError を発生
raise "エラーメッセージ"

# 特定の例外を発生
raise ArgumentError, "引数が不正です"
raise ArgumentError.new("引数が不正です")

# 例外の再発生
begin
  # 処理
rescue => e
  log_error(e)
  raise  # 同じ例外を再発生
end
```

## スレッド関連のベストプラクティス

### スレッドの基本

```ruby
# スレッドの生成
thread = Thread.new do
  # スレッドで実行する処理
end

# スレッドの終了を待つ
thread.join

# スレッド間の同期
mutex = Mutex.new
mutex.synchronize do
  # クリティカルセクション
  shared_resource.update
end
```

**注意事項**:

- Ruby VM は GVL（Giant VM Lock）を持ち、同時に実行されるスレッドは1つのみ
- IO 処理時は GVL が解放され、スレッドが並行実行される可能性がある
- スレッドで発生した例外は、そのスレッドを `join` している他のスレッドに伝播

### スレッドの例外処理

```ruby
# スレッドごとの例外処理フラグ
thread = Thread.new do
  # 処理
end
thread.abort_on_exception = true  # このスレッドの例外でプログラム全体を停止

# グローバル設定
Thread.abort_on_exception = true  # すべてのスレッドに適用
```

## エイリアスと定義の操作

### alias による別名定義

```ruby
class String
  # 既存メソッドの退避
  alias original_upcase upcase
  
  # メソッドの再定義（元の定義を利用）
  def upcase
    result = original_upcase
    puts "upcased: #{result}"
    result
  end
end
```

### undef による定義の取り消し

```ruby
class MyClass
  def old_method
    # 古い実装
  end
  
  # メソッドの取り消し
  undef old_method
  # または: undef :old_method
end
```

## 命名規則

### 変数とメソッド

- ローカル変数、メソッド名: `snake_case`
- インスタンス変数: `@instance_variable`
- クラス変数: `@@class_variable`
- グローバル変数: `$global_variable`
- 定数、クラス名、モジュール名: `CONSTANT`, `ClassName`, `ModuleName`（`CamelCase`）

### 真偽値を返すメソッド

```ruby
def empty?
  # 空かどうかを判定
end

def valid?
  # 有効かどうかを判定
end
```

**慣習**: 真偽値を返すメソッドには `?` を末尾に付ける

### 破壊的メソッド

```ruby
class Array
  def upcase
    map { |s| s.upcase }  # 新しい配列を返す
  end
  
  def upcase!
    map! { |s| s.upcase }  # 自身を変更
  end
end
```

**慣習**: オブジェクト自身を変更する破壊的メソッドには `!` を末尾に付ける

## よくある落とし穴

### ローカル変数のスコープ

```ruby
# for ループ
for i in 1..3
  x = i
end
p x  # => 3（for はスコープを作らない）

# イテレータ
[1, 2, 3].each do |i|
  y = i
end
# p y  # => NameError（ブロックは新しいスコープを作る）
```

### 複数の戻り値

```ruby
def multiple_values
  return 1, 2, 3  # 配列として返される
end

a, b, c = multiple_values  # => a=1, b=2, c=3
values = multiple_values   # => values=[1, 2, 3]
```

### メソッド定義のネスト

```ruby
class Example
  def outer
    def inner  # メソッド実行時に定義される
      "inner method"
    end
  end
end

obj = Example.new
# obj.inner  # => NoMethodError
obj.outer    # inner が定義される
obj.inner    # => "inner method"
```

**注意**: メソッド定義のネストは実行時に定義されるため、通常は避けるべき

## セキュリティとパフォーマンス

### 凍結（Freeze）

```ruby
# オブジェクトの変更を防止
str = "immutable".freeze
# str << " text"  # => FrozenError

# 定数の凍結
CONSTANT = [1, 2, 3].freeze
```

### スレッドセーフ

```ruby
# スレッドセーフな実装
class Counter
  def initialize
    @count = 0
    @mutex = Mutex.new
  end
  
  def increment
    @mutex.synchronize do
      @count += 1
    end
  end
  
  def value
    @mutex.synchronize { @count }
  end
end
```

## 終了処理

### END ブロック

```ruby
# プログラム終了時に実行される
END {
  puts "Cleanup"
  close_resources
}

# 複数の END ブロックは登録と逆順で実行される
END { puts "First registered" }
END { puts "Second registered" }
# 出力:
# Second registered
# First registered
```

### at_exit

```ruby
# より柔軟な終了処理（動的に登録可能）
at_exit do
  puts "Exiting..."
  cleanup_resources
end
```

## 参考リンク

- [Ruby 3.4 リファレンスマニュアル](https://docs.ruby-lang.org/ja/3.4/doc/index.html)
- [制御構造](https://docs.ruby-lang.org/ja/3.4/doc/spec/control.html)
- [クラス/メソッドの定義](https://docs.ruby-lang.org/ja/3.4/doc/spec/def.html)
- [スレッド](https://docs.ruby-lang.org/ja/3.4/doc/spec/thread.html)
- [組み込みライブラリ](https://docs.ruby-lang.org/ja/3.4/library/_builtin.html)
- [オブジェクト](https://docs.ruby-lang.org/ja/3.4/doc/spec/object.html)
