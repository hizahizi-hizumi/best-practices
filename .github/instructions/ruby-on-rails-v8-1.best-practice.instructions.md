---
description: Ruby on Rails v8.1 のベストプラクティス
applyTo: "**/*.rb, **/Gemfile*, **/Rakefile, **/config.ru, **/*.rake"
---

# Ruby on Rails v8.1 ベストプラクティス

## 基本原則

- Convention over Configuration（設定より規約）とDRY（Don't Repeat Yourself）の原則に従う
- MVCアーキテクチャを適切に維持する
- 公式のRails規約に準拠したコードを書く
- セキュリティを最優先事項として扱う
- パフォーマンスとスケーラビリティを考慮した設計を行う

## プロジェクト構成

### 新規プロジェクトの作成

- `rails new myapp` で新規プロジェクトを作成する
- Rails 8.1ではデフォルトで以下が含まれる：
  - Kamal（デプロイメント）
  - Solid Queue（バックグラウンドジョブ）
  - Solid Cache（キャッシング）
  - Solid Cable（WebSocket）
  - Propshaft（アセットパイプライン）
  - Hotwire（フロントエンド）

### ディレクトリ構造

- 初期化コードは `config/application.rb`、`config/environments/*.rb`、`config/initializers/*.rb` に配置する
- Rails読み込み前にコード実行が必要な場合は、`config/application.rb` の `require 'rails/all'` の**上**に配置する
- 環境別の設定は `config/environments/` 配下で管理する

### Gemfile の整理

- 用途別にグループ化する（`:development`, `:test`, `:production`）
- バージョンを適切に指定する（`~>` を使用）
- 不要なGemは含めない

```ruby
# Gemfile の推奨構成
source 'https://rubygems.org'

ruby '3.3.0'

gem 'rails', '~> 8.1.0'
gem 'sqlite3', '>= 1.4'
gem 'puma', '>= 5.0'

group :development, :test do
  gem 'debug', platforms: %i[ mri windows ]
  gem 'rubocop-rails', require: false
  gem 'rubocop-rspec', require: false
  gem 'rubocop-performance', require: false
end

group :development do
  gem 'web-console'
  gem 'rack-mini-profiler'
  gem 'bullet'
end

group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
end
```

## アプリケーション設定

### バージョン固有のデフォルト値

- Rails 8.1のデフォルト設定を使用する：

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

- 接続プールサイズを環境変数で管理する
- タイムアウト値を適切に設定する
- 本番環境では適切なデータベースアダプタを使用する（PostgreSQL、MySQL等）

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

- Rails 8.0以降の認証ジェネレーターを使用する：

```bash
bin/rails generate authentication
bin/rails db:migrate
```

- パスワードは `has_secure_password` を使用してハッシュ化する
- パスワードの最小文字数は12文字以上にする
- セッション管理を適切に実装する

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  validates :email_address, presence: true, uniqueness: true
  validates :password, length: { minimum: 12 }
end
```

### セッション設定

- 本番環境では secure フラグを有効にする
- httponly を有効にしてXSS攻撃を防ぐ
- same_site を適切に設定する

```ruby
# config/application.rb または config/initializers/session_store.rb
config.session_store :cookie_store,
  key: '_myapp_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 14.days
```

### CSRF対策

- `protect_from_forgery with: :exception` を ApplicationController に設定する
- GET リクエストでデータを変更しない
- 適切なHTTPメソッドを使用する（POST、PUT、PATCH、DELETE）

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```

### SQLインジェクション対策

- **絶対に**文字列補間を使用しない
- プレースホルダーまたはハッシュ形式を使用する

```ruby
# ❌ 危険（絶対に使用禁止）
User.where("name = '#{params[:name]}'")

# ✅ 安全（プレースホルダー）
User.where("name = ?", params[:name])

# ✅ 安全（ハッシュ形式）
User.where(name: params[:name])

# ✅ 安全（名前付きプレースホルダー）
User.where("name = :name", name: params[:name])
```

### XSS対策

- Railsのデフォルト自動エスケープを活用する
- `raw` や `html_safe` は信頼できるデータにのみ使用する
- ユーザー入力を表示する場合は `sanitize` ヘルパーを使用する

```erb
<!-- 自動的にエスケープされる -->
<%= @user.name %>

<!-- 許可リスト方式でサニタイズ -->
<%= sanitize @user.bio, tags: %w(strong em a), attributes: %w(href) %>

<!-- すべてのHTMLタグを除去 -->
<%= strip_tags(@user.bio) %>
```

### HTTPセキュリティヘッダー

- デフォルトのセキュリティヘッダーを維持する
- Content Security Policy（CSP）を設定する
- Strict-Transport-Security（HSTS）を有効にする

```ruby
# config/application.rb
config.action_dispatch.default_headers = {
  'X-Frame-Options' => 'SAMEORIGIN',
  'X-XSS-Protection' => '0',
  'X-Content-Type-Options' => 'nosniff',
  'X-Permitted-Cross-Domain-Policies' => 'none',
  'Referrer-Policy' => 'strict-origin-when-cross-origin'
}

# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
end

# config/environments/production.rb
config.force_ssl = true
config.ssl_options = { hsts: { subdomains: true, preload: true } }
```

### Strong Parameters（マスアサインメント対策）

- すべてのコントローラでStrong Parametersを使用する
- 許可する属性を明示的に定義する

```ruby
# ❌ 危険
def create
  @user = User.create(params[:user])
end

# ✅ 推奨
def create
  @user = User.create(user_params)
end

private

def user_params
  params.require(:user).permit(:name, :email)
end
```

### Credentials管理

- 秘密情報はCredentialsで管理する
- `config/master.key` は**絶対に**バージョン管理に含めない
- 環境別にCredentialsを分ける

```bash
# Credentialsの編集
bin/rails credentials:edit

# 環境別Credentials
bin/rails credentials:edit --environment production
```

```ruby
# Credentialsへのアクセス
Rails.application.credentials.api_key
Rails.application.credentials.dig(:aws, :access_key_id)
```

### パラメータフィルタリング

- 機密情報をログから除外する

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password, :password_confirmation,
  :passw, :email, :secret, :token,
  :_key, :crypt, :salt, :certificate,
  :otp, :ssn, :cvv, :cvc
]
```

### セキュリティ監査

- Brakemanで定期的にセキュリティスキャンを実行する
- Bundle Auditで依存関係の脆弱性をチェックする
- CI/CDパイプラインに組み込む

```bash
# セキュリティスキャン
bin/brakeman

# 依存関係の脆弱性チェック
bundle audit check --update
```

## Active Recordベストプラクティス

### 命名規約

- モデルクラス名：単数形、キャメルケース（例：`User`, `LineItem`）
- テーブル名：複数形、スネークケース（例：`users`, `line_items`）
- 主キー：`id`（自動生成）
- 外部キー：`単数形テーブル名_id`（例：`user_id`, `article_id`）

#### 予約カラム

- `created_at`: レコード作成時刻（自動設定）
- `updated_at`: レコード更新時刻（自動設定）
- `lock_version`: 楽観的ロック
- `type`: STI（Single Table Inheritance）
- `関連付け名_type`: ポリモーフィック関連付け
- `テーブル名_count`: カウンタキャッシュ

### バリデーション

- モデルレベルでバリデーションを実装する
- 適切なバリデーションヘルパーを使用する
- カスタムバリデーションは明確な命名にする

```ruby
class User < ApplicationRecord
  # 基本的なバリデーション
  validates :name, presence: true
  validates :email, uniqueness: { case_sensitive: false }
  validates :password, length: { minimum: 8, maximum: 128 }
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age, numericality: { only_integer: true, greater_than_or_equal_to: 18 }
  validates :status, inclusion: { in: %w[active inactive] }
  validates :subdomain, exclusion: { in: %w[www admin] }

  # カスタムバリデーション
  validate :email_domain_allowed

  private

  def email_domain_allowed
    unless email.end_with?('@example.com')
      errors.add(:email, 'must be from example.com domain')
    end
  end
end
```

### 条件付きバリデーション

- `if` / `unless` オプションを使用して条件付きバリデーションを実装する

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

- コールバックは最小限に抑える
- 複雑なビジネスロジックはサービスオブジェクトに切り出す
- コールバックの実行順序に注意する

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
end
```

#### コールバックのアンチパターン

```ruby
# ❌ 避けるべき（複雑すぎるコールバック）
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
  # 各メソッドの実装
end
```

### 関連付け

- 適切な関連付けを定義する
- `dependent` オプションでカスケード削除を設定する
- Through関連付けを活用する

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

### ポリモーフィック関連付け

- 複数のモデルに属する場合はポリモーフィック関連付けを使用する

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

### スコープ

- 再利用可能なクエリはスコープとして定義する
- デフォルトスコープは避ける（予期しない動作の原因）

```ruby
# ✅ 推奨（明示的なスコープ）
class Post < ApplicationRecord
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_author, ->(author) { where(author: author) }
end

# ❌ 避けるべき（デフォルトスコープ）
class Post < ApplicationRecord
  default_scope { where(published: true) }  # 予期しない動作の原因
end
```

## パフォーマンス最適化

### N+1クエリ問題の解決

- `includes` を使用してEager Loadingを行う
- 必要な関連付けのみロードする
- Bulletを使用してN+1クエリを検出する

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

### クエリの最適化

- 必要なカラムのみを取得する
- バッチ処理を使用する
- カウンタキャッシュを活用する

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

### インデックスの追加

- 頻繁に検索される外部キーにインデックスを追加する
- 複合インデックスを適切に使用する
- 部分インデックスを活用する

```ruby
# db/migrate/xxxxx_add_performance_indexes.rb
class AddPerformanceIndexes < ActiveRecord::Migration[8.1]
  def change
    # 外部キー
    add_index :posts, :user_id
    add_index :comments, [:post_id, :created_at]

    # 複合インデックス
    add_index :posts, [:published, :created_at]

    # 部分インデックス
    add_index :posts, :created_at,
      where: "published = true",
      name: "index_posts_published_on_created_at"

    # ユニークインデックス
    add_index :users, :email, unique: true
  end
end
```

### キャッシング

#### フラグメントキャッシュ

- ビューの一部をキャッシュする
- キャッシュキーにモデルを使用する

```erb
<% cache @product do %>
  <h1><%= @product.name %></h1>
  <p><%= @product.description %></p>
<% end %>
```

#### ロシアンドール（ネスト）キャッシュ

- ネストしたキャッシュで効率化する

```erb
<% cache @product do %>
  <%= render @product %>
  <% cache @product.games do %>
    <%= render @product.games %>
  <% end %>
<% end %>
```

#### 低レベルキャッシング

- 計算結果やAPI応答をキャッシュする

```ruby
class PostsController < ApplicationController
  def statistics
    @stats = Rails.cache.fetch("post_statistics", expires_in: 1.hour) do
      {
        total: Post.count,
        published: Post.published.count,
        popular: Post.order(views: :desc).limit(10).pluck(:id, :title)
      }
    end
  end
end
```

#### HTTPキャッシング

- ETags や Last-Modified を使用する

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])

    # ETags
    fresh_when(@post)

    # または Last-Modified
    # fresh_when(last_modified: @post.updated_at, etag: @post)
  end
end
```

### Background Jobs

#### Solid Queue（Rails 8.0デフォルト）

- 長時間実行される処理はバックグラウンドジョブで実行する
- 適切なキュー名を設定する
- エラーハンドリングを実装する

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

#### Active Job Continuations（Rails 8.1新機能）

- 長時間実行ジョブを中断可能なステップに分割する
- デプロイ時のジョブロストを防ぐ

```ruby
class ProcessImportJob < ApplicationJob
  include ActiveJob::Continuable

  def perform(import_id)
    @import = Import.find(import_id)

    # 初期化ステップ
    step :initialize do
      @import.initialize!
    end

    # カーソル付きステップ（中断・再開可能）
    step :process do |step|
      @import.records.find_each(start: step.cursor) do |record|
        record.process!
        step.advance! from: record.id
      end
    end

    # 完了ステップ
    step :finalize
  end

  private

  def finalize
    @import.mark_as_completed!
  end
end
```

### アセット管理

#### Propshaft（Rails 8.0デフォルト）

- Propshaftをデフォルトで使用する
- 本番環境ではアセットをプリコンパイルする

```bash
bin/rails assets:precompile
```

#### 画像の最適化

- 遅延読み込みを使用する
- 適切なサイズにリサイズする
- WebP形式を活用する

```erb
<%= image_tag @post.image.variant(resize_to_limit: [800, 600]),
    loading: :lazy,
    srcset: {
      @post.image.variant(resize_to_limit: [400, 300]) => "400w",
      @post.image.variant(resize_to_limit: [800, 600]) => "800w"
    } %>
```

## ルーティング

### リソースフルルーティング

- RESTfulなルーティングを使用する
- 不要なアクションは制限する
- ネストは1階層までにする

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # 基本的なリソース
  resources :articles

  # 制限付きリソース
  resources :photos, only: [:index, :show]
  resources :users, except: [:destroy]

  # ネストしたリソース（shallow推奨）
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

- 管理画面などは名前空間で分離する

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

- ルートパスを必ず定義する

```ruby
root 'pages#home'
```

### HTTPメソッドの適切な使用

- GET：データの取得のみ（副作用なし）
- POST：新規作成
- PATCH/PUT：更新
- DELETE：削除

```ruby
# ❌ 避けるべき
get '/users/:id/delete', to: 'users#destroy'

# ✅ 推奨
delete '/users/:id', to: 'users#destroy'
```

## コーディング規約

### コントローラ

- Skinny Controller、Fat Modelの原則に従う
- 複雑なビジネスロジックはサービスオブジェクトに切り出す
- 1アクション1責務を心がける

```ruby
# ❌ Fat Controller
class UsersController < ApplicationController
  def create
    # 100行以上のビジネスロジック
  end
end

# ✅ Skinny Controller
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

### モデル

- ビジネスロジックはモデルに配置する
- 1つのメソッドは1つの責務を持つ
- 複雑なロジックはサービスオブジェクトやConcernに切り出す

### 正規表現の使用

- `^` と `$` ではなく `\A` と `\z` を使用する（改行文字での回避を防ぐ）

```ruby
# ❌ 危険（改行文字で回避可能）
validates :url, format: { with: /^https?:\/\/.+$/ }

# ✅ 推奨
validates :url, format: { with: /\Ahttps?:\/\/.+\z/ }
```

### トランザクション処理

- 複数の更新操作は必ずトランザクションで囲む
- 例外を発生させるメソッド（`!`付き）を使用する

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

### 環境依存の処理

- 環境別設定ファイルで管理する
- コード内で `if Rails.env.production?` は避ける

```ruby
# ❌ 避けるべき
if Rails.env.production?
  # production固有の処理
end

# ✅ 推奨（環境別設定ファイルで管理）
# config/environments/production.rb
config.some_setting = true
```

### コードスタイル

- RuboCopを使用してコードスタイルを統一する
- 自動修正機能を活用する

```bash
# コードスタイルチェック
bin/rubocop

# 自動修正
bin/rubocop -A
```

## Hotwire/Turbo

### Turbo Framesの使用

- 部分更新にTurbo Framesを使用する
- 遅延読み込みを活用する

```erb
<!-- app/views/conversations/show.html.erb -->
<%= turbo_frame_tag "conversation-messages",
    src: conversation_messages_path(@conversation),
    loading: :lazy do %>
  <p>メッセージを読み込み中...</p>
<% end %>
```

### Turbo Streamsの使用

- リアルタイム更新にTurbo Streamsを使用する
- 複数の操作を組み合わせる

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  def create
    @message = @conversation.messages.create!(message_params)

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.append("messages",
            partial: "messages/message",
            locals: { message: @message }
          ),
          turbo_stream.replace("message-form-wrapper",
            partial: "messages/form",
            locals: { conversation: @conversation }
          )
        ]
      end
      format.html { redirect_to @conversation }
    end
  end
end
```

### リアルタイム更新（Turbo Streams + Solid Cable）

- モデルのコールバックでブロードキャストする

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :conversation

  after_create_commit do
    broadcast_append_to conversation,
      target: "messages",
      partial: "messages/message",
      locals: { message: self }
  end

  after_update_commit do
    broadcast_replace_to conversation,
      target: self,
      partial: "messages/message",
      locals: { message: self }
  end
end
```

### Turbo Morphing

- ページ全体の更新でもスムーズなUXを実現する

```erb
<!-- app/views/layouts/application.html.erb -->
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
</head>
```

### Turbo Frameからのリダイレクト

- フレームから抜けるには `turbo_frame: "_top"` を使用する

```ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.create(article_params)
    redirect_to @article, turbo_frame: "_top"
  end
end
```

## Structured Event Reporting（Rails 8.1新機能）

### イベント通知

- 構造化イベントでアプリケーションの動作を追跡する

```ruby
class UsersController < ApplicationController
  def create
    @user = User.create!(user_params)

    Rails.event.notify("user.signup",
      user_id: @user.id,
      email: @user.email,
      plan: @user.plan
    )

    redirect_to dashboard_path
  end
end
```

### タグとコンテキスト

- イベントにタグとコンテキストを追加する

```ruby
class GraphqlController < ApplicationController
  around_action :add_event_context

  def execute
    Rails.event.tagged("graphql", "api") do
      result = MySchema.execute(params[:query])

      Rails.event.notify("graphql.query",
        query_name: extract_query_name(params[:query]),
        duration_ms: result.duration
      )

      render json: result
    end
  end

  private

  def add_event_context
    Rails.event.set_context(
      request_id: request.request_id,
      user_id: current_user&.id,
      ip_address: request.remote_ip
    )
    yield
  ensure
    Rails.event.clear_context!
  end
end
```

### カスタムサブスクライバー

- 外部サービスへのイベント送信を実装する

```ruby
# config/initializers/event_subscriber.rb
class JsonLogSubscriber
  def emit(event)
    log_data = {
      timestamp: Time.current.iso8601,
      event: event[:name],
      payload: event[:payload],
      tags: event[:tags],
      context: event[:context]
    }

    Rails.logger.info(JSON.generate(log_data))
  end
end

Rails.event.subscribe(JsonLogSubscriber.new)
```

## テスト

### 基本的なテスト構成

- すべての重要な機能にテストを書く
- モデル、コントローラ、システムテストを適切に使い分ける

```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "should not save user without email" do
    user = User.new
    assert_not user.save
  end

  test "should save valid user" do
    user = User.new(email: "test@example.com", password: "secure_password123")
    assert user.save
  end
end
```

### フィクスチャの使用

- テストデータはフィクスチャで管理する

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

### Local CI（Rails 8.1新機能）

- ローカル環境でCI全体を実行する

```ruby
# config/ci.rb
ci do
  # テストスイート
  test "unit", "bin/rails test"
  test "system", "bin/rails test:system"

  # セキュリティチェック
  security "brakeman", "bin/brakeman --quiet --no-summary"
  security "bundle audit", "bundle exec bundle-audit check --update"

  # コード品質
  lint "rubocop", "bin/rubocop"
end
```

```bash
# ローカルでCI実行
bin/ci

# 並列実行
bin/ci --parallel
```

## デプロイメント

### Kamalの使用

- Kamalでコンテナベースのデプロイメントを実現する

```bash
# Kamalのセットアップ
bundle add kamal
kamal init
```

```yaml
# config/deploy.yml
service: myapp
image: myapp

servers:
  web:
    hosts:
      - 192.168.1.1
    labels:
      traefik.http.routers.myapp.rule: Host(`myapp.com`)

registry:
  server: ghcr.io
  username: myusername
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
```

```bash
# デプロイ実行
kamal deploy
```

### CI/CD統合

- GitHub Actionsで自動テストとデプロイを実行する

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.3.0
          bundler-cache: true

      - name: Setup database
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
        run: bin/rails db:create db:schema:load

      - name: Run tests
        run: bin/ci
```

### 環境変数の管理

- Credentialsで環境変数を管理する
- Kamalのシークレット機能を活用する

```bash
# Rails credentialsに保存
rails credentials:edit --environment production

# Kamalでシークレットを取得
kamal secrets fetch
```

## よくある落とし穴

### 1. マスアサインメントの脆弱性

- Strong Parametersを必ず使用する

### 2. N+1クエリ問題

- `includes` を使用してEager Loadingを行う
- Bulletで検出する

### 3. 正規表現の誤用

- `\A` と `\z` を使用する（`^` と `$` は避ける）

### 4. デフォルトスコープの濫用

- 明示的なスコープを使用する

### 5. トランザクション処理の不備

- 複数の更新操作はトランザクションで囲む

### 6. コールバックの複雑化

- サービスオブジェクトに切り出す

### 7. 過度なEager Loading

- 必要な関連付けのみロードする

### 8. 環境別設定の混在

- 環境別設定ファイルで管理する

### 9. 秘密情報のハードコーディング

- Credentialsで管理する

### 10. 巨大なコントローラ

- サービスオブジェクトパターンを使用する

## チーム開発

### コード規約の統一

- RuboCopで自動チェックする
- Git Hooksでコミット前にチェックする

```yaml
# .rubocop.yml
require:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance

AllCops:
  NewCops: enable
  TargetRubyVersion: 3.3
  Exclude:
    - 'db/schema.rb'
    - 'db/migrate/*'
    - 'bin/*'

Layout/LineLength:
  Max: 120

Metrics/BlockLength:
  Exclude:
    - 'spec/**/*'
    - 'config/routes.rb'
```

### Pull Requestのガイドライン

- PRテンプレートを作成する
- チェックリストを活用する
- コードレビューを必須にする

### ブランチ戦略

- 機能ブランチ戦略を使用する
- ブランチ命名規則を統一する
  - `feature/機能名` - 新機能
  - `fix/バグ名` - バグ修正
  - `refactor/対象` - リファクタリング

### ドキュメンテーション

- READMEを最新に保つ
- セットアップ手順を明確にする
- トラブルシューティングを記載する

### セットアップスクリプト

- `bin/setup` スクリプトを用意する
- 新規メンバーが簡単に環境構築できるようにする

## 参考リソース

### 公式ドキュメント

- [Ruby on Rails Guides](https://guides.rubyonrails.org/)
- [Rails API Documentation](https://api.rubyonrails.org/)
- [Rails 8.1 Release Notes](https://guides.rubyonrails.org/8_1_release_notes.html)
- [Rails Security Guide](https://guides.rubyonrails.org/security.html)

### コミュニティリソース

- [Rails Forum](https://discuss.rubyonrails.org/)
- [Ruby on Rails Tutorial](https://www.railstutorial.org/)
- [Kamal Deploy](https://kamal-deploy.org/)
- [Hotwire](https://hotwired.dev/)
