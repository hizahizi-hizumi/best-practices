---
description: 'Ruby 3.4 のコーディング規約とベストプラクティス'
applyTo: '**/*.rb, **/Gemfile, **/Rakefile, **/config.ru, **/*.rake'
---

# Ruby 3.4 ベストプラクティス

Ruby 3.4 でのコーディング規約、パフォーマンス最適化、セキュリティ実践を定義します。

## 目的とスコープ

本 instructions ファイルは Ruby 3.4 を使用するプロジェクトに適用され、公式ドキュメントとコミュニティの実践知を統合した実行可能なガイドラインを提供します。特に YJIT 有効化、Prism パーサー対応、Ruby 4.0 への移行準備を考慮します。

## 基本原則

- Ruby の動的型付けと純粋なオブジェクト指向特性を活用する
- 明示的で読みやすいコードを優先し、暗黙的な動作に依存しない
- パフォーマンスとメンテナンス性のバランスを取る
- YJIT の特性を理解し、最適化されたコードパターンを使用する
- Ruby 4.0 の frozen string literals デフォルト化に備える

## 命名規則

### 変数とメソッド名

- ローカル変数とメソッド名は `snake_case` を使用する
- インスタンス変数は `@` プレフィックスを付ける
- クラス変数は `@@` プレフィックスを使用（通常は避ける）
- グローバル変数は `$` プレフィックスを使用（通常は避ける）
- 定数、クラス名、モジュール名は `CamelCase` を使用する

**推奨**:

```ruby
class UserAccount
  MAX_LOGIN_ATTEMPTS = 3
  
  def initialize(user_name)
    @user_name = user_name
    @login_attempts = 0
  end
  
  def valid_credentials?(password)
    # 認証処理
  end
  
  def reset_password!
    # パスワードリセット
  end
end
```

**非推奨**:

```ruby
class userAccount  # クラス名は大文字で始める
  def userName  # メソッド名はsnake_caseを使用
    @UserName  # インスタンス変数は小文字で始める
  end
end
```

### メソッド名の慣習

- 真偽値を返すメソッドには `?` を末尾に付ける
- オブジェクト自身を変更する破壊的メソッドには `!` を末尾に付ける
- 設定用メソッド（セッター）には `=` を末尾に付ける

**推奨**:

```ruby
class Array
  def empty?
    size == 0
  end
  
  def clear!
    self.replace([])
    self
  end
end
```

## クラスとメソッドの定義

### クラス定義の基本

- クラス名は必ずアルファベットの大文字で始める
- スーパークラスを省略した場合は自動的に `Object` を継承
- `initialize` メソッドでインスタンス変数を初期化する
- 同じオブジェクト構造を一貫して維持する（YJIT 最適化のため）

**推奨**:

```ruby
class User
  def initialize(name:, email:, admin: false)
    @name = name
    @email = email
    @admin = admin
    @created_at = Time.now
  end
  
  def admin?
    @admin
  end
end
```

**根拠**: すべてのインスタンスで同じ構造を持つことで、YJIT がオブジェクトの形状を最適化でき、パフォーマンスが向上します。

**非推奨**:

```ruby
class User
  def initialize(attrs = {})
    @name = attrs[:name]
    @email = attrs[:email]
    @admin = attrs[:admin] if attrs.key?(:admin)  # 構造が不定
  end
end
```

### メソッド定義のベストプラクティス

- 引数の順序: 必須引数 → デフォルト引数 → `*`引数 → 後置必須引数 → キーワード引数 → `**`引数 → `&`引数
- デフォルト引数は連続した区間に配置する
- キーワード引数を使用して、引数の意味を明確にする

**推奨**:

```ruby
def create_user(name, email, role: 'user', permissions: [], &block)
  user = User.new(name: name, email: email, role: role)
  user.permissions = permissions
  block.call(user) if block
  user
end
```

**非推奨**:

```ruby
def create_user(name, email, role = 'user', permissions = [])
  # 引数が多い場合、何が何か分かりにくい
  User.new(name, email, role, permissions)
end
```

### イテレータの定義

- `yield` を使用してブロックを実行する（推奨）
- ブロックの存在を `block_given?` で確認する
- 複雑なブロック処理には明示的な `&block` 引数を使用する

**推奨**:

```ruby
def each_item
  return enum_for(__method__) unless block_given?
  
  @items.each do |item|
    yield item
  end
  self
end
```

**根拠**: `yield` は `Proc.new` や `&block` よりもパフォーマンスが高く、YJIT でさらに最適化されます。

## メソッドの可視性

- 外部から呼ぶ必要のないヘルパーメソッドは `private` にする
- 同じクラスのインスタンス間で比較演算などを行う場合は `protected` を使用する
- `initialize` は自動的に `private` になる

**推奨**:

```ruby
class Calculator
  def calculate(x, y, operation)
    case operation
    when :add then add(x, y)
    when :multiply then multiply(x, y)
    else raise ArgumentError, "Unknown operation: #{operation}"
    end
  end
  
  private
  
  def add(x, y)
    x + y
  end
  
  def multiply(x, y)
    x * y
  end
end
```

**根拠**: API を明確にし、内部実装の変更を容易にします。

## 制御構造

### 条件分岐

- `false` と `nil` のみが偽、それ以外（`0` や空文字列 `""` を含む）はすべて真
- 単純な条件には修飾子形式を使用する
- `case` 文は `===` 演算子を使用して判定する

**推奨**:

```ruby
# 単純な条件
return unless user.authenticated?

# 複雑な条件
if user.admin?
  grant_full_access
elsif user.moderator?
  grant_moderate_access
else
  grant_basic_access
end

# case 文
case response_code
when 200..299
  handle_success
when 400..499
  handle_client_error
when 500..599
  handle_server_error
else
  handle_unknown_error
end
```

**非推奨**:

```ruby
# 0 を偽として扱わない
if count == 0  # 明示的に比較する
  handle_empty
end

# C スタイルの条件式は避ける
result = condition ? value1 : value2  # 短いケースでは可
```

### ループと繰り返し

- イテレータ（`each`, `map`, `select` など）を優先的に使用する
- `for` ループはスコープを作らないため、通常は避ける
- ループ制御には `break`, `next`, `redo` を適切に使用する

**推奨**:

```ruby
users.each do |user|
  next unless user.active?
  process_user(user)
end

active_users = users.select(&:active?)

user_names = users.map(&:name)
```

**非推奨**:

```ruby
for user in users  # スコープを作らない
  x = user.name
end
puts x  # スコープ外で参照可能（意図しない動作）
```

## 例外処理

### 基本的な例外処理

- 具体的な例外クラスを指定して捕捉する
- `StandardError` とそのサブクラスのみを捕捉する（`Exception` は捕捉しない）
- `ensure` 節はリソース解放に使用する
- 例外を再発生させる場合は `raise` を引数なしで呼び出す

**推奨**:

```ruby
def process_file(filename)
  file = File.open(filename)
  process(file)
rescue Errno::ENOENT => e
  logger.error("File not found: #{filename}")
  raise
rescue StandardError => e
  logger.error("Processing error: #{e.message}")
  handle_error(e)
ensure
  file&.close
end
```

**根拠**: 具体的な例外を捕捉することで、予期しないエラーを見逃さず、デバッグが容易になります。

**非推奨**:

```ruby
begin
  risky_operation
rescue  # すべての StandardError を捕捉（広すぎる）
  # エラー処理
end

begin
  critical_operation
rescue Exception  # システム例外も捕捉（危険）
  # 絶対に避ける
end
```

### rescue 修飾子

- 単純なデフォルト値の提供にのみ使用する
- 優先順位に注意が必要な場合は括弧を使用する

**推奨**:

```ruby
config = load_config rescue DEFAULT_CONFIG

value = (JSON.parse(input) rescue nil)
```

**非推奨**:

```ruby
# 複雑な処理には rescue 修飾子を使わない
user = User.find_by(email: email) rescue User.create(email: email)  # 可読性が低い
```

## Ruby 3.4 新機能の活用

### デフォルトブロックパラメータ `it`

- 単純な一行ブロックには `it` を使用する
- 複雑なロジックや複数行の場合は明示的なパラメータ名を使用する
- `it` とローカル変数名の衝突に注意する

**推奨**:

```ruby
# 単純な変換
numbers.map { it ** 2 }

prices.select { it > 100 }

# 複雑な処理には明示的な名前を使用
users.map { |user|
  {
    name: user.name,
    email: user.email,
    role: user.role
  }
}
```

**根拠**: `it` は `_1` よりも読みやすく、Ruby コミュニティで広く推奨されています。

**非推奨**:

```ruby
# 番号付きパラメータ（読みにくい）
users.map { _1.name }

# it とローカル変数の衝突
it = current_item
items.each { it.process }  # 衝突の可能性
```

### Frozen String Literals への移行準備

- 変更可能な文字列が必要な場合は `+""` を使用する
- 文字列補間は自動的に変更可能な文字列を生成する
- 不要な `# frozen_string_literal: true` コメントは削除可能

**推奨**:

```ruby
# 変更可能な文字列の明示的な作成
buffer = +""
buffer << "some text"
buffer << "more text"

# 文字列補間（自動的に変更可能）
message = "Hello, #{name}!"
message << " Welcome!"

# 固定文字列（デフォルト）
constant_text = "This is immutable"
```

**根拠**: Ruby 4.0 では string literals がデフォルトで freeze されます。早期に対応することで、将来の移行がスムーズになります。

**非推奨**:

```ruby
# 警告が発生するパターン
text = "hello"
text << " world"  # Ruby 3.4 では警告（将来的にエラー）
```

### Range#size の互換性対応

- イテレート不可能な範囲には `#count` を使用する
- `Float` や `Time` の範囲には `#size` を使用しない

**推奨**:

```ruby
# イテレート可能な範囲
(1..10).size  #=> 10

# 文字列範囲
('a'..'z').count  #=> 26

# 日付範囲（ActiveSupport 使用時）
(Date.today..Date.today + 7).count  #=> 8
```

**非推奨**:

```ruby
(0.5..1.3).size  # TypeError: can't iterate from Float
(Time.now..Time.now + 1).size  # TypeError: can't iterate from Time
```

## YJIT パフォーマンス最適化

### YJIT 有効化の設定

- 本番環境では YJIT を有効化する
- 開発環境では無効化し、テスト環境では選択的に有効化する
- メモリリークを防ぐためにコード GC を有効化する

**推奨**:

```ruby
# config/application.rb
class Application < Rails::Application
  if RubyVM::YJIT.respond_to?(:enable)
    RubyVM::YJIT.enable unless Rails.env.test?

    if Rails.env.production?
      ENV['RUBY_YJIT_MIN_CALLS'] = '20'
      ENV['RUBY_YJIT_EXEC_MEM_SIZE'] = '512'  # MB
      ENV['RUBY_YJIT_CODE_GC'] = '1'
    end
  end
end
```

**根拠**: YJIT は本番環境で 15-30% のパフォーマンス向上をもたらしますが、追加のメモリを消費します。

### YJIT フレンドリーなコードパターン

- オブジェクト構造を一貫して保つ
- 動的メソッド定義は初期化時に完了させる
- メソッドチェーンを積極的に活用する

**推奨**:

```ruby
class User
  ATTRIBUTES = [:name, :email, :age].freeze

  # 初期化時に静的にメソッドを定義
  ATTRIBUTES.each do |attr|
    define_method(attr) do
      instance_variable_get("@#{attr}")
    end
    
    define_method("#{attr}=") do |value|
      instance_variable_set("@#{attr}", value)
    end
  end
  
  def initialize(attrs = {})
    @name = attrs[:name]
    @email = attrs[:email]
    @age = attrs[:age] || 0
    @created_at = Time.now
  end
end

# メソッドチェーンの活用
users = User.query
  .where(active: true)
  .order(:name)
  .limit(50)
  .to_a
```

**根拠**: 一貫したオブジェクト構造と静的なメソッド定義により、YJIT が効果的に最適化できます。

**非推奨**:

```ruby
class DynamicUser
  def initialize(attrs = {})
    @name = attrs[:name]
    # 条件付きインスタンス変数（構造が不定）
    @admin = attrs[:admin] if attrs.key?(:admin)
  end
  
  # 実行時の動的メソッド定義（deoptimization の原因）
  def add_attribute(name)
    define_singleton_method(name) do
      instance_variable_get("@#{name}")
    end
  end
end
```

### YJIT メモリ監視

- 長時間稼働するプロセスではメモリ使用量を監視する
- メモリ閾値を設定し、超過時にアラートを発する
- 定期的に `RubyVM::YJIT.runtime_stats` を確認する

**推奨**:

```ruby
class YjitMemoryMonitor
  MEMORY_THRESHOLD_MB = 2048

  def self.check_and_alert
    return unless RubyVM::YJIT.enabled?

    stats = RubyVM::YJIT.runtime_stats
    memory_mb = stats[:exec_mem_size] / 1024 / 1024

    if memory_mb > MEMORY_THRESHOLD_MB
      Rails.logger.warn(
        "YJIT memory high: #{memory_mb}MB, " \
        "compiled blocks: #{stats[:compiled_block_count]}"
      )
      GC.start if ENV['RUBY_YJIT_CODE_GC'] == '1'
    end
  end
end

# 定期実行
Thread.new do
  loop do
    sleep 3600
    YjitMemoryMonitor.check_and_alert
  end
end
```

## モジュールと Mix-in

### モジュールの適切な使用

- 関連するクラスをまとめる名前空間として使用する
- 共通機能を複数のクラスで共有する Mix-in として使用する
- 定数をグループ化するコンテナとして使用する

**推奨**:

```ruby
module Authentication
  module Strategies
    class Password
      def authenticate(credentials)
        # パスワード認証
      end
    end
    
    class OAuth
      def authenticate(token)
        # OAuth 認証
      end
    end
  end
  
  def self.authenticate(strategy, credentials)
    strategy_class = Strategies.const_get(strategy.to_s.capitalize)
    strategy_class.new.authenticate(credentials)
  end
end

# Mix-in の使用
module Loggable
  def log_info(message)
    logger.info("#{self.class.name}: #{message}")
  end
end

class User
  include Loggable
  
  def create
    # ユーザー作成処理
    log_info("User created: #{email}")
  end
end
```

**根拠**: モジュールを使用することで、コードの再利用性と保守性が向上します。

### include と extend の使い分け

- `include` はインスタンスメソッドとして追加する
- `extend` はクラスメソッドとして追加する
- `prepend` は既存メソッドの前に挿入する

**推奨**:

```ruby
module Timestampable
  def created_at
    @created_at ||= Time.now
  end
end

class Model
  include Timestampable  # インスタンスメソッドとして追加
end

module ClassMethods
  def find_by_email(email)
    # クラスメソッドの実装
  end
end

class User
  extend ClassMethods  # クラスメソッドとして追加
end
```

## スレッドとマルチプロセス

### スレッドセーフな実装

- 共有リソースへのアクセスは `Mutex` で保護する
- スレッドローカル変数を活用する
- IO 処理には積極的にスレッドを使用する

**推奨**:

```ruby
class ThreadSafeCounter
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

# スレッドローカル変数の使用
Thread.current[:user] = current_user

# 並行 IO 処理
threads = urls.map do |url|
  Thread.new { fetch_data(url) }
end
results = threads.map(&:join).map(&:value)
```

**根拠**: Ruby VM は GVL（Giant VM Lock）を持ちますが、IO 処理時は GVL が解放され、スレッドが並行実行されます。

**非推奨**:

```ruby
# Mutex なしの共有リソースアクセス
class UnsafeCounter
  def initialize
    @count = 0
  end
  
  def increment
    @count += 1  # スレッドセーフではない
  end
end
```

## セキュリティのベストプラクティス

### オブジェクトの凍結

- 変更されるべきでない定数やオブジェクトは `freeze` を使用する
- 定数に代入する配列やハッシュは凍結する

**推奨**:

```ruby
ALLOWED_ROLES = ['admin', 'user', 'guest'].freeze

DEFAULT_CONFIG = {
  timeout: 30,
  retries: 3
}.freeze

class Configuration
  def initialize
    @settings = {}.freeze
  end
end
```

**根拠**: 凍結されたオブジェクトは変更できないため、予期しない変更を防ぎます。

### 入力検証

- 外部入力は必ず検証する
- ホワイトリスト方式を使用する
- SQL インジェクションや XSS を防ぐ

**推奨**:

```ruby
def create_user(params)
  allowed_params = params.slice(:name, :email, :role)
  role = allowed_params[:role]
  
  unless ALLOWED_ROLES.include?(role)
    raise ArgumentError, "Invalid role: #{role}"
  end
  
  User.create!(allowed_params)
end
```

**非推奨**:

```ruby
def create_user(params)
  User.create!(params)  # すべてのパラメータを受け入れる（危険）
end
```

## よくある落とし穴

### ローカル変数のスコープ

- `for` ループはスコープを作らないため、イテレータを使用する
- ブロックは新しいスコープを作る

**推奨**:

```ruby
[1, 2, 3].each do |i|
  temp = i * 2
  puts temp
end
# temp はここでは未定義
```

**非推奨**:

```ruby
for i in [1, 2, 3]
  temp = i * 2
  puts temp
end
puts temp  # => 6（意図しないスコープ漏れ）
```

### メソッド定義のネスト

- メソッド内でのメソッド定義は実行時に定義されるため避ける

**非推奨**:

```ruby
class Example
  def outer
    def inner  # メソッド実行時に定義される
      "inner method"
    end
  end
end

obj = Example.new
obj.inner  # => NoMethodError
obj.outer
obj.inner  # => "inner method"（意図しない動作）
```

### キーワード引数への nil の展開

- Ruby 3.4 では `**nil` が空のキーワード引数として扱われる

**推奨**:

```ruby
def handle_options(**kwargs)
  p kwargs
end

extra_options = some_condition? ? {foo: 1} : nil
handle_options(**extra_options)
# some_condition が false の場合、{} が渡される
```

## パフォーマンスの考慮事項

### メソッドチェーン

- メソッドチェーンは YJIT で効果的に最適化される
- 中間オブジェクトの生成を避ける場合は `lazy` を使用する

**推奨**:

```ruby
# YJIT で最適化されるパターン
result = collection
  .select { |item| item.active? }
  .map { |item| item.name }
  .sort
  .first(10)

# 大規模データには lazy を使用
large_collection
  .lazy
  .select { |item| item.active? }
  .map { |item| item.process }
  .first(10)
```

### 文字列操作

- 大量の文字列連結には `String#<<` または配列の `join` を使用する
- 文字列補間を優先する

**推奨**:

```ruby
# 文字列補間
message = "Hello, #{user.name}! Welcome to #{site_name}."

# 大量の連結
buffer = +""
items.each do |item|
  buffer << item.to_s
  buffer << "\n"
end

# または
result = items.map(&:to_s).join("\n")
```

**非推奨**:

```ruby
# 繰り返しの文字列連結（非効率）
message = ""
items.each do |item|
  message = message + item.to_s + "\n"  # 毎回新しい文字列を生成
end
```

## ガベージコレクションの最適化

### GC 設定の調整

- メモリが十分にある環境では Major GC を制御する
- レスポンスタイム優先の場合は GC 頻度を調整する

**推奨**:

```ruby
# Major GC の制御（レスポンスタイム優先）
GC.config(rgengc_allow_full_mark: false)

# 設定の確認
current_config = GC.config
Rails.logger.info("GC config: #{current_config}")
```

**根拠**: レスポンスタイムを最小化する必要がある API サーバーなどで有効です。

## 終了処理

### END ブロックと at_exit

- プログラム終了時の処理には `at_exit` を使用する
- `END` ブロックは登録と逆順で実行される

**推奨**:

```ruby
at_exit do
  logger.info("Application shutting down")
  cleanup_resources
  close_connections
end
```

**根拠**: `at_exit` は動的に登録でき、より柔軟な終了処理が可能です。

## 参考資料

### 公式ドキュメント

- [Ruby 3.4 リファレンスマニュアル](https://docs.ruby-lang.org/ja/3.4/doc/index.html)
- [Ruby 3.4.0 リリースノート](https://www.ruby-lang.org/en/news/2024/12/25/ruby-3-4-0-released/)
- [Ruby 3.4 NEWS.md](https://github.com/ruby/ruby/blob/v3_4_0/NEWS.md)

### コミュニティリソース

- [Ruby 3.4 YJIT Performance Guide - JetThoughts](https://jetthoughts.com/blog/ruby-3-4-yjit-performance-guide/)
- [What's New in Ruby 3.4 - Honeybadger](https://www.honeybadger.io/blog/ruby-3-4/)
- [Ruby 3.4 Changes（詳細版）](https://rubyreferences.github.io/rubychanges/3.4.html)

### プロダクション事例

- [Shopify YJIT Development Blog](https://shopify.engineering/yjit-just-in-time-compiler-cruby)
- Ruby 3.4 により Shopify は 18% のレスポンスタイム改善、GitHub は 24% のパフォーマンス向上を達成

### 内部リソース

- `docs/best_practice/Ruby3.4/official.md`: 公式ドキュメントから収集したベストプラクティス
- `docs/best_practice/Ruby3.4/community.md`: コミュニティの実践知と事例
