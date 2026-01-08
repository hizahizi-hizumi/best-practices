## 概要

Ruby 3.4は2024年12月25日にリリースされ、コミュニティから高い評価を受けています。ShopifyのYJITチームが主導するパフォーマンス改善、Prismパーサーのデフォルト化、新しい言語機能の追加など、実用性の高い改善が特徴です。特にRailsアプリケーションにおいて15-30%のパフォーマンス向上が報告されており、プロダクション環境での採用が進んでいます。

主な使用シーン：

- 高トラフィックなRailsアプリケーション（Shopify、GitHub等の実績あり）
- CPU集約型のRubyワークロード
- マイクロサービスアーキテクチャのAPI
- リアルタイム処理を要求される金融系アプリケーション

## セットアップのベストプラクティス

### 基本的なインストール

```bash
# rbenvでのインストール
rbenv install 3.4.0
rbenv global 3.4.0

# または、asdfでのインストール
asdf install ruby 3.4.0
asdf global ruby 3.4.0
```

### YJIT有効化の推奨設定

コミュニティでは、本番環境でのYJIT有効化が推奨されています。

```ruby
# config/application.rb
class Application < Rails::Application
  if RubyVM::YJIT.respond_to?(:enable)
    RubyVM::YJIT.enable unless Rails.env.test?

    if Rails.env.production?
      # 本番環境用の推奨設定
      ENV['RUBY_YJIT_MIN_CALLS'] = '20'
      ENV['RUBY_YJIT_EXEC_MEM_SIZE'] = '512'  # MB
      ENV['RUBY_YJIT_CODE_GC'] = '1'  # メモリリーク防止
    end
  end
end
```

### Dockerコンテナでの推奨設定

```dockerfile
FROM ruby:3.4-alpine

# YJIT用環境変数
ENV RUBY_YJIT_ENABLE=1
ENV RUBY_YJIT_MIN_CALLS=30
ENV RUBY_YJIT_EXEC_MEM_SIZE=256

# YJITのために追加メモリを確保
# 通常のメモリ要件 + 256MB
```

参考リンク：

- [Ruby 3.4 YJIT Performance Guide](https://jetthoughts.com/blog/ruby-3-4-yjit-performance-guide/)

## 実践的な使用パターン

### 開発環境での活用

#### 新しいデフォルトブロックパラメータ `it`

Ruby 3.4では `it` がデフォルトブロックパラメータとして導入され、コードがより読みやすくなりました。

```ruby
# Ruby 3.3以前: 番号付きパラメータ
[1, 2, 3].map { _1 ** 2 }

# Ruby 3.4: itパラメータ（より直感的）
[1, 2, 3].map { it ** 2 }

# ネストされたブロックでも動作
[[1, 2], [3, 4]].each { it.each { puts it } }
# 出力: 1, 2, 3, 4
# 内側のitは内側のブロックを参照
```

**コミュニティのベストプラクティス**：

- 単純な一行ブロックには `it` を使用
- 複雑なロジックや複数行の場合は明示的なパラメータ名を使用
- `it` とローカル変数名の衝突に注意

#### Frozen String Literalsの移行

Ruby 3.4ではstring literalsがデフォルトで"chilled"状態になり、Ruby 4.0で完全にfreezeされる予定です。

```ruby
# 警告を有効化して移行準備
Warning[:deprecated] = true

# 変更可能な文字列が必要な場合
buf = +""  # 明示的に変更可能にする
buf << "test"  # 警告なし

# 文字列補間は自動的に変更可能
"#{value}" << "more text"  # 警告なし
```

**移行戦略**（GitHubの事例より）：

1. 開発環境で `Warning[:deprecated] = true` を有効化
2. すべての警告を修正
3. 不要な `# frozen_string_literal: true` コメントを削除
4. 変更可能な文字列が必要な箇所に `+""` を追加

### CI/CD統合

#### 段階的ロールアウト

Shopify、GitHubの事例から学ぶ段階的デプロイメント：

```ruby
# フィーチャーフラグによる段階的有効化
class YjitRollout
  def self.enable_for_percentage(percentage)
    pod_id = ENV['POD_ID'].to_i
    ENV['RUBY_YJIT_ENABLE'] = '1' if pod_id % 100 < percentage
  end
end

# デプロイ初期: 10%のトラフィックで有効化
YjitRollout.enable_for_percentage(10)
```

**推奨ロールアウトスケジュール**：

1. Week 1: ステージング環境で本番同等の負荷テスト
2. Week 2: 10%の本番トラフィックで有効化、メトリクス監視
3. Week 3: 問題なければ50%まで拡大
4. Week 4: 全面展開

#### メモリ監視の実装

```ruby
# YJITメモリ監視（本番環境で必須）
class YjitMemoryMonitor
  MEMORY_THRESHOLD_MB = 2048

  def self.check_and_alert
    return unless RubyVM::YJIT.enabled?

    stats = RubyVM::YJIT.runtime_stats
    memory_mb = stats[:exec_mem_size] / 1024 / 1024

    if memory_mb > MEMORY_THRESHOLD_MB
      alert_high_memory(memory_mb, stats)
      GC.start if ENV['RUBY_YJIT_CODE_GC'] == '1'
    end
  end

  def self.alert_high_memory(memory_mb, stats)
    Rails.logger.warn(
      "YJIT memory high: #{memory_mb}MB, " \
      "compiled blocks: #{stats[:compiled_block_count]}"
    )
  end
end
```

## よくある問題と解決策

### メモリリークへの対処

**問題**: 長時間稼働するプロセスでメモリ使用量が増え続ける

**原因**: コンパイルされたコードがGCされずに蓄積

**解決策**:

```ruby
# config/initializers/yjit_memory_management.rb
if RubyVM::YJIT.enabled?
  # コードGCを有効化
  ENV['RUBY_YJIT_CODE_GC'] = '1'

  # 定期的なメモリチェック
  Thread.new do
    loop do
      sleep 3600  # 1時間ごと
      YjitMemoryMonitor.check_and_alert
    end
  end
end
```

### Deoptimizationの高発生率

**問題**: パフォーマンスが時間経過とともに劣化

**原因**: 動的なメソッド定義やオブジェクト構造の変化

**回避方法**:

```ruby
# 悪い例: 動的メソッド定義がdeoptimizationを引き起こす
class DynamicModel
  def self.create_accessor(name)
    define_method(name) do
      instance_variable_get("@#{name}")
    end
  end
end

# 良い例: 静的にメソッドを定義
class StaticModel
  ATTRIBUTES = [:name, :email, :age].freeze

  ATTRIBUTES.each do |attr|
    define_method(attr) do
      instance_variable_get("@#{attr}")
    end
  end
end
```

**オブジェクト構造の一貫性を保つ**:

```ruby
# 悪い例: 条件付きインスタンス変数がオブジェクト構造を変化させる
class User
  def initialize(attrs = {})
    @name = attrs[:name]
    @email = attrs[:email]
    @admin = attrs[:admin] if attrs.key?(:admin)  # 構造が不定
  end
end

# 良い例: 常に同じ構造を維持
class User
  def initialize(attrs = {})
    @name = attrs[:name]
    @email = attrs[:email]
    @admin = attrs[:admin] || false  # 常に定義
    @created_at = attrs[:created_at] || Time.current
  end
end
```

### Range#sizeの互換性問題

**問題**: `Range#size` がTypeErrorを発生させる

```ruby
# Ruby 3.4では以下がエラーになる
(0.5..1.3).size  # TypeError: can't iterate from Float
(Time.now..Time.now + 1).size  # TypeError: can't iterate from Time
```

**解決策**: イテレート可能な範囲には `#count` を使用

```ruby
# 文字列範囲のカウント
('a'..'z').count  #=> 26

# 日付範囲のカウント（ActiveSupport使用時）
(Date.today..Date.today + 7).count  #=> 8
```

### **nilのキーワード引数への対応

Ruby 3.4では `**nil` が空のキーワード引数として扱われます。

```ruby
def handle_options(**kwargs)
  p kwargs
end

# Ruby 3.4で動作
extra_options = some_condition? ? {foo: 1} : nil
handle_options(**extra_options)
# some_conditionがfalseの場合、{}が渡される
```

## パフォーマンス最適化

### YJITの効果的な活用

#### ウォームアップ戦略

Shopifyの事例から学ぶウォームアップパターン：

```ruby
# config/initializers/yjit_warmup.rb
class YjitWarmup
  def self.perform
    return unless RubyVM::YJIT.enabled?

    # クリティカルパスのウォームアップ
    warm_up_product_catalog
    warm_up_order_processing
    warm_up_user_authentication

    log_warmup_stats
  end

  private

  def self.warm_up_product_catalog
    # 商品閲覧パターンをシミュレート
    50.times do |i|
      Product.featured.limit(20).includes(:variants, :images).to_a
      Product.find(i % 1000 + 1) rescue nil
    end
  end

  def self.warm_up_order_processing
    # 注文計算ロジックを実行
    sample_cart = Cart.new
    10.times { sample_cart.add_sample_product }
    sample_cart.calculate_total
  end

  def self.log_warmup_stats
    stats = RubyVM::YJIT.runtime_stats
    Rails.logger.info(
      "YJIT warmup completed: " \
      "#{stats[:compiled_block_count]} blocks compiled"
    )
  end
end

# デプロイ後に実行
Rails.application.config.after_initialize do
  YjitWarmup.perform if Rails.env.production?
end
```

#### パフォーマンスメトリクスの監視

```ruby
# lib/yjit_dashboard.rb
class YjitDashboard
  def self.metrics
    return nil unless RubyVM::YJIT.enabled?

    stats = RubyVM::YJIT.runtime_stats

    {
      compilation_ratio: compilation_percentage(stats),
      exit_percentage: exit_percentage(stats),
      memory_utilization: memory_utilization_percentage(stats),
      health_score: calculate_health_score(stats)
    }
  end

  def self.compilation_percentage(stats)
    return 0 if stats[:iseq_count] == 0
    (stats[:compiled_iseq_count].to_f / stats[:iseq_count] * 100).round(2)
  end

  def self.exit_percentage(stats)
    total_exits = stats[:side_exit_count] || 0
    total_calls = stats[:compiled_method_calls] || 1
    (total_exits.to_f / total_calls * 100).round(2)
  end

  def self.calculate_health_score(stats)
    score = 100
    score -= 20 if compilation_percentage(stats) < 30
    score -= 15 if exit_percentage(stats) > 15
    score -= 10 if memory_utilization_percentage(stats) > 90
    [score, 0].max
  end
end
```

### ベンチマーク結果（コミュニティ報告）

#### Railsアプリケーション

- **Shopify**: 18%のレスポンスタイム改善、年間240万ドルのインフラコスト削減
- **GitHub**: メインアプリケーション24%改善、API 31%スループット向上
- **E-learningプラットフォーム**: 27%のコンテンツレンダリング改善

#### CPU集約型ワークロード

```ruby
# フィボナッチ計算（再帰）
# Ruby 3.3: 0.85 i/s
# Ruby 3.4 + YJIT: 1.41 i/s (1.66x faster)

# 文字列処理（1MBテキストファイル）
# Ruby 3.3: 2.1 i/s
# Ruby 3.4 + YJIT: 2.9 i/s (1.38x faster)

# JSON API シリアライゼーション
# Ruby 3.3: 89.4 i/s
# Ruby 3.4 + YJIT: 127.8 i/s (1.43x faster)
```

### パフォーマンスが向上するコードパターン

#### メソッドチェーン

```ruby
# YJITは以下のようなパターンで45%の改善
User.query
  .where(active: true)
  .order(:name)
  .limit(50)
  .to_sql
```

#### 数値計算

```ruby
# 金融計算で35%の改善
class PriceCalculator
  def calculate_total(line_items)
    subtotal = line_items.sum { |item| item.price * item.quantity }
    tax = calculate_tax(subtotal)
    shipping = calculate_shipping(line_items)
    
    subtotal + tax + shipping
  end
end
```

## チーム開発での運用

### 開発環境の標準化

```ruby
# .ruby-version
3.4.0

# Gemfile
ruby "3.4.0"

# .env.development（チーム共有）
RUBY_YJIT_ENABLE=0  # 開発環境では無効化推奨
RUBY_YJIT_STATS=0   # オーバーヘッド削減
```

### CI/CDパイプラインの設定

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        ruby: ['3.3', '3.4']
        yjit: ['0', '1']
    
    env:
      RUBY_YJIT_ENABLE: ${{ matrix.yjit }}
    
    steps:
      - uses: actions/checkout@v3
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ matrix.ruby }}
          bundler-cache: true
      
      - name: Run tests
        run: bundle exec rspec
      
      - name: YJIT stats
        if: matrix.yjit == '1'
        run: |
          echo "YJIT compilation stats:"
          ruby -e 'puts RubyVM::YJIT.runtime_stats' || echo "YJIT not available"
```

### モニタリングとアラート設定

```ruby
# lib/monitoring/yjit_alerting.rb
module Monitoring
  class YjitAlerting
    THRESHOLDS = {
      memory_utilization: 85,
      exit_percentage: 20,
      compilation_errors: 5
    }

    def self.check_and_alert
      metrics = YjitDashboard.metrics
      return unless metrics

      check_memory_threshold(metrics)
      check_exit_rate(metrics)
      check_compilation_errors(metrics)
    end

    def self.check_memory_threshold(metrics)
      utilization = metrics[:memory_utilization]
      return if utilization < THRESHOLDS[:memory_utilization]

      AlertManager.send_alert(
        severity: :warning,
        title: "YJIT High Memory Usage",
        message: "Memory utilization: #{utilization}%",
        tags: ['yjit', 'memory']
      )
    end
  end
end
```

### チームでの知識共有

**ドキュメント化の推奨項目**：

1. YJIT有効化の判断基準と設定値
2. パフォーマンスベースライン測定結果
3. Deoptimization を引き起こすコードパターンのリスト
4. インシデント対応手順（メモリリーク等）

**定期レビュー項目**：

- 月次: YJITメトリクスのレビュー
- 四半期: パフォーマンスベンチマークの再測定
- 半期: Ruby新バージョンへの移行計画

## Prismパーサーの活用

Ruby 3.4ではPrismがデフォルトパーサーになりました。

### パーサー切り替え

```bash
# 従来のパーサーを使用する場合
ruby --parser=parse.y script.rb

# Prismパーサー（デフォルト）
ruby script.rb
```

### エラーメッセージの改善

Prismはより詳細でわかりやすいエラーメッセージを提供します。

```ruby
# Ruby 3.3のエラー
# test.rb:3:in `/': divided by 0 (ZeroDivisionError)
#   from test.rb:3:in `divide'

# Ruby 3.4のエラー（Prism）
# test.rb:3:in 'Integer#/': divided by 0 (ZeroDivisionError)
#   from test.rb:3:in 'Calculator#divide'
#   from test.rb:8:in '<main>'
```

モジュール名とメソッド名が表示され、デバッグが容易になります。

## ガベージコレクションの最適化

### モジュラーGCの活用

```bash
# 代替GC実装の有効化（実験的）
export RUBY_GC_LIBRARY=mmtk  # MMTk GC（Rust実装）
ruby my_app.rb
```

### GC設定の調整

```ruby
# Major GCの制御
GC.config(rgengc_allow_full_mark: false)  # Major GCを無効化
# メモリ解放よりレスポンスタイムを優先する場合

# 設定の確認
GC.config
#=> {rgengc_allow_full_mark: false, implementation: "default"}
```

**使用ケース**：

- リクエスト処理中のレイテンシを最小化したい場合
- 十分なメモリがあり、GC頻度を減らしたい場合

## 移行ガイドライン

### Ruby 3.3からRuby 3.4への移行チェックリスト

#### 事前準備

- [ ] テストカバレッジを80%以上に
- [ ] 依存gemのRuby 3.4互換性を確認
- [ ] ステージング環境を用意

#### 互換性確認

```ruby
# 互換性チェックスクリプト
class Ruby34CompatibilityChecker
  def self.run
    check_range_size_usage
    check_keyword_splatting
    check_frozen_string_literals
    check_block_arguments
  end

  def self.check_range_size_usage
    # Float/TimeのRange#size使用を検索
    puts "Checking for Range#size on non-iterables..."
    # grep or静的解析ツール使用
  end

  def self.check_frozen_string_literals
    puts "Checking for mutable string operations..."
    # 文字列変更操作を検索
  end
end
```

#### 段階的移行

1. **Week 1**: 開発環境でRuby 3.4テスト
2. **Week 2**: CI/CDでRuby 3.4ビルド追加
3. **Week 3**: ステージング環境移行
4. **Week 4**: 本番環境10%ロールアウト
5. **Week 5-6**: 段階的に100%展開

#### ロールバック計画

```yaml
# Kubernetes deployment with rollback
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: app
        image: myapp:ruby-3.4
        env:
        - name: RUBY_VERSION
          value: "3.4.0"
        - name: RUBY_YJIT_ENABLE
          value: "1"
```

## 参考リンク

### 公式リソース

- [Ruby 3.4.0 リリースノート](https://www.ruby-lang.org/en/news/2024/12/25/ruby-3-4-0-released/)
- [Ruby 3.4 Changes（詳細版）](https://rubyreferences.github.io/rubychanges/3.4.html)
- [Ruby 3.4 NEWS.md](https://github.com/ruby/ruby/blob/v3_4_0/NEWS.md)

### コミュニティ記事

- [Ruby 3.4 YJIT Performance Guide - JetThoughts](https://jetthoughts.com/blog/ruby-3-4-yjit-performance-guide/)
- [What's New in Ruby 3.4 - Honeybadger](https://www.honeybadger.io/blog/ruby-3-4/)
- [What Is New In Ruby 3.4 - Saeloun](https://blog.saeloun.com/2024/12/19/what-is-new-in-ruby-3-4/)
- [Ruby 3.4 + Rails 8 YJIT Boost - Medium](https://medium.com/@backendbyeli/ruby-3-4-rails-8-yjit-boost-why-your-app-will-run-faster-8a85cc2dd0a6)

### プロダクション事例

- [Shopify YJIT Development Blog](https://shopify.engineering/yjit-just-in-time-compiler-cruby)
- [Ruby 3.4.0 Release Impact - IronIn](https://www.ironin.it/blog/ruby-3.4-release-tech-and-business-impact.html)

### ツールとライブラリ

- [Prism Parser](https://github.com/ruby/prism)
- [Ruby Documentation](https://docs.ruby-lang.org/en/3.4/)
- [stdgems.org - Standard Library Gems](https://stdgems.org/)

### ディスカッション

- [Ruby Discourse](https://discuss.ruby-lang.org/)
- [Ruby Bug Tracker](https://bugs.ruby-lang.org/)
- [Hacker News Discussion - Ruby 3.4](https://news.ycombinator.com/item?id=42507312)
