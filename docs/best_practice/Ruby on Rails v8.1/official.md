# Ruby on Rails v8.1 公式ベストプラクティス

## 概要

Ruby on Rails v8.1は、Webアプリケーション開発を効率的に行うためのフルスタックフレームワークです。MVCアーキテクチャに基づき、Convention over Configuration（設定より規約）とDRY（Don't Repeat Yourself）の原則を採用しています。

**主要機能:**
- Active Record: ORM（オブジェクト/リレーショナルマッピング）
- Action View: ビューテンプレートシステム
- Action Controller: リクエスト処理とルーティング
- Active Support: コア拡張とユーティリティ
- Action Mailer: メール送受信
- Active Storage: ファイルアップロード
- Action Text: リッチテキストエディタ
- Hotwire: サーバーサイドHTML中心のフロントエンド

## 推奨設定

### 基本設定

#### アプリケーション設定ファイルの配置

初期化コードは以下の場所に配置します：

```ruby
# config/application.rb
# 環境ごとの設定ファイル（config/environments/*.rb）
# イニシャライザファイル（config/initializers/*.rb）
# アフターイニシャライザファイル
```

#### Rails実行前のコード実行

Rails自体が読み込まれる前にコードを実行する必要がある場合は、`config/application.rb`の`require 'rails/all'`の行の**上**に配置します。

#### バージョン固有のデフォルト値

Rails 8.1のデフォルト設定：

```ruby
# config/application.rb
config.action_controller.action_on_path_relative_redirect = :raise
config.action_controller.escape_json_responses = false
config.action_view.remove_hidden_field_autocomplete = true
config.action_view.render_tracker = :ruby
config.active_record.raise_on_missing_required_finder_order_columns = true
config.active_support.escape_js_separators_in_json = false
config.yjit = !Rails.env.local?
```

### 環境別設定

#### Development環境

```ruby
# config/environments/development.rb
config.cache_classes = false
config.eager_load = false
config.consider_all_requests_local = true
config.active_support.deprecation = :log
config.active_record.migration_error = :page_load
config.assets.debug = true
```

#### Production環境

```ruby
# config/environments/production.rb
config.cache_classes = true
config.eager_load = true
config.consider_all_requests_local = false
config.log_level = :info
config.force_ssl = true
```

#### Test環境

```ruby
# config/environments/test.rb
config.cache_classes = true
config.eager_load = false
config.active_support.deprecation = :stderr
```

### データベース設定

#### 基本設定

```yaml
# config/database.yml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

test:
  <<: *default
  database: storage/test.sqlite3

production:
  <<: *default
  database: storage/production.sqlite3
```

## セキュリティ

### 認証機能

Rails 8.0から認証機能ジェネレータがデフォルトで付属：

```bash
$ bin/rails generate authentication
$ bin/rails db:migrate
```

生成されるコンポーネント：
- `User`モデル（パスワードのハッシュ化にBCryptを使用）
- `Session`モデル
- 認証用コントローラとビュー
- `Authentication` concern

#### パスワード管理のベストプラクティス

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  
  validates :email_address, presence: true, uniqueness: true
  validates :password, length: { minimum: 12 }
end
```

### CSRF（クロスサイトリクエストフォージェリ）対策

#### 必須セキュリティトークン

Railsはデフォルトで全フォームにCSRFトークンを含めます：

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```

#### GETとPOSTの適切な使い分け

```ruby
# ❌ 避けるべき
get '/users/:id/delete', to: 'users#destroy'

# ✅ 推奨
delete '/users/:id', to: 'users#destroy'
```

**重要:** GET リクエストはデータの変更を伴うべきではありません。

### SQLインジェクション対策

#### 安全なクエリの書き方

```ruby
# ❌ 危険（SQLインジェクションに脆弱）
User.where("name = '#{params[:name]}'")

# ✅ 推奨（プレースホルダーを使用）
User.where("name = ?", params[:name])

# ✅ 推奨（ハッシュ形式）
User.where(name: params[:name])

# ✅ 推奨（名前付きプレースホルダー）
User.where("name = :name", name: params[:name])
```

### XSS（クロスサイトスクリプティング）対策

#### 自動エスケープ

Railsはデフォルトで全出力をエスケープします：

```erb
<!-- 自動的にエスケープされる -->
<%= @user.name %>

<!-- HTMLとして出力（注意して使用） -->
<%== @user.bio %>
<%# または %>
<%= raw @user.bio %>
<%= @user.bio.html_safe %>
```

#### サニタイズヘルパー

```ruby
# 許可リスト方式でHTMLをサニタイズ
sanitize(@user.bio, tags: %w(strong em a), attributes: %w(href))

# すべてのHTMLタグを除去
strip_tags(@user.bio)
```

### HTTPセキュリティヘッダー

#### デフォルトのセキュリティヘッダー

Rails 7.1以降のデフォルト：

```ruby
config.action_dispatch.default_headers = {
  'X-Frame-Options' => 'SAMEORIGIN',
  'X-XSS-Protection' => '0',
  'X-Content-Type-Options' => 'nosniff',
  'X-Permitted-Cross-Domain-Policies' => 'none',
  'Referrer-Policy' => 'strict-origin-when-cross-origin'
}
```

#### Content Security Policy（CSP）

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
end
```

#### Strict-Transport-Security（HSTS）

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = { hsts: { subdomains: true, preload: true } }
```

### パラメータフィルタリング

機密情報をログから除外：

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password, :password_confirmation,
  :passw, :email, :secret, :token,
  :_key, :crypt, :salt, :certificate,
  :otp, :ssn, :cvv, :cvc
]
```

### Credentials管理

```bash
# credentialsの編集
$ bin/rails credentials:edit

# 環境別credentials
$ bin/rails credentials:edit --environment production
```

```yaml
# config/credentials.yml.enc（復号化後）
secret_key_base: xxxxxx
aws:
  access_key_id: xxxxxx
  secret_access_key: xxxxxx
```

```ruby
# 使用方法
Rails.application.credentials.aws[:access_key_id]
```

**重要:** `config/master.key`は絶対にバージョン管理に含めないこと。

## パフォーマンス

### データベース最適化

#### N+1クエリ問題の解決

```ruby
# ❌ N+1クエリ問題
@posts = Post.all
@posts.each do |post|
  puts post.author.name  # 各postごとにクエリ実行
end

# ✅ Eager Loading
@posts = Post.includes(:author)
@posts.each do |post|
  puts post.author.name  # 1つのクエリで解決
end
```

#### クエリの最適化

```ruby
# 必要なカラムのみを取得
User.select(:id, :name, :email)

# バッチ処理
User.find_each(batch_size: 100) do |user|
  # 処理
end

# カウンタキャッシュ
class Article < ApplicationRecord
  belongs_to :author, counter_cache: true
end
```

### キャッシング

#### フラグメントキャッシュ

```erb
<% cache @product do %>
  <h1><%= @product.name %></h1>
  <p><%= @product.description %></p>
<% end %>
```

#### ロシアンドール（ネスト）キャッシュ

```erb
<% cache @product do %>
  <%= render @product %>
  <% cache @product.games do %>
    <%= render @product.games %>
  <% end %>
<% end %>
```

#### キャッシュストアの設定

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store

# その他のオプション
# config.cache_store = :mem_cache_store
# config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
```

### アセット管理

#### Propshaftの使用

Rails 8.0以降のデフォルト：

```ruby
# config/application.rb
require "propshaft/railtie"
```

#### アセットのプリコンパイル

```bash
$ bin/rails assets:precompile
```

### Background Jobs

#### Solid Queue（Rails 8.0デフォルト）

```ruby
# app/jobs/user_notification_job.rb
class UserNotificationJob < ApplicationJob
  queue_as :default

  def perform(user)
    UserMailer.welcome(user).deliver_now
  end
end

# 使用方法
UserNotificationJob.perform_later(@user)
```

## Active Recordベストプラクティス

### 命名規約

#### モデルとテーブル

| モデルクラス（単数形） | データベーステーブル（複数形） |
| ---------------------- | ------------------------------ |
| `Article`              | `articles`                     |
| `LineItem`             | `line_items`                   |
| `Product`              | `products`                     |
| `Person`               | `people`                       |

#### スキーマ規約

**主キー:** `id`（自動生成）

**外部キー:** `単数形テーブル名_id`（例: `user_id`、`article_id`）

**予約カラム:**
- `created_at`: レコード作成時刻
- `updated_at`: レコード更新時刻
- `lock_version`: 楽観的ロック
- `type`: STI（Single Table Inheritance）
- `関連付け名_type`: ポリモーフィック関連付け
- `テーブル名_count`: カウンタキャッシュ

### バリデーション

#### 基本的なバリデーション

```ruby
class User < ApplicationRecord
  # 存在チェック
  validates :name, presence: true
  
  # 一意性チェック
  validates :email, uniqueness: { case_sensitive: false }
  
  # 長さチェック
  validates :password, length: { minimum: 8, maximum: 128 }
  
  # フォーマットチェック
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  
  # 数値チェック
  validates :age, numericality: { 
    only_integer: true,
    greater_than_or_equal_to: 18
  }
  
  # 包含チェック
  validates :status, inclusion: { in: %w[active inactive] }
  
  # 除外チェック
  validates :subdomain, exclusion: { in: %w[www admin] }
end
```

#### カスタムバリデーション

```ruby
class User < ApplicationRecord
  validate :email_domain_allowed

  private

  def email_domain_allowed
    unless email.end_with?('@example.com')
      errors.add(:email, 'must be from example.com domain')
    end
  end
end
```

#### 条件付きバリデーション

```ruby
class Order < ApplicationRecord
  validates :card_number, presence: true, if: :paid_with_card?
  validates :address, presence: true, unless: :digital_product?

  private

  def paid_with_card?
    payment_type == 'card'
  end

  def digital_product?
    product.digital?
  end
end
```

### コールバック

```ruby
class User < ApplicationRecord
  before_validation :normalize_email
  before_save :encrypt_password
  after_create :send_welcome_email
  after_destroy :cleanup_associated_data

  private

  def normalize_email
    self.email = email.downcase.strip if email.present?
  end

  def encrypt_password
    # パスワードの暗号化処理
  end

  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end

  def cleanup_associated_data
    # 関連データのクリーンアップ
  end
end
```

### 関連付け

```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_one :profile, dependent: :destroy
  
  # Through関連付け
  has_many :post_comments, through: :posts, source: :comments
end

class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :tags, through: :taggings
end

class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user
end
```

#### ポリモーフィック関連付け

```ruby
class Picture < ApplicationRecord
  belongs_to :imageable, polymorphic: true
end

class Employee < ApplicationRecord
  has_many :pictures, as: :imageable
end

class Product < ApplicationRecord
  has_many :pictures, as: :imageable
end
```

## ルーティングベストプラクティス

### リソースフルルーティング

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # 基本的なリソース
  resources :articles
  
  # 制限付きリソース
  resources :photos, only: [:index, :show]
  resources :users, except: [:destroy]
  
  # ネストしたリソース（1階層まで推奨）
  resources :articles do
    resources :comments, shallow: true
  end
  
  # 単数形リソース
  resource :profile, only: [:show, :edit, :update]
  
  # カスタムアクション
  resources :products do
    member do
      post :purchase
    end
    collection do
      get :search
    end
  end
end
```

### 名前空間とスコープ

```ruby
# 管理画面
namespace :admin do
  resources :articles
end

# URL変更のみ
scope '/admin' do
  resources :articles, as: :admin_articles
end

# モジュール指定
scope module: 'admin' do
  resources :articles
end
```

### ルートパス

```ruby
root 'pages#home'
```

## よくある落とし穴

### 1. マスアサインメントの脆弱性

```ruby
# ❌ 危険
def create
  @user = User.create(params[:user])
end

# ✅ 推奨（Strong Parameters使用）
def create
  @user = User.create(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email)
end
```

### 2. N+1クエリ問題

```ruby
# ❌ 非効率
@users.each { |user| user.posts.count }

# ✅ 推奨
@users = User.includes(:posts)
@users.each { |user| user.posts.size }
```

### 3. 正規表現の誤用

```ruby
# ❌ 危険（改行文字で回避可能）
validates :url, format: { with: /^https?:\/\/.+$/ }

# ✅ 推奨（\A と \z を使用）
validates :url, format: { with: /\Ahttps?:\/\/.+\z/ }
```

### 4. デフォルトスコープの濫用

```ruby
# ❌ 避けるべき（予期しない動作の原因）
class Post < ApplicationRecord
  default_scope { where(published: true) }
end

# ✅ 推奨（明示的なスコープ）
class Post < ApplicationRecord
  scope :published, -> { where(published: true) }
end
```

### 5. トランザクション処理の不備

```ruby
# ❌ 不適切
def transfer_funds
  from_account.withdraw(amount)
  to_account.deposit(amount)
end

# ✅ 推奨
def transfer_funds
  ActiveRecord::Base.transaction do
    from_account.withdraw!(amount)
    to_account.deposit!(amount)
  end
end
```

### 6. コールバックの複雑化

```ruby
# ❌ 避けるべき
class User < ApplicationRecord
  after_create :send_welcome_email
  after_create :create_profile
  after_create :log_user_creation
  after_create :notify_admins
  # ... 多数のコールバック
end

# ✅ 推奨（サービスオブジェクトに切り出し）
class UserRegistrationService
  def initialize(user)
    @user = user
  end

  def call
    ActiveRecord::Base.transaction do
      @user.save!
      send_welcome_email
      create_profile
      log_user_creation
      notify_admins
    end
  end

  private
  # ...
end
```

### 7. 過度なeager loading

```ruby
# ❌ 不必要なロード
@users = User.includes(:posts, :comments, :profile, :settings)
# 実際にはpostsしか使わない

# ✅ 必要なもののみロード
@users = User.includes(:posts)
```

### 8. 環境別設定の混在

```ruby
# ❌ 避けるべき
if Rails.env.production?
  # production固有の処理
end

# ✅ 推奨（環境別設定ファイルで管理）
# config/environments/production.rb
config.some_setting = true
```

### 9. 秘密情報のハードコーディング

```ruby
# ❌ 危険
class ApiClient
  API_KEY = "sk_live_xxxxxxxxxxxxxx"
end

# ✅ 推奨
class ApiClient
  def self.api_key
    Rails.application.credentials.api_key
  end
end
```

### 10. 巨大なコントローラ

```ruby
# ❌ Fat Controller
class UsersController < ApplicationController
  def create
    # 100行以上のビジネスロジック
  end
end

# ✅ Skinny Controller（サービスオブジェクトパターン）
class UsersController < ApplicationController
  def create
    result = UserRegistrationService.new(user_params).call
    
    if result.success?
      redirect_to result.user
    else
      render :new
    end
  end
end
```

## テスト

### 基本的なテスト構成

```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "should not save user without email" do
    user = User.new
    assert_not user.save
  end

  test "should save valid user" do
    user = User.new(email: "test@example.com", password: "password")
    assert user.save
  end
end
```

### フィクスチャの使用

```yaml
# test/fixtures/users.yml
john:
  email: john@example.com
  name: John Doe

jane:
  email: jane@example.com
  name: Jane Doe
```

```ruby
test "should find user" do
  john = users(:john)
  assert_equal "John Doe", john.name
end
```

## 追加の推奨事項

### Gemfileの整理

```ruby
# Gemfile
source 'https://rubygems.org'

ruby '3.2.0'

gem 'rails', '~> 8.1.0'
gem 'sqlite3', '>= 1.4'
gem 'puma', '>= 5.0'

group :development, :test do
  gem 'debug', platforms: %i[ mri windows ]
end

group :development do
  gem 'web-console'
end

group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
end
```

### RuboCopの使用

```bash
# コードスタイルチェック
$ bin/rubocop

# 自動修正
$ bin/rubocop -a
```

### Brakemanの使用

```bash
# セキュリティスキャン
$ bin/brakeman
```

### GitHub Actionsでの自動化

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails test
      - run: bin/rubocop
      - run: bin/brakeman
```

## 参考リンク

- [Rails Guides 日本語版](https://railsguides.jp/)
- [Rails API Documentation](https://api.rubyonrails.org/)
- [Rails Security Guide](https://railsguides.jp/security.html)
- [Active Record Query Interface](https://railsguides.jp/active_record_querying.html)
- [Routing Guide](https://railsguides.jp/routing.html)
