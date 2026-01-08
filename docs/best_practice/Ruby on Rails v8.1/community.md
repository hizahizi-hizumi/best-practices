## 概要

Ruby on Rails 8.1は、2025年10月にリリースされた革新的なアップデートで、コミュニティから高い評価を受けています。ShopifyやHEYといった大規模プロダクションアプリケーションで数ヶ月にわたって運用されており、その安定性が実証されています。このバージョンでは、**Active Job Continuations**、**Structured Event Reporting**、**Local CI**などの実践的な機能が追加され、モダンなWeb開発をさらに効率化しています。

主な使用シーン：
- Kamalを使用したコンテナベースのデプロイメント
- 大規模バックグラウンドジョブの処理
- 構造化ログとイベントトラッキング
- ローカルでの統合テスト環境の構築
- Hotwireを活用したSPA風のリアルタイムアプリケーション

参考リンク：
- [Rails 8.1 公式リリースノート](https://guides.rubyonrails.org/8_1_release_notes.html)
- [Rails 8.1 発表記事](https://rubyonrails.org/2025/10/22/rails-8-1)

## セットアップのベストプラクティス

### 新規プロジェクトの作成

Rails 8.1では、セットアップが大幅に簡略化されています。

```bash
# Rails 8.1のインストール
gem install rails -v 8.1.1

# 新規プロジェクトの作成
rails new myapp

# デフォルトで以下が含まれる：
# - Kamal for deployment
# - Solid Queue for background jobs
# - Solid Cache for caching
# - Solid Cable for WebSocket
# - Propshaft for asset pipeline
```

### 認証システムのセットアップ

Rails 8では、ビルトイン認証ジェネレーターが追加され、Deviseのような複雑なgemに依存する必要がなくなりました。

```bash
# 基本認証の生成
rails generate authentication

# 生成されるファイル：
# - app/models/user.rb
# - app/models/session.rb
# - app/controllers/sessions_controller.rb
# - app/controllers/passwords_controller.rb
# - db/migrate/[timestamp]_create_users.rb
```

**プロダクションでの推奨設定：**

```ruby
# config/application.rb
config.force_ssl = true
config.session_store :cookie_store, 
  key: '_myapp_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax
```

参考リンク：
- [Rails 8 Authentication Generator Guide](https://www.bigbinary.com/blog/rails-8-introduces-a-basic-authentication-generator)
- [Extending Rails Authentication](https://nts.strzibny.name/rails-authentication-registrations/)

### Kamalデプロイメントの初期設定

```bash
# Kamal設定ファイルの初期化
bundle add kamal
kamal init

# config/deploy.yml の重要な設定
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
    options:
      network: "private"

registry:
  server: ghcr.io
  username: myusername
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
```

**セキュリティベストプラクティス：**

```bash
# Rails credentialsからKamalのシークレットを取得（Rails 8.1新機能）
kamal secrets fetch

# ホストのセキュリティ強化
# - SSHキーベース認証の設定
# - ファイアウォール設定（ufw）
# - 自動セキュリティアップデートの有効化
```

参考リンク：
- [Kamal Security Best Practices](https://paraxial.io/blog/kamal-security)
- [Kamal Deployment Tutorial](https://rameerez.com/kamal-tutorial-how-to-deploy-a-postgresql-rails-app/)

## 実践的な使用パターン

### Active Job Continuationsの活用

Rails 8.1の最大の目玉機能の一つ。長時間実行されるジョブを中断可能なステップに分割できます。

```ruby
# app/jobs/process_import_job.rb
class ProcessImportJob < ApplicationJob
  include ActiveJob::Continuable
  
  def perform(import_id)
    @import = Import.find(import_id)
    
    # ブロック形式のステップ
    step :initialize do
      @import.initialize!
      logger.info "Import initialized: #{import_id}"
    end
    
    # カーソル付きステップ（中断・再開が可能）
    step :process do |step|
      @import.records.find_each(start: step.cursor) do |record|
        record.process!
        
        # カーソルを更新して進捗を保存
        step.advance! from: record.id
      end
    end
    
    # メソッド形式のステップ
    step :finalize
    
    # 後処理ステップ
    step :notify do
      ImportNotificationMailer.completed(@import).deliver_later
    end
  end
  
  private
  
  def finalize
    @import.mark_as_completed!
  end
end
```

**実践的なユースケース：**

1. **大規模データインポート**：CSVやJSONファイルから数万件のレコードをインポート
2. **バッチ処理**：定期的なレポート生成やデータクリーンアップ
3. **外部API連携**：大量のAPIリクエストを段階的に処理

**Kamalデプロイメントとの相性：**
- Kamalのデフォルトshutdownタイムアウトは30秒
- Continuationsにより、デプロイ時にジョブが最初からやり直しにならない
- デプロイの信頼性が向上し、ダウンタイムが最小化

参考リンク：
- [Active Job Continuations Guide](https://railsportal.com/blog/active-job-continuations-in-rails-8-1-what-they-solve-and-how-to-use-them)
- [MarsBased Active Job Article](https://marsbased.com/blog/2025/10/15/active-job-continuations-the-end-of-lost-jobs)

### Structured Event Reportingの実装

Rails 8.1の新しいイベントレポーティングシステムは、構造化ログとモニタリングを簡単に実現します。

```ruby
# 基本的なイベント通知
class UsersController < ApplicationController
  def create
    @user = User.create!(user_params)
    
    # 構造化イベントの発行
    Rails.event.notify("user.signup", 
      user_id: @user.id, 
      email: @user.email,
      plan: @user.plan
    )
    
    redirect_to dashboard_path
  end
end
```

**タグとコンテキストの活用：**

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  around_action :add_event_context
  
  def execute
    Rails.event.tagged("graphql", "api") do
      result = MySchema.execute(
        params[:query],
        variables: params[:variables],
        context: { current_user: current_user }
      )
      
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

**カスタムサブスクライバーの作成：**

```ruby
# config/initializers/event_subscriber.rb
class JsonLogSubscriber
  def emit(event)
    log_data = {
      timestamp: Time.current.iso8601,
      event: event[:name],
      payload: event[:payload],
      tags: event[:tags],
      context: event[:context],
      source: "#{event[:source_location][:filepath]}:#{event[:source_location][:lineno]}"
    }
    
    Rails.logger.info(JSON.generate(log_data))
  end
end

# DataDog、NewRelic等への送信用サブスクライバー
class DatadogSubscriber
  def emit(event)
    Datadog::Statsd.new.event(
      event[:name],
      event[:payload].to_json,
      tags: event[:tags].keys
    )
  end
end

Rails.event.subscribe(JsonLogSubscriber.new)
Rails.event.subscribe(DatadogSubscriber.new) if Rails.env.production?
```

参考リンク：
- [Rails 8.1 Structured Events](https://guides.rubyonrails.org/8_1_release_notes.html#structured-event-reporting)

### Hotwire/Turboパターン

Rails 8では、HotwireがデフォルトのJavaScriptソリューションとなり、コミュニティでのベストプラクティスが確立されています。

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  def create
    @message = @conversation.messages.create!(message_params)
    
    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          # メッセージリストに追加
          turbo_stream.append("messages", 
            partial: "messages/message", 
            locals: { message: @message }
          ),
          # フォームをリセット
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

**Turbo Framesの遅延読み込み：**

```erb
<!-- app/views/conversations/show.html.erb -->
<%= turbo_frame_tag "conversation-messages", 
    src: conversation_messages_path(@conversation),
    loading: :lazy do %>
  <p>メッセージを読み込み中...</p>
<% end %>
```

**リアルタイム更新（Turbo Streams + Action Cable）：**

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

**Turbo MorphingによるUX改善：**

```erb
<!-- app/views/layouts/application.html.erb -->
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
</head>
```

参考リンク：
- [Rails 8 Hotwire Guide](https://devot.team/blog/rails-8-hotwire)
- [Ultimate Hotwire Guide](https://blog.cloud66.com/the-ultimate-guide-to-implementing-hotwired-and-turbo-in-a-rails-application)

### 開発環境での活用

**Local CIの活用：**

Rails 8.1では、ローカルマシンでCI環境を実行できる新機能が追加されました。

```ruby
# config/ci.rb
ci do
  # テストスイート
  test "unit", "bin/rails test"
  test "system", "bin/rails test:system"
  test "integration", "bin/rails test:integration"
  
  # セキュリティチェック
  security "brakeman", "bin/brakeman --quiet --no-summary"
  security "bundle audit", "bundle exec bundle-audit check --update"
  
  # コード品質
  lint "rubocop", "bin/rubocop"
  lint "erblint", "bin/erblint --lint-all"
  
  # 型チェック（オプション）
  typecheck "steep", "steep check" if File.exist?("Steepfile")
end
```

```bash
# ローカルでCI全体を実行
bin/ci

# 特定のテストのみ実行
bin/ci test:unit

# 並列実行でスピードアップ
bin/ci --parallel
```

参考リンク：
- [Rails 8.1 Local CI Feature](https://blog.saeloun.com/2025/12/17/rails-introduces-ci-to-streamline-new-dsl)

### CI/CD統合

**GitHub Actionsとの統合：**

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
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.3.0
          bundler-cache: true
      
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - name: Install dependencies
        run: |
          bundle install
          yarn install
      
      - name: Setup database
        env:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
        run: |
          bin/rails db:create db:schema:load
      
      - name: Run tests
        env:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379/0
        run: |
          bin/ci
```

**Kamalを使った自動デプロイ：**

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.3.0
          bundler-cache: true
      
      - name: Install Kamal
        run: gem install kamal
      
      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      
      - name: Deploy with Kamal
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
        run: |
          kamal deploy
```

## よくある問題と解決策

### N+1クエリ問題

Rails 8でも最も一般的なパフォーマンス問題です。

**問題のあるコード：**

```ruby
# app/controllers/posts_controller.rb
def index
  @posts = Post.all
  # N+1クエリが発生！
end
```

```erb
<!-- app/views/posts/index.html.erb -->
<% @posts.each do |post| %>
  <div class="post">
    <h2><%= post.title %></h2>
    <p>著者: <%= post.author.name %></p> <!-- ここで追加のクエリ -->
    <p>コメント数: <%= post.comments.count %></p> <!-- ここでも追加のクエリ -->
  </div>
<% end %>
```

**解決策：**

```ruby
# includes, preload, eager_loadを適切に使用
def index
  @posts = Post
    .includes(:author)  # author関連のレコードをプリロード
    .left_joins(:comments)
    .select('posts.*, COUNT(comments.id) as comments_count')
    .group('posts.id')
end
```

**開発環境での検出：**

```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.bullet_logger = true
  Bullet.console = true
  Bullet.rails_logger = true
end
```

### Turbo Frameの予期しない動作

**問題：** Turbo Frameが親フレームにリダイレクトしない

```ruby
# 間違った実装
class ArticlesController < ApplicationController
  def create
    @article = Article.create(article_params)
    redirect_to @article  # Turbo Frameから抜けられない！
  end
end
```

**解決策：**

```ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.create(article_params)
    
    respond_to do |format|
      format.html { redirect_to @article }
      format.turbo_stream do
        # トップレベルにリダイレクト
        render turbo_stream: turbo_stream.action(:redirect, @article)
      end
    end
  end
end

# または、Turbo-Frame: _top ヘッダーを使用
def create
  @article = Article.create(article_params)
  redirect_to @article, turbo_frame: "_top"
end
```

### Kamalデプロイメントでの環境変数エラー

**問題：** デプロイ後に環境変数が正しく読み込まれない

```bash
# エラーログ
Missing required environment variable: DATABASE_URL
```

**解決策：**

```yaml
# config/deploy.yml
env:
  clear:
    RAILS_ENV: production
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - REDIS_URL

# Rails credentialsを使用する場合
```

```bash
# Rails credentialsに環境変数を保存
rails credentials:edit --environment production
```

```yaml
# config/credentials/production.yml.enc
database_url: postgresql://user:pass@host:5432/db
redis_url: redis://localhost:6379/1
secret_key_base: your_secret_key
```

```ruby
# config/database.yml
production:
  url: <%= Rails.application.credentials.database_url %>
```

### Solid Queueのジョブ実行遅延

**問題：** バックグラウンドジョブの処理が遅い

**診断方法：**

```ruby
# Solid Queueの状態確認
SolidQueue::Job.pending.count
SolidQueue::Job.running.count
SolidQueue::Job.failed.count
```

**解決策：**

```yaml
# config/queue.yml
production:
  dispatchers:
    - polling_interval: 1
      batch_size: 100
  workers:
    - queues: "*"
      threads: 5
      processes: 4
      polling_interval: 0.1
```

```ruby
# ジョブの優先度設定
class ImportantJob < ApplicationJob
  queue_as :critical
  
  def perform
    # 重要な処理
  end
end

# config/queue.yml
production:
  workers:
    - queues: critical
      threads: 3
      processes: 2
    - queues: "*"
      threads: 5
      processes: 2
```

参考リンク：
- [Common Rails Pitfalls](https://medium.com/@salujabhavesh/mind-the-trap-common-pitfalls-in-rails-transactions-and-how-to-avoid-them-81d5dd70e9be)

## パフォーマンス最適化

### データベースクエリの最適化

**1. 適切なインデックスの追加：**

```ruby
# db/migrate/20250108000001_add_performance_indexes.rb
class AddPerformanceIndexes < ActiveRecord::Migration[8.1]
  def change
    # 頻繁に検索される外部キー
    add_index :posts, :user_id
    add_index :comments, [:post_id, :created_at]
    
    # 複合インデックス（WHERE句に複数条件）
    add_index :posts, [:published, :created_at]
    
    # 部分インデックス（条件付き）
    add_index :posts, :created_at, 
      where: "published = true", 
      name: "index_posts_published_on_created_at"
    
    # ユニークインデックス
    add_index :users, :email, unique: true
  end
end
```

**2. カウンターキャッシュの活用：**

```ruby
class Post < ApplicationRecord
  has_many :comments, counter_cache: true
end

class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# マイグレーション
class AddCommentsCountToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
    
    # 既存データの更新
    reversible do |dir|
      dir.up do
        Post.find_each do |post|
          Post.reset_counters(post.id, :comments)
        end
      end
    end
  end
end
```

**3. バッチ処理での効率化：**

```ruby
# メモリ効率の良い処理
Post.find_each(batch_size: 1000) do |post|
  post.update_metrics!
end

# 特定の条件で処理
Post.where(published: true).in_batches(of: 500) do |batch|
  batch.update_all(processed_at: Time.current)
end
```

### キャッシング戦略

Rails 8では、Solid Cacheがデフォルトになりました。

**1. フラグメントキャッシング：**

```erb
<!-- app/views/posts/show.html.erb -->
<% cache @post do %>
  <article>
    <h1><%= @post.title %></h1>
    <%= @post.body %>
  </article>
<% end %>

<% cache [@post, "comments-section"] do %>
  <div id="comments">
    <%= render @post.comments %>
  </div>
<% end %>
```

**2. ロシアンドールキャッシング：**

```erb
<!-- app/views/posts/index.html.erb -->
<% cache ["posts-v1", Post.maximum(:updated_at)] do %>
  <div class="posts">
    <% @posts.each do |post| %>
      <% cache post do %>
        <%= render post %>
      <% end %>
    <% end %>
  </div>
<% end %>
```

**3. 低レベルキャッシング：**

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

**4. HTTPキャッシング：**

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    # ETags
    fresh_when(@post)
    
    # または Last-Modified
    # fresh_when(last_modified: @post.updated_at, etag: @post)
    
    # 明示的なキャッシュヘッダー
    # expires_in 1.hour, public: true
  end
end
```

### アセットとフロントエンドの最適化

```ruby
# config/environments/production.rb
config.public_file_server.headers = {
  "Cache-Control" => "public, max-age=31536000, immutable"
}

# 画像の最適化
# - WebP形式の使用
# - 適切なサイズへのリサイズ
# - 遅延読み込み
```

```erb
<!-- app/views/posts/_post.html.erb -->
<%= image_tag @post.image.variant(resize_to_limit: [800, 600]), 
    loading: :lazy,
    srcset: {
      @post.image.variant(resize_to_limit: [400, 300]) => "400w",
      @post.image.variant(resize_to_limit: [800, 600]) => "800w"
    } %>
```

### パフォーマンス測定ツール

```ruby
# Gemfile
group :development do
  gem 'rack-mini-profiler'
  gem 'memory_profiler'
  gem 'stackprof'
  gem 'bullet'
end
```

```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
end
```

参考リンク：
- [Rails Performance Best Practices](https://medium.com/@sophiasmith791/best-practices-for-optimizing-ruby-on-rails-performance-b38ee08cc8dc)
- [High-Performance Rails Guide](https://medium.com/@risharma654/the-ultimate-guide-to-high-performance-ruby-on-rails-applications-10c1042015de)

## チーム開発での運用

### コード規約の統一

**1. Rubocopの設定：**

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
    - 'node_modules/**/*'
    - 'vendor/**/*'

Style/Documentation:
  Enabled: false

Layout/LineLength:
  Max: 120
  Exclude:
    - 'config/**/*'

Metrics/BlockLength:
  Exclude:
    - 'spec/**/*'
    - 'config/routes.rb'
```

```bash
# 自動修正の実行
bin/rubocop -A

# CIでの実行
bin/rubocop --parallel
```

**2. Git Hooksの活用：**

```bash
# .git/hooks/pre-commit
#!/bin/bash

# Rubocopの実行
echo "Running Rubocop..."
bundle exec rubocop $(git diff --cached --name-only --diff-filter=ACM | grep '\.rb$')

if [ $? -ne 0 ]; then
  echo "Rubocop failed. Please fix the errors before committing."
  exit 1
fi

# テストの実行
echo "Running tests..."
bundle exec rails test

if [ $? -ne 0 ]; then
  echo "Tests failed. Please fix the errors before committing."
  exit 1
fi
```

### Pull Requestのベストプラクティス

**1. PRテンプレートの作成：**

```markdown
<!-- .github/pull_request_template.md -->
## 変更内容

<!-- この PR で何を変更したか簡潔に説明してください -->

## 変更理由

<!-- なぜこの変更が必要なのか説明してください -->

## チェックリスト

- [ ] テストを追加/更新した
- [ ] ドキュメントを更新した
- [ ] マイグレーションが必要な場合、rollbackを確認した
- [ ] N+1クエリをチェックした
- [ ] セキュリティへの影響を検討した
- [ ] パフォーマンスへの影響を検討した

## スクリーンショット（該当する場合）

<!-- UI変更がある場合はスクリーンショットを添付 -->

## 関連Issue

<!-- Closes #123 のように関連するissueを記載 -->
```

**2. コードレビューガイドライン：**

- Rails規約に従っているか
- セキュリティ問題はないか（SQLインジェクション、XSS等）
- N+1クエリが発生していないか
- テストカバレッジは十分か
- ドキュメントは更新されているか

### ブランチ戦略

```bash
# 機能ブランチの作成
git checkout -b feature/user-authentication

# 作業完了後
git push origin feature/user-authentication

# main/masterへのマージはPR経由のみ
```

**推奨ブランチ命名規則：**
- `feature/機能名` - 新機能
- `fix/バグ名` - バグ修正
- `refactor/対象` - リファクタリング
- `docs/対象` - ドキュメント更新

### 環境ごとの設定管理

```ruby
# config/credentials.yml.enc（暗号化）の使用

# 開発環境
EDITOR="code --wait" rails credentials:edit --environment development

# ステージング環境
EDITOR="code --wait" rails credentials:edit --environment staging

# 本番環境
EDITOR="code --wait" rails credentials:edit --environment production
```

```ruby
# 環境ごとの設定値へのアクセス
Rails.application.credentials.api_key
Rails.application.credentials.dig(:aws, :access_key_id)
```

### チーム共有のセットアップスクリプト

```bash
# bin/setup
#!/usr/bin/env ruby
require "fileutils"

# アプリケーションディレクトリへ移動
APP_ROOT = File.expand_path("..", __dir__)
Dir.chdir(APP_ROOT) do
  puts "== Installing dependencies =="
  system! "gem install bundler --conservative"
  system("bundle check") || system!("bundle install")
  system!("bin/yarn")

  puts "\n== Preparing database =="
  system! "bin/rails db:prepare"

  puts "\n== Removing old logs and tempfiles =="
  system! "bin/rails log:clear tmp:clear"

  puts "\n== Installing git hooks =="
  system! "ln -sf ../../.git-hooks/pre-commit .git/hooks/pre-commit"

  puts "\n== Setting up test database =="
  system! "bin/rails db:test:prepare"

  puts "\n== Running tests to verify setup =="
  system! "bin/rails test"

  puts "\n== All set! =="
end
```

### ドキュメンテーション

**READMEに含めるべき情報：**

```markdown
# プロジェクト名

## セットアップ

\`\`\`bash
# リポジトリのクローン
git clone https://github.com/your-org/your-app.git
cd your-app

# セットアップスクリプトの実行
bin/setup
\`\`\`

## 開発サーバーの起動

\`\`\`bash
bin/dev
\`\`\`

## テストの実行

\`\`\`bash
# 全テスト
bin/rails test

# 特定のテスト
bin/rails test test/models/user_test.rb

# システムテスト
bin/rails test:system
\`\`\`

## デプロイ

\`\`\`bash
# ステージング環境
kamal deploy --destination staging

# 本番環境
kamal deploy --destination production
\`\`\`

## トラブルシューティング

### データベース接続エラー
...

### アセットのコンパイルエラー
...
\`\`\`
```

参考リンク：
- [Ruby on Rails Best Practices](https://learnetto.com/tutorials/ruby-on-rails-best-practices)
- [Rails Style Guide](https://github.com/rubocop/ruby-style-guide)

## セキュリティベストプラクティス

### 強力なパラメータの使用

```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    # ...
  end
  
  private
  
  def user_params
    params.require(:user).permit(
      :email, 
      :password, 
      :password_confirmation,
      :name
    )
  end
end
```

### CSRFトークンの適切な処理

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  
  # APIモードの場合
  # protect_from_forgery with: :null_session
end
```

### SQLインジェクション対策

```ruby
# 危険（使用禁止）
User.where("name = '#{params[:name]}'")

# 安全
User.where("name = ?", params[:name])
User.where(name: params[:name])
```

### XSS対策

```erb
<!-- 自動的にエスケープされる -->
<%= @user.name %>

<!-- 明示的にエスケープ
<%= sanitize @post.body, tags: %w(p br strong em) %>

<!-- 信頼できるHTMLのみ -->
<%== @trusted_html %>
```

### セキュリティヘッダーの設定

```ruby
# config/application.rb
config.force_ssl = true

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

### 認証とセッション管理

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: '_myapp_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 14.days
```

### 定期的なセキュリティ監査

```bash
# 依存関係の脆弱性チェック
bundle audit check --update

# Brakemanによるコードスキャン
brakeman --quiet --no-summary

# CI/CDパイプラインへの組み込み
# .github/workflows/security.yml
```

```yaml
name: Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # 毎週日曜日

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Brakeman
        run: |
          gem install brakeman
          brakeman --no-pager
      
      - name: Run Bundle Audit
        run: |
          gem install bundler-audit
          bundle-audit check --update
```

参考リンク：
- [Securing Rails Applications](https://guides.rubyonrails.org/security.html)
- [Rails Security Best Practices](https://medium.com/@Anita-ihuman/ruby-on-rails-security-best-practices-for-cloud-deployments-on-upcloud-897a3347ddce)

## 移行とアップグレード

### Rails 7.xから8.1への移行

**1. 準備段階：**

```bash
# 現在のテストスイートを実行して、全てパスすることを確認
bin/rails test

# 依存関係を最新の安定版に更新
bundle update

# 非推奨警告を確認
DEPRECATION_WARNINGS=true bin/rails test
```

**2. Gemfileの更新：**

```ruby
# Gemfile
gem "rails", "~> 8.1.0"

# Rails 8のデフォルトGems
gem "solid_queue"
gem "solid_cache"
gem "solid_cable"
gem "kamal"
gem "thruster"

# 不要になった可能性のあるGems（確認して削除）
# gem "redis"  # Solid Cable/Cacheを使う場合
# gem "sidekiq"  # Solid Queueを使う場合
```

```bash
bundle update rails
```

**3. 設定ファイルの更新：**

```bash
# 新しい設定ファイルをコピー
bin/rails app:update

# 差分を確認して、必要な変更を適用
```

**4. データベースマイグレーション：**

```bash
# Solid Queue, Solid Cache, Solid Cableのテーブルを作成
bin/rails solid_queue:install
bin/rails solid_cache:install
bin/rails solid_cable:install

bin/rails db:migrate
```

**5. 本番環境での段階的ロールアウト：**

```ruby
# 新機能を段階的に有効化
# config/environments/production.rb
config.active_job.queue_adapter = if ENV["USE_SOLID_QUEUE"] == "true"
  :solid_queue
else
  :sidekiq  # 既存のアダプター
end
```

**6. モニタリングと検証：**

- エラー率の監視
- パフォーマンスメトリクスの比較
- ユーザーフィードバックの収集
- ロールバック計画の準備

### よくある移行時の問題

**問題1：Importmapsの動作**

```ruby
# Rails 7でSprocketsを使用していた場合
# config/importmap.rb を確認
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

**問題2：Solid Queueへの移行**

```ruby
# Sidekiqの設定を確認
# config/sidekiq.yml → config/queue.yml

# Sidekiqのカスタムワーカーの調整
class MyWorker
  include Sidekiq::Worker
  
  def perform(arg)
    # ...
  end
end

# ↓ ActiveJobに移行

class MyJob < ApplicationJob
  queue_as :default
  
  def perform(arg)
    # ...
  end
end
```

参考リンク：
- [Upgrading Ruby on Rails Guide](https://guides.rubyonrails.org/upgrading_ruby_on_rails.html)
- [Rails 8.1 Release Notes](https://guides.rubyonrails.org/8_1_release_notes.html)

## まとめ

Rails 8.1は、実践的な機能追加と改善により、モダンなWeb開発をさらに効率化しています。特に以下の点が注目されています：

1. **Active Job Continuations** - 長時間実行ジョブの中断・再開が可能になり、Kamalデプロイメントとの相性が向上
2. **Structured Event Reporting** - 構造化ログにより、モニタリングとデバッグが容易に
3. **Local CI** - ローカルマシンでのCI実行により、開発サイクルが高速化
4. **Solid Trifecta** - データベースベースのジョブキュー、キャッシュ、WebSocketにより、依存関係が削減
5. **ビルトイン認証** - Deviseなしでセキュアな認証システムを簡単に構築

これらの機能を適切に活用することで、より保守性が高く、スケーラブルなRailsアプリケーションを構築できます。

### 参考リソース

**公式ドキュメント：**
- [Ruby on Rails Guides 8.1](https://guides.rubyonrails.org/)
- [Rails API Documentation](https://api.rubyonrails.org/)
- [Rails 8.1 Release Notes](https://guides.rubyonrails.org/8_1_release_notes.html)

**コミュニティリソース：**
- [Rails Forum](https://discuss.rubyonrails.org/)
- [Rails Reddit](https://www.reddit.com/r/rails/)
- [Ruby on Rails YouTube Channel](https://www.youtube.com/@railsofficial)

**学習リソース：**
- [Getting Started with Rails 8 Guide](https://guides.rubyonrails.org/getting_started.html)
- [Rails 8 Unpacked Video Series](https://typecraft.dev/rails-8-unpacked)
- [Ruby on Rails Tutorial](https://www.railstutorial.org/)

**ツールとライブラリ：**
- [Kamal](https://kamal-deploy.org/)
- [Hotwire](https://hotwired.dev/)
- [Turbo](https://turbo.hotwired.dev/)
- [Stimulus](https://stimulus.hotwired.dev/)
