# RSpec 公式ドキュメント ベストプラクティス

## 概要

RSpecは、Ruby用のBDD（振る舞い駆動開発）テストフレームワークです。2005年にSteven Bakerによって開始され、現在も活発に開発が続けられています。RSpecは以下の4つの主要なライブラリで構成されています：

- **rspec-core**: テスト実行エンジン。柔軟な設定とレポート機能を提供
- **rspec-expectations**: 期待値を表現するための読みやすいAPI
- **rspec-mocks**: テストダブル（モック）フレームワーク
- **rspec-rails**: Ruby on Railsアプリケーション向けの統合サポート

RSpecは、Semantic Versioningに従った公開APIのみを使用することが推奨されており、プライベートAPIは予告なく変更される可能性があります。

## 推奨設定

### 基本設定

`.rspec`ファイルで基本的なコマンドラインオプションを設定できます：

```text
--require spec_helper
--color
--format documentation
```

`spec_helper.rb`での推奨設定：

```ruby
RSpec.configure do |config|
  # 期待値構文の設定（expect構文のみを使用）
  config.expect_with :rspec do |expectations|
    expectations.syntax = :expect
  end

  # モック構文の設定
  config.mock_with :rspec do |mocks|
    mocks.syntax = :expect
    mocks.verify_partial_doubles = true
  end

  # テストの実行順序をランダム化
  config.order = :random
  
  # ランダム実行のシードを表示
  Kernel.srand config.seed
end
```

### ゼロモンキーパッチモード

RSpecの機能をグローバルな名前空間に追加せず、明示的に使用する設定：

```ruby
RSpec.configure do |config|
  config.expose_dsl_globally = false
end
```

このモードでは、明示的にモジュールをインクルードする必要があります：

```ruby
require 'rspec'

class MySpec
  include RSpec::Matchers
  
  def check_value
    expect(5).to eq(5)
  end
end
```

### fail_fastオプション

最初の失敗で即座にテストを中断する設定：

```ruby
RSpec.configure do |config|
  config.fail_fast = true
  # または失敗数を指定
  config.fail_fast = 3
end
```

コマンドラインでも指定可能：

```bash
rspec --fail-fast
```

### 実行順序とシード値の設定

テストの実行順序を制御：

```ruby
RSpec.configure do |config|
  # ランダム実行（推奨）
  config.order = :random
  
  # 定義順
  config.order = :defined
  
  # 特定のシード値でランダム実行を再現
  config.order = :random
  config.seed = 1234
end
```

コマンドラインでの指定：

```bash
rspec --order random
rspec --seed 1234
```

### プロファイリング

最も遅いテストを特定するための設定：

```ruby
RSpec.configure do |config|
  config.profile_examples = 10
end
```

コマンドラインでも指定可能：

```bash
rspec --profile 10
```

## テスト構造のベストプラクティス

### describe と context の使用

```ruby
RSpec.describe Account do
  # クラスやメソッドの説明にはdescribeを使用
  describe "#balance" do
    # 条件や状態の説明にはcontextを使用
    context "when account is newly created" do
      it "has a balance of zero" do
        account = Account.new
        expect(account.balance).to eq(0)
      end
    end
    
    context "after deposit" do
      it "increases the balance" do
        account = Account.new
        account.deposit(100)
        expect(account.balance).to eq(100)
      end
    end
  end
end
```

### subject の活用

テスト対象を明示的に定義：

```ruby
RSpec.describe Account do
  subject(:account) { Account.new }
  
  it "has zero balance initially" do
    expect(account.balance).to eq(0)
  end
  
  # 一行テスト（subjectを暗黙的に使用）
  it { is_expected.to respond_to(:deposit) }
end
```

### let と let! の使用

```ruby
RSpec.describe User do
  # 遅延評価（最初に参照されたときに評価）
  let(:user) { User.new(name: "John") }
  
  # 即時評価（各テストの前に必ず実行）
  let!(:admin) { User.create(name: "Admin", role: :admin) }
  
  it "creates user on demand" do
    expect(user).not_to be_persisted
  end
  
  it "ensures admin exists" do
    expect(User.count).to eq(1)
  end
end
```

### フックの使用

```ruby
RSpec.describe Database do
  # 全テストの前に一度だけ実行
  before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end
  
  # 各コンテキストの前に一度だけ実行
  before(:context) do
    @connection = Database.connect
  end
  
  # 各テストの前に実行
  before(:example) do
    Database.start_transaction
  end
  
  # 各テストの後に実行
  after(:example) do
    Database.rollback_transaction
  end
  
  # aroundフックでテストをラップ
  around(:example) do |example|
    Database.transaction do
      example.run
      raise ActiveRecord::Rollback
    end
  end
end
```

## Expectations（期待値）のベストプラクティス

### expect構文の使用

RSpec 3以降は`expect`構文を使用することが推奨されています：

```ruby
# 推奨
expect(actual).to eq(expected)
expect(actual).not_to eq(expected)

# 非推奨（should構文）
actual.should eq(expected)
actual.should_not eq(expected)
```

### 組み込みマッチャーの活用

#### 等価性チェック

```ruby
# オブジェクトの等価性（==）
expect(actual).to eq(expected)

# オブジェクトの同一性（equal?）
expect(actual).to be(expected)
expect(actual).to equal(expected)

# eql?を使用した等価性
expect(actual).to eql(expected)
```

#### 比較

```ruby
expect(actual).to be > expected
expect(actual).to be >= expected
expect(actual).to be < expected
expect(actual).to be <= expected
expect(actual).to be_between(min, max).inclusive
expect(actual).to be_between(min, max).exclusive
expect(actual).to be_within(delta).of(expected)
```

#### 型チェック

```ruby
expect(actual).to be_instance_of(Class)
expect(actual).to be_kind_of(Class)
expect(actual).to respond_to(:method_name)
```

#### 真偽値

```ruby
expect(actual).to be_truthy    # nil と false 以外
expect(actual).to be true      # true と厳密に等しい
expect(actual).to be_falsey    # nil または false
expect(actual).to be false     # false と厳密に等しい
expect(actual).to be_nil       # nil
```

#### コレクション

```ruby
expect(array).to include(element)
expect(array).to contain_exactly(1, 2, 3)  # 順不同で完全一致
expect(array).to match_array([3, 2, 1])    # contain_exactly のエイリアス
expect(string).to include("substring")
expect(hash).to include(key: value)
```

#### 例外とエラー

```ruby
expect { raise_error_code }.to raise_error
expect { raise_error_code }.to raise_error(ErrorClass)
expect { raise_error_code }.to raise_error(ErrorClass, "message")
expect { raise_error_code }.to raise_error("message")
```

#### 変更の検証

```ruby
expect { action }.to change { object.value }.from(old).to(new)
expect { action }.to change { object.value }.by(delta)
expect { action }.to change { object.value }.by_at_least(min)
expect { action }.to change { object.value }.by_at_most(max)

# 実例
expect { user.activate! }.to change { user.active? }.from(false).to(true)
expect { cart.add_item(item) }.to change { cart.items.count }.by(1)
```

#### ブロックの振る舞い

```ruby
expect { |block| object.method(&block) }.to yield_control
expect { |block| object.method(&block) }.to yield_with_no_args
expect { |block| object.method(&block) }.to yield_with_args(1, 2)
expect { |block| [1, 2].each(&block) }.to yield_successive_args(1, 2)
```

#### 出力の検証

```ruby
expect { print "hello" }.to output("hello").to_stdout
expect { warn "error" }.to output("error\n").to_stderr
expect { puts "test" }.to output(/test/).to_stdout
```

### 述語マッチャー

メソッド名が`?`で終わる場合、自動的にマッチャーが生成されます：

```ruby
expect([]).to be_empty           # [].empty? が true
expect(user).to be_admin         # user.admin? が true
expect(hash).to have_key(:name)  # hash.has_key?(:name) が true
```

### マッチャーの合成

RSpec 3で導入された強力な機能で、マッチャーを組み合わせて複雑な期待値を表現できます：

```ruby
# ネストした構造の検証
expect(hash).to match(
  name: a_string_matching(/John/),
  age: a_value_between(20, 30),
  email: a_string_ending_with("@example.com")
)

# コレクション内の要素の検証
expect(users).to contain_exactly(
  an_object_having_attributes(name: "Alice", role: :admin),
  an_object_having_attributes(name: "Bob", role: :user)
)

# 複数のマッチャーの組み合わせ
expect(response).to start_with("Success").and end_with("OK")
expect(value).to be_positive.or be_zero
```

マッチャーのエイリアス（名詞形）：

```ruby
be < 2           => a_value < 2
be_instance_of   => an_instance_of
be_within        => a_value_within
contain_exactly  => a_collection_containing_exactly
start_with       => a_string_starting_with
end_with         => a_string_ending_with
match            => a_string_matching
```

### カスタムマッチャーの定義

```ruby
RSpec::Matchers.define :be_a_multiple_of do |expected|
  match do |actual|
    actual % expected == 0
  end
  
  failure_message do |actual|
    "expected #{actual} to be a multiple of #{expected}"
  end
  
  failure_message_when_negated do |actual|
    "expected #{actual} not to be a multiple of #{expected}"
  end
  
  description do
    "be a multiple of #{expected}"
  end
end

# 使用例
expect(9).to be_a_multiple_of(3)
```

### 複合期待値

```ruby
# AND条件
expect(alphabet).to start_with("a").and end_with("z")

# OR条件
expect(stoplight.color).to eq("red").or eq("green").or eq("yellow")
```

### 失敗の集約

複数の期待値をまとめて検証し、すべての失敗を報告：

```ruby
aggregate_failures "user attributes" do
  expect(user.name).to eq("John")
  expect(user.email).to eq("john@example.com")
  expect(user.age).to eq(30)
end
```

## Mocks（モック）のベストプラクティス

### テストダブルの基本

```ruby
# シンプルなダブル
book = double("book")

# メソッドスタブ付きのダブル
book = double("book", title: "RSpec Book", pages: 250)

# メッセージの許可
allow(book).to receive(:title).and_return("RSpec Book")
allow(book).to receive(:pages).and_return(250)

# メッセージの期待
expect(logger).to receive(:log).with("User logged in")
user.login(logger)
```

### 検証ダブルの使用（推奨）

通常のダブルよりも厳格で、実際のオブジェクトに存在しないメソッドのスタブを防ぎます：

```ruby
# インスタンスダブル（推奨）
book = instance_double("Book", title: "RSpec Book", pages: 250)

# クラスダブル
book_class = class_double("Book").as_stubbed_const

# オブジェクトダブル
book = object_double(Book.new, title: "RSpec Book")
```

検証ダブルの利点：

- 実際のクラスに存在しないメソッドをスタブするとエラーになる
- メソッドの引数の数が正しいかチェックされる
- テストを依存関係なしで実行できる
- 実際のオブジェクトが利用可能な場合は検証される

### パーシャルダブル

実際のオブジェクトの一部のメソッドだけをスタブ：

```ruby
user = User.new
allow(user).to receive(:valid?).and_return(true)

# 実際のメソッドは通常通り動作
user.name = "John"
expect(user.name).to eq("John")

# スタブされたメソッドはモックの振る舞い
expect(user.valid?).to be true
```

### スパイ

メソッドが呼ばれた後に検証を行う：

```ruby
logger = spy("logger")

# アクションを実行
user.login(logger)

# 後で検証
expect(logger).to have_received(:log).with("User logged in")
```

### Null Object パターン

未定義のメッセージに対して自身を返すダブル：

```ruby
dbl = double("null object").as_null_object

dbl.foo        # nil の代わりに dbl を返す
dbl.foo.bar    # チェーンが可能
dbl.foo.bar.baz
```

### メッセージの制約

```ruby
# 引数の制約
expect(logger).to receive(:log).with("message")
expect(calculator).to receive(:add).with(1, 2)

# 任意の引数
expect(logger).to receive(:log).with(anything)
expect(logger).to receive(:log).with(a_kind_of(String))
expect(logger).to receive(:log).with(a_string_matching(/error/))

# 呼び出し回数の制約
expect(logger).to receive(:log).once
expect(logger).to receive(:log).twice
expect(logger).to receive(:log).exactly(3).times
expect(logger).to receive(:log).at_least(2).times
expect(logger).to receive(:log).at_most(5).times
```

### 複雑なレスポンスの設定

```ruby
# 複数の戻り値
allow(die).to receive(:roll).and_return(1, 2, 3)

# ブロックでの実装
allow(calc).to receive(:add) { |a, b| a + b }

# 例外を発生
allow(api).to receive(:fetch).and_raise(NetworkError)

# 元のメソッドを呼び出す
allow(user).to receive(:save).and_call_original

# yieldする
allow(file).to receive(:open).and_yield(file_content)
```

### 定数のスタブ

```ruby
# 定数を一時的に変更
stub_const("MyClass::CONSTANT", "new value")

# クラス自体を置き換え
stub_const("MyClass", class_double("MyClass"))
```

## RSpec Rails のベストプラクティス

### インストールと設定

Gemfileに追加：

```ruby
group :test, :development do
  gem 'rspec-rails', '~> 7.0.0'
end
```

インストール：

```bash
bundle exec rails generate rspec:install
```

これにより以下が作成されます：

- `.rspec` - コマンドラインオプション
- `spec/` - テストディレクトリ
- `spec/spec_helper.rb` - RSpec設定
- `spec/rails_helper.rb` - Rails固有の設定

### ディレクトリ構造

```
spec/
├── rails_helper.rb
├── spec_helper.rb
├── models/
├── controllers/
├── requests/       # 推奨（統合テスト）
├── features/       # Capybara使用時
├── system/         # システムテスト（推奨）
├── views/
├── helpers/
├── mailers/
├── routing/
├── jobs/
└── support/        # 共有コード
```

### モデルスペック

```ruby
RSpec.describe User, type: :model do
  subject(:user) { described_class.new(email: "test@example.com") }
  
  describe "validations" do
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_uniqueness_of(:email) }
  end
  
  describe "associations" do
    it { is_expected.to have_many(:posts) }
    it { is_expected.to belong_to(:organization) }
  end
  
  describe "#full_name" do
    it "combines first and last name" do
      user.first_name = "John"
      user.last_name = "Doe"
      expect(user.full_name).to eq("John Doe")
    end
  end
end
```

### リクエストスペック（推奨）

コントローラースペックの代わりにリクエストスペックを使用することが推奨されています：

```ruby
RSpec.describe "Users", type: :request do
  describe "GET /users" do
    it "returns a successful response" do
      get users_path
      expect(response).to have_http_status(:success)
    end
    
    it "returns JSON with user data" do
      user = create(:user, name: "John")
      get users_path, as: :json
      
      expect(response).to have_http_status(:ok)
      expect(response.content_type).to match(%r{application/json})
      json = JSON.parse(response.body)
      expect(json.first["name"]).to eq("John")
    end
  end
  
  describe "POST /users" do
    context "with valid parameters" do
      it "creates a new user" do
        expect {
          post users_path, params: { user: { name: "John", email: "john@example.com" } }
        }.to change(User, :count).by(1)
      end
      
      it "redirects to the created user" do
        post users_path, params: { user: { name: "John", email: "john@example.com" } }
        expect(response).to redirect_to(User.last)
      end
    end
    
    context "with invalid parameters" do
      it "does not create a new user" do
        expect {
          post users_path, params: { user: { name: "" } }
        }.not_to change(User, :count)
      end
      
      it "returns unprocessable entity status" do
        post users_path, params: { user: { name: "" } }
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
end
```

### システムスペック

ブラウザを使った統合テスト：

```ruby
RSpec.describe "User registration", type: :system do
  before do
    driven_by(:rack_test)
  end
  
  it "allows a user to sign up" do
    visit new_user_registration_path
    
    fill_in "Email", with: "user@example.com"
    fill_in "Password", with: "password123"
    fill_in "Password confirmation", with: "password123"
    
    click_button "Sign up"
    
    expect(page).to have_content("Welcome!")
    expect(page).to have_current_path(root_path)
  end
end
```

JavaScriptを使用するテスト：

```ruby
RSpec.describe "Interactive features", type: :system, js: true do
  before do
    driven_by(:selenium_chrome_headless)
  end
  
  it "updates content dynamically" do
    visit dashboard_path
    click_button "Load Data"
    
    expect(page).to have_content("Data loaded")
  end
end
```

### Railsマッチャー

```ruby
# ルーティング
expect(get: "/users").to route_to("users#index")
expect(get: "/users/1").to route_to("users#show", id: "1")

# レスポンス
expect(response).to have_http_status(:success)
expect(response).to have_http_status(200)
expect(response).to redirect_to(users_path)
expect(response).to render_template(:index)

# ビュー
expect(rendered).to match(/Some text/)
expect(rendered).to have_selector("div.alert")
expect(rendered).to have_link("Profile", href: profile_path)
```

## パフォーマンスとベストプラクティス

### テストの高速化

1. **データベーストランザクション**

```ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = true
end
```

2. **遅延読み込み**

```ruby
# let は遅延評価される
let(:user) { create(:user) }

# 必要な場合のみ let! を使用
let!(:admin) { create(:admin) }
```

3. **FactoryBot の戦略**

```ruby
# build を優先（DBに保存しない）
let(:user) { build(:user) }

# 必要な場合のみ create
let(:user) { create(:user) }

# build_stubbed は最速（ARオブジェクトのスタブ）
let(:user) { build_stubbed(:user) }
```

4. **フィルタリングの活用**

```ruby
# 特定のタグのみ実行
rspec --tag focus
rspec --tag ~slow

# タグの設定
it "runs fast", :focus do
  # ...
end

it "runs slow", :slow do
  # ...
end
```

### テストの独立性

各テストは独立して実行できるようにします：

```ruby
# 悪い例
let(:counter) { 0 }

it "first test" do
  counter += 1
  expect(counter).to eq(1)
end

it "second test" do # これは失敗する可能性がある
  expect(counter).to eq(0)
end

# 良い例
it "increments counter" do
  counter = 0
  counter += 1
  expect(counter).to eq(1)
end

it "starts at zero" do
  counter = 0
  expect(counter).to eq(0)
end
```

### DRY原則（Don't Repeat Yourself）

```ruby
# 共有コンテキスト
RSpec.shared_context "authenticated user" do
  let(:user) { create(:user) }
  
  before do
    sign_in user
  end
end

# 使用
RSpec.describe "Dashboard", type: :request do
  include_context "authenticated user"
  
  it "shows user dashboard" do
    get dashboard_path
    expect(response).to be_successful
  end
end

# 共有例
RSpec.shared_examples "a sortable collection" do
  it "sorts by name" do
    expect(collection.sort_by_name).to be_sorted
  end
end

# 使用
RSpec.describe Product do
  it_behaves_like "a sortable collection" do
    let(:collection) { Product.all }
  end
end
```

## よくある落とし穴

### 1. should構文の使用

```ruby
# 悪い例（非推奨）
result.should eq(5)
result.should_not be_nil

# 良い例
expect(result).to eq(5)
expect(result).not_to be_nil
```

### 2. テストダブルの過度な使用

```ruby
# 悪い例（過度なモック）
allow(user).to receive(:name).and_return("John")
allow(user).to receive(:email).and_return("john@example.com")
allow(user).to receive(:age).and_return(30)

# 良い例（実際のオブジェクトを使用）
user = User.new(name: "John", email: "john@example.com", age: 30)
```

### 3. letとlet!の混同

```ruby
# let は遅延評価（参照されるまで実行されない）
let(:user) { create(:user) }

# let! は即時評価（各テストの前に必ず実行）
let!(:user) { create(:user) }

# 落とし穴
describe "User count" do
  let(:user) { create(:user) }
  
  it "has users" do
    # user が参照されないため、作成されない
    expect(User.count).to eq(0)  # これはパスする
  end
end
```

### 4. before(:all) の使用

```ruby
# 危険（トランザクションの外で実行される）
before(:all) do
  @user = User.create(name: "John")
end

# 推奨
before(:each) do
  @user = User.create(name: "John")
end

# またはletを使用
let(:user) { create(:user, name: "John") }
```

### 5. 実装の詳細のテスト

```ruby
# 悪い例（内部実装に依存）
it "calls calculate_total" do
  expect(order).to receive(:calculate_total)
  order.process
end

# 良い例（振る舞いをテスト）
it "updates the total amount" do
  order.process
  expect(order.total).to eq(100)
end
```

### 6. 一つのテストで複数のことをテストする

```ruby
# 悪い例
it "creates and validates user" do
  user = User.create(name: "John", email: "john@example.com")
  expect(user).to be_valid
  expect(user.name).to eq("John")
  expect(user.email).to eq("john@example.com")
  expect(User.count).to eq(1)
end

# 良い例
describe "User creation" do
  let(:user) { User.create(name: "John", email: "john@example.com") }
  
  it "is valid with valid attributes" do
    expect(user).to be_valid
  end
  
  it "saves the name" do
    expect(user.name).to eq("John")
  end
  
  it "saves the email" do
    expect(user.email).to eq("john@example.com")
  end
  
  it "increments the user count" do
    expect { user }.to change(User, :count).by(1)
  end
end
```

### 7. raise_errorの不適切な使用

```ruby
# 悪い例（どんなエラーでもパス）
expect { risky_operation }.to raise_error

# 良い例（特定のエラーを指定）
expect { risky_operation }.to raise_error(SpecificError)
expect { risky_operation }.to raise_error(SpecificError, /error message/)
```

### 8. 検証ダブルを使わない

```ruby
# 悪い例（実際に存在しないメソッドでもエラーにならない）
dbl = double("Book")
allow(dbl).to receive(:non_existent_method)

# 良い例（存在しないメソッドだとエラーになる）
dbl = instance_double("Book")
allow(dbl).to receive(:title)  # Book#title が存在する必要がある
```

## セキュリティ

### 機密情報の取り扱い

```ruby
# 環境変数を使用
ENV['API_KEY'] = 'test_key' if Rails.env.test?

# 本番環境の認証情報を使用しない
RSpec.describe ApiClient do
  let(:client) { ApiClient.new(api_key: 'test_key') }
  
  it "authenticates successfully" do
    # テスト用のキーを使用
  end
end
```

### スタブの検証

```ruby
# セキュリティ関連のメソッドは実際の振る舞いをテスト
describe "Authentication" do
  it "encrypts password" do
    user = User.new(password: "secret")
    # 実際の暗号化をテスト
    expect(user.encrypted_password).not_to eq("secret")
    expect(user.encrypted_password).to be_present
  end
end
```

## 参考リンク

- [RSpec 公式サイト](https://rspec.info/)
- [RSpec Core ドキュメント](https://rspec.info/features/3-13/rspec-core/)
- [RSpec Expectations ドキュメント](https://rspec.info/features/3-13/rspec-expectations/)
- [RSpec Mocks ドキュメント](https://rspec.info/features/3-13/rspec-mocks/)
- [RSpec Rails ドキュメント](https://rspec.info/features/8-0/rspec-rails/)
- [Better Specs (コミュニティベストプラクティス)](http://www.betterspecs.org/)
- [RSpec新しい期待値構文について](https://rspec.info/blog/2012/06/rspecs-new-expectation-syntax/)
- [RSpec 3の主な変更点](https://rspec.info/blog/2014/05/notable-changes-in-rspec-3/)
- [合成可能なマッチャー](https://rspec.info/blog/2014/01/new-in-rspec-3-composable-matchers/)
