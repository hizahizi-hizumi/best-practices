---
description: 'RuboCop の静的コード解析とコードフォーマットのベストプラクティス'
applyTo: '**/*.rb, **/.rubocop.yml, **/Gemfile'
---

# RuboCop ベストプラクティス

Ruby の静的コード解析ツール（リンター）およびコードフォーマッターである RuboCop の効果的な使用方法と設定のガイドライン

## 目的とスコープ

RuboCop を使用して Ruby コードの品質、一貫性、セキュリティを向上させる。Ruby Style Guide に基づいた包括的なルール群を活用し、チーム開発における統一された開発体験を実現する。

## 基本原則

- プロジェクトルートに `.rubocop.yml` 設定ファイルを配置する
- 新しいバージョンで追加される cops を有効化するため `NewCops: enable` を設定する
- `TargetRubyVersion` を明示的に指定し、使用する Ruby バージョンを明確にする
- Gemfile の開発・テストグループに `require: false` オプション付きで追加する
- Lint cops を無効化しない（潜在的なバグを検出するため）
- 自動修正後は必ずテストスイートを実行する
- Security cops は常に有効に保つ

## セットアップと設定

### インストール

Gemfile に以下を追加する：

```ruby
group :development, :test do
  gem 'rubocop', require: false
  gem 'rubocop-rails', require: false
  gem 'rubocop-rspec', require: false
  gem 'rubocop-performance', require: false
end
```

**根拠**: `require: false` により、Rails の起動時間への影響を防ぐ

### 基本設定ファイル

プロジェクトルートに `.rubocop.yml` を作成する：

```yaml
AllCops:
  NewCops: enable
  TargetRubyVersion: 3.0
  DisplayCopNames: true
  DisplayStyleGuide: true
  Exclude:
    - 'vendor/**/*'
    - 'db/schema.rb'
    - 'node_modules/**/*'

Layout/LineLength:
  Max: 120
```

**根拠**: `DisplayCopNames` と `DisplayStyleGuide` により、違反の理解と検索が容易になる

### プラグインの有効化

RuboCop 1.72+ では plugins システムを使用する：

```yaml
plugins:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance
```

**根拠**: `require:` よりも `plugins:` の方がメンテナンスが容易で推奨される

## 段階的導入戦略

### 既存プロジェクトへの導入

一度にすべての違反を修正しようとせず、以下の手順で段階的に導入する：

1. TODO ファイルを生成する：

```bash
rubocop --auto-gen-config
```

2. 生成された `.rubocop_todo.yml` を `.rubocop.yml` から継承する：

```yaml
inherit_from: .rubocop_todo.yml
```

3. ルールごとまたはファイルごとに段階的に修正する

**根拠**: 大規模なレガシーコードベースでも無理なく導入でき、チームへの負担を最小化できる

### 自動修正の活用

```bash
# 安全な自動修正のみ実行（推奨）
rubocop -a

# 安全・非安全含むすべての自動修正（注意が必要）
rubocop -A
```

**根拠**: `-A` オプションはコードのセマンティクスを変更する可能性があるため、慎重に使用する

## 実行コマンドパターン

### 基本的な使用方法

```bash
# カレントディレクトリの全 Ruby ファイルをチェック
rubocop

# 特定のファイルやディレクトリをチェック
rubocop app spec lib/something.rb

# 並列実行でパフォーマンス向上
rubocop --parallel

# 詳細情報とスタイルガイド URL を表示
rubocop -E -S

# 特定の cops のみを実行
rubocop --only Rails/Blank,Layout/HeredocIndentation
```

### 出力フォーマット

```bash
# シンプルなテキスト出力
rubocop -f simple

# JSON 形式（CI/CD 統合用）
rubocop -f json

# HTML レポート
rubocop -f html -o rubocop_report.html
```

## CI/CD 統合

### GitHub Actions

`.github/workflows/rubocop.yml` を作成する：

```yaml
name: RuboCop

on: [pull_request]

jobs:
  rubocop:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.0
          bundler-cache: true
      - name: Run RuboCop
        run: bundle exec rubocop --parallel
```

**根拠**: PR ごとに自動チェックを実行し、コードレビューの負担を軽減する

### Git pre-commit フック

新しいファイルや変更されたファイルのみをチェックする：

```bash
#!/bin/bash
# .git/hooks/pre-commit

files=$(git status -s | grep -E 'A|M' | awk '{print $2}')
files="$files $(git status -s | grep -E 'R' | awk '{print $4}')"
echo $files | xargs rubocop --display-cop-names --parallel --force-exclusion
```

実行権限を付与する：

```bash
chmod +x .git/hooks/pre-commit
```

**根拠**: コミット前にローカルでチェックし、CI の失敗を防ぐ

## Cops の設定パターン

### 行の長さ

```yaml
Layout/LineLength:
  Max: 120
  Exclude:
    - 'config/routes.rb'
```

**根拠**: Rails プロジェクトでは 120 文字が一般的で、モダンなディスプレイに適している

### ドキュメントコメント

```yaml
Style/Documentation:
  Enabled: true
  Exclude:
    - 'spec/**/*'
    - 'test/**/*'
```

**根拠**: テストファイルではドキュメントコメントは不要で、冗長になる

### メソッドの長さ

```yaml
Metrics/MethodLength:
  Max: 30
  CountComments: false
  Exclude:
    - 'db/migrate/**/*'
```

**根拠**: マイグレーションファイルは構造的に長くなる傾向があるため除外する

### ハッシュの整列

```yaml
Layout/HashAlignment:
  EnforcedColonStyle: table
  EnforcedHashRocketStyle: table
```

**推奨**:

```ruby
user = {
  name:  'John',
  email: 'john@example.com',
  role:  'admin'
}
```

**非推奨**:

```ruby
user = {
  name: 'John',
  email: 'john@example.com',
  role: 'admin'
}
```

**根拠**: テーブル形式の整列により、可読性が向上する

## セキュリティのベストプラクティス

### Security Cops を常に有効化

```yaml
Security/Eval:
  Enabled: true

Security/JSONLoad:
  Enabled: true

Security/MarshalLoad:
  Enabled: true

Security/Open:
  Enabled: true

Security/YAMLLoad:
  Enabled: true
```

**根拠**: これらの cops はセキュリティの脆弱性を検出し、安全でない API の使用を防ぐ

### 安全でないメソッドの検出例

**推奨**:

```ruby
# JSON.parse を使用（安全）
data = JSON.parse(json_string)

# YAML.safe_load を使用（安全）
config = YAML.safe_load(yaml_string, permitted_classes: [Symbol])

# URI.parse + URI.open を使用（安全）
uri = URI.parse(url)
content = uri.open.read
```

**非推奨**:

```ruby
# eval を使用（危険）
eval(user_input)

# JSON.load を使用（危険）
data = JSON.load(json_string)

# YAML.load を使用（危険）
config = YAML.load(yaml_string)

# Kernel#open を使用（コマンドインジェクションの危険性）
content = open(url).read
```

## パフォーマンス最適化

### rubocop-performance の活用

```yaml
plugins:
  - rubocop-performance

Performance/StringInclude:
  Enabled: true

Performance/RedundantBlockCall:
  Enabled: true

Performance/RegexpMatch:
  Enabled: true
```

### パフォーマンス改善例

**推奨**:

```ruby
# filter_map を使用（効率的）
array.filter_map { |item| transform(item) if condition(item) }

# include? を使用（高速）
string.include?('substring')

# match? を使用（マッチ結果が不要な場合）
return unless /pattern/.match?(string)

# 明示的な return を省略
def foo
  [1, 2, 3]
end
```

**非推奨**:

```ruby
# select + map の連鎖（非効率）
array.select { |item| condition(item) }.map { |item| transform(item) }

# 正規表現を文字列検索に使用（低速）
string =~ /substring/

# =~ や match を使用（マッチ結果が不要な場合）
return unless string =~ /pattern/

# 無駄な return 文
def foo
  return [1, 2, 3]
end
```

**根拠**: パフォーマンス最適化により、大規模データセットでの実行速度が改善される

## Rails 固有の設定

### Rails Cops の有効化

```yaml
plugins:
  - rubocop-rails

Rails:
  Enabled: true

Rails/HasManyOrHasOneDependent:
  Enabled: true

Rails/SkipsModelValidations:
  Enabled: true
  AllowedMethods:
    - touch
    - update_all

Rails/UnknownEnv:
  Environments:
    - production
    - development
    - test
    - staging
```

**根拠**: Rails 固有のベストプラクティスを強制し、一般的な落とし穴を回避する

### Rails 推奨例

**推奨**:

```ruby
# has_many に dependent オプションを指定
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
end

# find_each を使用して大量レコードを処理
User.find_each do |user|
  user.update_profile
end

# Rails.env の既知の環境のみを使用
if Rails.env.production?
  # ...
end
```

**非推奨**:

```ruby
# dependent オプションがない（メモリリークの危険性）
class User < ApplicationRecord
  has_many :posts
end

# each を使用（メモリ問題の可能性）
User.all.each do |user|
  user.update_profile
end

# カスタム環境名をハードコード
if Rails.env == 'custom_env'
  # ...
end
```

## RSpec 固有の設定

### RSpec Cops の活用

```yaml
plugins:
  - rubocop-rspec

RSpec/ExampleLength:
  Max: 10

RSpec/MultipleExpectations:
  Max: 5

RSpec/NestedGroups:
  Max: 5

RSpec/DescribedClass:
  Enabled: true
  EnforcedStyle: described_class
```

**根拠**: RSpec のベストプラクティスを強制し、テストの可読性と保守性を向上させる

### RSpec 推奨例

**推奨**:

```ruby
RSpec.describe User do
  describe '#full_name' do
    it 'returns concatenated first and last name' do
      user = described_class.new(first_name: 'John', last_name: 'Doe')
      expect(user.full_name).to eq('John Doe')
    end
  end
end
```

**非推奨**:

```ruby
RSpec.describe User do
  describe '#full_name' do
    it 'returns concatenated first and last name' do
      user = User.new(first_name: 'John', last_name: 'Doe')
      expect(user.full_name).to eq('John Doe')
    end
  end
end
```

**根拠**: `described_class` によりテストコードのリファクタリングが容易になる

## 違反の無効化パターン

### インライン無効化（最小限に使用）

```ruby
# 1 行のみ無効化
long_string = "This is a very long string that exceeds line length" # rubocop:disable Layout/LineLength

# ブロック単位で無効化
# rubocop:disable Layout/LineLength
def method_with_long_lines
  # 長い行が複数ある処理
end
# rubocop:enable Layout/LineLength
```

**根拠**: インライン無効化は例外的な場合のみ使用し、設定ファイルでの対応を優先する

### ファイル全体の除外

```yaml
Style/FrozenStringLiteralComment:
  Exclude:
    - 'db/schema.rb'
    - 'db/migrate/**/*'
```

**根拠**: 自動生成されるファイルは除外し、無駄な違反を防ぐ

## よくある落とし穴

### 大量の違反への対処ミス

**問題**: 初回実行時に数百、数千の違反が報告され、圧倒される

**解決策**:
1. `rubocop --auto-gen-config` で TODO ファイルを生成する
2. `rubocop -a` で自動修正可能な違反を一括処理する
3. 残った違反は優先度順に段階的に対応する

**根拠**: 段階的なアプローチによりチームへの負担を分散し、継続的な改善が可能になる

### キャッシュの問題

**問題**: 外部依存関係を変更した後、古いキャッシュが使用される

**解決策**:

```bash
# キャッシュを無効化して実行
rubocop --cache false

# または、キャッシュディレクトリを削除
rm -rf tmp/rubocop_cache
```

**根拠**: キャッシュの無効化により、最新の設定と依存関係を反映できる

### Security Cops の無効化

**問題**: パフォーマンスやスタイルの理由で Security cops を無効化してしまう

**解決策**: Security cops は絶対に無効化せず、代替実装を検討する

**根拠**: セキュリティの脆弱性は重大なインシデントにつながる可能性がある

### 自動修正の盲目的な適用

**問題**: `-A` オプションでセマンティクスが変わる修正を無批判に適用

**解決策**:
1. まず `-a` で安全な修正のみを適用する
2. `-A` を使用する場合は、必ずテストスイートを実行する
3. 重要な変更は手動でレビューする

**根拠**: 自動修正がコードの動作を変更する可能性があるため、検証が必要

## チーム開発での運用

### 設定ファイルの共有戦略

複数の設定ファイルから継承する：

```yaml
inherit_from:
  - .rubocop_base.yml
  - .rubocop_rails.yml
  - .rubocop_rspec.yml
```

**根拠**: 設定を分割することで、チームメンバーが理解しやすくなる

### ルール変更のプロセス

1. チーム内で議論し、合意形成を行う
2. 別のブランチで設定変更を提案する
3. 影響範囲を確認する: `rubocop --only NewRule`
4. 自動修正を適用する: `rubocop -a --only NewRule`
5. PR でレビューする
6. マージ後、全員が最新の設定を取得する

**根拠**: 明確なプロセスにより、チーム全体での設定変更がスムーズになる

### エディタ統合の推奨

チーム全体でエディタ統合を有効化し、リアルタイムで違反を検出する：

- **VS Code**: `ruby-rubocop` 拡張機能をインストールする
- **RubyMine**: デフォルトで RuboCop サポートが有効になっている
- **Vim/NeoVim**: `syntastic` や `ale` プラグインを使用する

**根拠**: エディタ統合により、コミット前に問題を発見し、CI の失敗を防ぐ

## パフォーマンス設定

### キャッシュの活用

```yaml
AllCops:
  UseCache: true
  CacheRootDirectory: tmp/rubocop_cache
```

**根拠**: キャッシュにより、大規模プロジェクトでの実行速度が大幅に向上する

### 並列実行

```bash
# 並列実行でパフォーマンス向上
rubocop --parallel
```

**根拠**: 複数の CPU コアを活用し、実行時間を短縮する

### 不要なファイルの除外

```yaml
AllCops:
  Exclude:
    - 'vendor/**/*'
    - 'db/schema.rb'
    - 'node_modules/**/*'
    - 'bin/**/*'
```

**根拠**: 外部ライブラリや自動生成ファイルをスキップし、実行時間を削減する

## アップグレード戦略

### バージョンアップの推奨手順

1. 現在のバージョンを確認する: `bundle exec rubocop --version`
2. Gemfile を更新する
3. `bundle update rubocop` を実行する
4. 新しい違反を確認する: `bundle exec rubocop`
5. 新しい違反を TODO ファイルに追加する: `rubocop --auto-gen-config`
6. 段階的に修正する

**根拠**: 段階的なアップグレードにより、チームへの影響を最小化する

### NewCops の管理

```yaml
AllCops:
  # 新しい cops を自動的に有効化（推奨）
  NewCops: enable

  # または無効化（現在の動作を維持）
  NewCops: disable
```

**根拠**: `NewCops: enable` により、新しいベストプラクティスを早期に採用できる

## 参考資料

### 公式ドキュメント

- [RuboCop 公式サイト](https://docs.rubocop.org/rubocop/)
- [Ruby Style Guide](https://rubystyle.guide/)
- [Rails Style Guide](https://rails.rubystyle.guide/)
- [RSpec Style Guide](https://rspec.rubystyle.guide/)
- [RuboCop GitHub Repository](https://github.com/rubocop/rubocop)

### 拡張機能

- [RuboCop Performance](https://docs.rubocop.org/rubocop-performance/)
- [RuboCop Rails](https://docs.rubocop.org/rubocop-rails/)
- [RuboCop RSpec](https://docs.rubocop.org/rubocop-rspec/)
- [RuboCop Minitest](https://github.com/rubocop/rubocop-minitest)

### コミュニティリソース

- [Shopify Ruby Style Guide](https://ruby-style-guide.shopify.dev/)
- [Standard Ruby](https://github.com/standardrb/standard)
- [Relaxed Ruby Style](https://relaxed.ruby.style/)
- [RuboCop GitHub Action](https://github.com/marketplace/actions/rubocop-linter-action)

### ソースファイル

- [Official Best Practices](../../docs/best_practice/RuboCop/official.md)
- [Community Best Practices](../../docs/best_practice/RuboCop/community.md)
