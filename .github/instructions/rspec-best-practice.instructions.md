---
description: 'Ruby RSpecを使用したBDDテストのベストプラクティス'
applyTo: '**/*_spec.rb, **/spec/**/*.rb'
---

# RSpec ベストプラクティス

RSpecを使用したBDD（振る舞い駆動開発）テストの実装において、可読性、保守性、パフォーマンスを重視した指針を提供します。

## 目的とスコープ

このインストラクションは、RSpecを使用したテストコードの品質向上を目的とします。テストの構造化、期待値の表現、モックの使用方法、パフォーマンス最適化など、実践的なベストプラクティスを網羅します。

**適用範囲**: RSpecテストファイル（`*_spec.rb`）およびテスト支援ファイル（`spec/support/`内のファイル）

## 基本原則

- expect構文を使用し、should構文は使用しない
- 各テストを独立して実行可能にする
- テスト対象の振る舞いをテストし、実装の詳細には依存しない
- describeはクラス/メソッドの説明に、contextは条件/状態の説明に使用する
- 一つのitブロックで一つの概念のみをテストする
- 遅延評価のletを優先し、let!は必要な場合のみ使用する

## 設定とセットアップ

### 推奨設定

`spec/spec_helper.rb`で以下の設定を行う:

```ruby
RSpec.configure do |config|
  # expect構文のみを使用
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
  Kernel.srand config.seed
  
  # 失敗したテストの状態を保存
  config.example_status_persistence_file_path = "spec/examples.txt"
  
  # focusタグで特定のテストのみ実行（開発時）
  config.filter_run_when_matching :focus
end
```

**根拠**: ランダム実行により、テスト間の依存関係を早期に検出できます。期待値構文の統一により、コードの一貫性が向上します。

### ディレクトリ構造

実装コードの構造と一致させる:

```
spec/
├── spec_helper.rb
├── rails_helper.rb      # Rails使用時
├── models/
│   └── user_spec.rb     # app/models/user.rb に対応
├── requests/            # 統合テスト（推奨）
├── system/              # E2Eテスト
└── support/             # 共有コード
    ├── shared_contexts/
    └── shared_examples/
```

**根拠**: 実装ファイルとテストファイルの対応関係が明確になり、テストの検索が容易になります。

## テスト構造

### describe、context、itの使用

**推奨**:
```ruby
RSpec.describe Article do
  subject(:article) { described_class.new }
  
  # クラスメソッドには.を使用
  describe '.published' do
    it 'returns published articles' do
      # テスト内容
    end
  end
  
  # インスタンスメソッドには#を使用
  describe '#summary' do
    context 'when summary is present' do
      it 'returns the summary' do
        # テスト内容
      end
    end
    
    context 'when summary is not present' do
      it 'returns a default message' do
        # テスト内容
      end
    end
  end
end
```

**非推奨**:
```ruby
RSpec.describe Article do
  # 対になるcontextがない（リファクタリングが必要なコードの臭い）
  describe '#summary' do
    context 'when summary is present' do
      it 'returns the summary' do
        # テスト内容
      end
    end
  end
end
```

**根拠**: contextには対になる反対のケースが必要です。単一のcontextのみの場合、条件分岐をテストする意味がありません。

### subjectの活用

**推奨**:
```ruby
RSpec.describe User do
  # 名前付きsubjectで可読性向上
  subject(:user) { User.new(name: "John") }
  
  it "has the correct name" do
    expect(user.name).to eq("John")
  end
  
  # 一行テスト
  it { is_expected.to respond_to(:email) }
end
```

**非推奨**:
```ruby
RSpec.describe User do
  subject { User.new(name: "John") }
  
  it "has the correct name" do
    # subjectという汎用的な名前では意図が不明確
    expect(subject.name).to eq("John")
  end
end
```

**根拠**: 名前付きsubjectにより、テストの意図が明確になり、可読性が向上します。

### letとlet!の使用

**推奨**:
```ruby
RSpec.describe Project do
  # 遅延評価を活用（参照されたときのみ作成）
  let(:user) { create(:user) }
  let(:profile) { create(:profile, user: user) }
  
  # 事前に作成が必要な場合のみlet!を使用
  let!(:project) { create(:project, profile: profile) }
  
  it 'finds the project' do
    expect(Project.find(project.id)).to eq(project)
  end
end
```

**非推奨**:
```ruby
RSpec.describe Project do
  # すべてlet!で即時評価（不要なオブジェクトも作成）
  let!(:user) { create(:user) }
  let!(:profile) { create(:profile) }
  let!(:project) { create(:project) }
  
  # このテストではuserとprofileは不要
  it 'counts projects' do
    expect(Project.count).to eq(1)
  end
end
```

**根拠**: letは遅延評価されるため、参照されない場合はオブジェクトが作成されず、テストの実行速度が向上します。

### フックの使用

**推奨**:
```ruby
RSpec.describe Database do
  # 各テストの前に実行
  before do
    Database.start_transaction
  end
  
  # 各テストの後に実行
  after do
    Database.rollback_transaction
  end
  
  # aroundフックでテストをラップ
  around do |example|
    Database.transaction do
      example.run
      raise ActiveRecord::Rollback
    end
  end
end
```

**非推奨**:
```ruby
RSpec.describe Database do
  # before(:all)は危険（トランザクションの外で実行）
  before(:all) do
    @user = User.create(name: "John")
  end
  
  # テスト間でデータが残る
end
```

**根拠**: before(:all)で作成されたデータはトランザクションによるロールバックが適用されず、テスト間でデータが漏洩します。before(:each)またはletを使用してください。

## Expectations（期待値）

### expect構文の使用

**推奨**:
```ruby
expect(actual).to eq(expected)
expect(actual).not_to eq(expected)
```

**非推奨**:
```ruby
# should構文は非推奨
actual.should eq(expected)
actual.should_not eq(expected)
```

**根拠**: RSpec 3以降、expect構文が標準であり、明示的で読みやすいコードになります。

### 組み込みマッチャーの活用

#### 等価性と同一性

```ruby
# オブジェクトの等価性（==）
expect(actual).to eq(expected)

# オブジェクトの同一性（equal?）
expect(actual).to be(expected)

# 範囲チェック
expect(actual).to be_between(min, max).inclusive
expect(actual).to be_within(delta).of(expected)
```

#### 真偽値

```ruby
# 推奨：明示的な真偽値チェック
expect(user.admin?).to be(true)
expect(user.active?).to be(false)

# 避ける：述語マッチャー（曖昧）
expect(user).to be_admin
```

**根拠**: 明示的なマッチャーを使用することで、テストの意図が明確になります。

#### コレクション

```ruby
expect(array).to include(element)
expect(array).to contain_exactly(1, 2, 3)  # 順不同で完全一致
expect(hash).to include(key: value)
```

#### 変更の検証

```ruby
# 値の変化を検証
expect { user.activate! }.to change { user.active? }.from(false).to(true)
expect { cart.add_item(item) }.to change { cart.items.count }.by(1)
expect { action }.to change { object.value }.by_at_least(min)
```

#### 例外とエラー

**推奨**:
```ruby
# 特定のエラークラスを指定
expect { risky_operation }.to raise_error(SpecificError)
expect { risky_operation }.to raise_error(SpecificError, /error message/)
```

**非推奨**:
```ruby
# どんなエラーでもパスする（テストとして不十分）
expect { risky_operation }.to raise_error
```

**根拠**: 特定のエラークラスを指定することで、予期しないエラーを検出できます。

### マッチャーの合成

```ruby
# ネストした構造の検証
expect(hash).to match(
  name: a_string_matching(/John/),
  age: a_value_between(20, 30),
  email: a_string_ending_with("@example.com")
)

# 複数のマッチャーの組み合わせ
expect(response).to start_with("Success").and end_with("OK")
expect(value).to be_positive.or be_zero
```

### 失敗の集約

複数の期待値をまとめて検証し、すべての失敗を報告:

```ruby
aggregate_failures "user attributes" do
  expect(user.name).to eq("John")
  expect(user.email).to eq("john@example.com")
  expect(user.age).to eq(30)
end
```

**根拠**: 最初の失敗でテストが中断されず、すべての失敗を一度に確認できます。

## Mocks（モック）

### 検証ダブルの使用（推奨）

**推奨**:
```ruby
# インスタンスダブル（実際のクラスを検証）
book = instance_double("Book", title: "RSpec Book", pages: 250)

# クラスダブル
book_class = class_double("Book").as_stubbed_const
```

**非推奨**:
```ruby
# 通常のダブル（存在しないメソッドでもエラーにならない）
book = double("Book")
allow(book).to receive(:non_existent_method)
```

**根拠**: 検証ダブルは、実際のクラスに存在しないメソッドをスタブするとエラーになるため、リファクタリング時の安全性が向上します。

### テスト対象のモック（アンチパターン）

**非推奨**:
```ruby
# テスト対象自体をモックしている
describe Article do
  subject(:article) { Article.new }
  
  it 'indicates unknown author' do
    allow(article).to receive(:author).and_return(nil)
    expect(article.description).to include('unknown author')
  end
end
```

**推奨**:
```ruby
# 適切な初期化でテスト
describe Article do
  subject(:article) { Article.new(author: nil) }
  
  it 'indicates unknown author' do
    expect(article.description).to include('unknown author')
  end
end
```

**根拠**: テスト対象をモックすると、実際の振る舞いをテストできません。

### スパイの活用

メソッドが呼ばれた後に検証:

```ruby
logger = spy("logger")

# アクションを実行
user.login(logger)

# 後で検証
expect(logger).to have_received(:log).with("User logged in")
```

### メッセージの制約

```ruby
# 引数の制約
expect(logger).to receive(:log).with(a_string_matching(/error/))
expect(logger).to receive(:log).with(anything)

# 呼び出し回数の制約
expect(logger).to receive(:log).once
expect(logger).to receive(:log).at_least(2).times
```

## Rails統合

### リクエストスペック（推奨）

コントローラースペックの代わりにリクエストスペックを使用:

```ruby
RSpec.describe "Users", type: :request do
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
    end
  end
end
```

### システムスペック

E2Eテストには、システムスペックを使用:

```ruby
RSpec.describe "User registration", type: :system do
  before do
    driven_by(:rack_test)
  end
  
  it "allows a user to sign up" do
    visit new_user_registration_path
    
    fill_in "Email", with: "user@example.com"
    fill_in "Password", with: "password123"
    
    click_button "Sign up"
    
    expect(page).to have_content("Welcome!")
  end
end
```

JavaScript使用時:

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

### トランザクショナルフィクスチャ

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  # 各テスト後に自動ロールバック
  config.use_transactional_fixtures = true
end
```

## パフォーマンス最適化

### FactoryBotの効率的な使用

**推奨**:
```ruby
# build: メモリ内のみ（データベース保存なし）
let(:user) { build(:user) }

# build_stubbed: 最速（関連も作成しない）
let(:profile) { build_stubbed(:profile) }

# create: データベース保存が必要な場合のみ
let!(:company) { create(:company) }
```

**非推奨**:
```ruby
# すべてcreateで保存（遅い）
let(:user) { create(:user) }
let(:profile) { create(:profile) }
let(:company) { create(:company) }
```

**根拠**: buildやbuild_stubbedを使用することで、不要なデータベースアクセスを削減し、テストの実行速度が大幅に向上します。

### データベースアクセスの最小化

**推奨**:
```ruby
# バリデーションのテストではbuildを使用
describe User do
  it 'validates presence of name' do
    user = build(:user, name: nil)
    expect(user).not_to be_valid
  end
end
```

**非推奨**:
```ruby
# 不要なデータベース保存
describe User do
  it 'validates presence of name' do
    user = create(:user, name: nil)  # データベースに保存しようとする
    expect(user).not_to be_valid
  end
end
```

### プロファイリング

最も遅いテストを特定:

```bash
# 最も遅い10個のテストを表示
rspec --profile 10

# test-profを使用
FPROF=1 bundle exec rspec
EVENT_PROF=sql.active_record bundle exec rspec
```

### let_it_beの活用（test-prof）

読み取り専用のフィクスチャを複数のテストで共有:

```ruby
RSpec.describe ProfileBlueprint do
  # すべてのテストで一度だけ作成
  let_it_be(:user) { create(:user) }
  let_it_be(:profile) { create(:profile, user: user) }
  
  it 'serializes id' do
    expect(described_class.render_as_hash(profile)).to include(id: profile.id)
  end
  
  it 'serializes username' do
    # 同じuserとprofileが再利用される
    expect(described_class.render_as_hash(profile)).to include(username: profile.username)
  end
end
```

**注意**: テストがフィクスチャを変更しない場合のみ使用してください。

## 共有コンテキストと共有サンプル

### 共有コンテキスト

重複するセットアップコードを抽出:

```ruby
# spec/support/shared_contexts/authenticated_user.rb
RSpec.shared_context 'authenticated user' do
  let(:user) { create(:user) }
  let(:token) { JWT.encode({ user_id: user.id }, Rails.application.secrets.secret_key_base) }
end

# 使用例
RSpec.describe 'Projects API' do
  include_context 'authenticated user'
  
  # user、tokenが利用可能
end
```

### 共有サンプル

共通の振る舞いをテスト:

```ruby
# spec/support/shared_examples/sortable_resource.rb
RSpec.shared_examples 'a sortable resource' do
  it 'sorts by name' do
    expect(collection.sort_by_name).to be_sorted
  end
end

# 使用例
RSpec.describe Product do
  it_behaves_like 'a sortable resource' do
    let(:collection) { Product.all }
  end
end
```

## よくある落とし穴

### 一つのテストで複数のことをテストする

**非推奨**:
```ruby
it "creates and validates user" do
  user = User.create(name: "John", email: "john@example.com")
  expect(user).to be_valid
  expect(user.name).to eq("John")
  expect(user.email).to eq("john@example.com")
  expect(User.count).to eq(1)
end
```

**推奨**:
```ruby
describe "User creation" do
  let(:user) { User.create(name: "John", email: "john@example.com") }
  
  it "is valid with valid attributes" do
    expect(user).to be_valid
  end
  
  it "saves the name" do
    expect(user.name).to eq("John")
  end
  
  it "increments the user count" do
    expect { user }.to change(User, :count).by(1)
  end
end
```

**根拠**: 一つのテストで一つの概念のみをテストすることで、失敗の原因が明確になります。

### テストの独立性の欠如

**非推奨**:
```ruby
let(:counter) { 0 }

it "first test" do
  counter += 1
  expect(counter).to eq(1)
end

it "second test" do
  # 実行順序によって失敗する可能性
  expect(counter).to eq(0)
end
```

**推奨**:
```ruby
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

### 実装の詳細のテスト

**非推奨**:
```ruby
# 内部実装に依存
it "calls calculate_total" do
  expect(order).to receive(:calculate_total)
  order.process
end
```

**推奨**:
```ruby
# 振る舞いをテスト
it "updates the total amount" do
  order.process
  expect(order.total).to eq(100)
end
```

**根拠**: 実装の詳細に依存すると、リファクタリング時にテストが壊れやすくなります。

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

### セキュリティ関連のメソッドは実際の振る舞いをテスト

```ruby
describe "Authentication" do
  it "encrypts password" do
    user = User.new(password: "secret")
    # 実際の暗号化をテスト
    expect(user.encrypted_password).not_to eq("secret")
    expect(user.encrypted_password).to be_present
  end
end
```

**根拠**: セキュリティ関連の機能をモックすると、実際の脆弱性を見逃す可能性があります。

## 参考資料

- [RSpec 公式ドキュメント](https://rspec.info/)
- [RSpec Style Guide](https://rspec.rubystyle.guide/)
- [Better Specs](https://www.betterspecs.org/)
- [test-prof Documentation](https://test-prof.evilmartians.io/)
- [RSpec Antipatterns](https://alchemists.io/articles/rspec_antipatterns)
- [docs/best_practice/RSpec/official.md](../../docs/best_practice/RSpec/official.md)
- [docs/best_practice/RSpec/community.md](../../docs/best_practice/RSpec/community.md)
