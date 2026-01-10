## 概要

RuboCopは、Rubyの静的コード解析ツール（リンター）およびコードフォーマッターです。コミュニティの[Ruby Style Guide](https://rubystyle.guide/)に基づいた多数のガイドラインを自動的にチェックし、コード品質を向上させます。

### 主要機能

- すべての主要なRuby実装で動作
- 多くのコード違反の自動修正機能
- 強力なコードフォーマット機能
- 複数の結果フォーマッター（インタラクティブ使用およびツール連携用）
- コードベースの異なる部分に異なる設定を適用可能
- 特定のファイルやファイルの一部に対してのみcopsを無効化可能
- 極めて柔軟な設定により、あらゆるスタイルや好みに対応
- カスタムcopsとフォーマッターによる拡張が容易
- 多数の既製拡張機能（rubocop-rails、rubocop-rspec、rubocop-performanceなど）
- 広範なエディタ/IDE対応

### 哲学

RuboCopは極度の設定可能性と柔軟性を哲学としています。Rubyコミュニティには多様なスタイルと好みが存在するため、RuboCopはあらゆるスタイルをサポートします。コミュニティのスタイルガイドに従うことを推奨しつつも、ユーザーの好みに応じて自由にカスタマイズできます。

## 推奨設定

### 基本設定

プロジェクトのルートディレクトリに`.rubocop.yml`設定ファイルを作成することが最も一般的な方法です。

```yaml
# 基本的な設定例
AllCops:
  # 新しいバージョンで追加されるcopsを有効化
  NewCops: enable
  # Rubyバージョンを指定
  TargetRubyVersion: 3.0
  # 除外するディレクトリやファイル
  Exclude:
    - 'vendor/**/*'
    - 'db/schema.rb'
    - 'node_modules/**/*'

# 行の長さを調整
Layout/LineLength:
  Max: 120

# メソッド名のスタイル
Naming/MethodName:
  Enabled: true
```

### 設定ファイルの配置優先順位

RuboCopは以下の順序で設定ファイルを検索します：

1. 検査対象ファイルのディレクトリから親ディレクトリへ遡って`.rubocop.yml`を探索
2. プロジェクトルートの`.config/.rubocop.yml`または`.config/rubocop/config.yml`
3. ユーザーのホームディレクトリ`~/.rubocop.yml`
4. XDG Base Directory仕様に基づく`$XDG_CONFIG_HOME/rubocop/config.yml`（デフォルトは`~/.config/rubocop/config.yml`）

### 設定の継承

他の設定ファイルから継承することができます：

```yaml
inherit_from: ../.rubocop.yml

# または複数のファイルから継承
inherit_from:
  - .rubocop_base.yml
  - .rubocop_rails.yml
```

### Active Support拡張の有効化

Active Supportのメソッド（`Hash#except`など）をチェックする場合：

```yaml
AllCops:
  ActiveSupportExtensionsEnabled: true
```

### frozen string literalsのサポート

Ruby 3.4以降に向けて、すべての文字列リテラルが凍結されることを前提とする場合：

```yaml
AllCops:
  StringLiteralsFrozenByDefault: true
```

## 基本的な使用方法

### コードスタイルチェッカーとして

```bash
# カレントディレクトリの全Rubyファイルをチェック
$ rubocop

# 特定のファイルやディレクトリをチェック
$ rubocop app spec lib/something.rb
```

### 安全な自動修正

```bash
# 安全な自動修正のみを実行
$ rubocop -a
# または
$ rubocop --autocorrect
```

### すべての自動修正（安全・非安全含む）

```bash
# すべての自動修正を実行
$ rubocop -A
# または
$ rubocop --autocorrect-all
```

**重要**: 自動修正機能を使用した後は、必ずテストスイートを実行してください。一部の自動修正はコードのセマンティクスをわずかに変更する可能性があります。

### Lintチェックのみ実行

```bash
$ rubocop -l
# または
$ rubocop --lint
```

### フォーマッターとして（レイアウト違反のみ修正）

```bash
$ rubocop -x
# または
$ rubocop --fix-layout
```

### 設定ファイルの自動生成

```bash
# 現在のコードベースに基づいてTODOリストとなる設定ファイルを生成
$ rubocop --auto-gen-config
```

このコマンドは`.rubocop_todo.yml`を生成し、既存の違反を一時的に無効化します。これにより、段階的にコード品質を改善できます。

### 特定のcopsのみを実行

```bash
# 複数のcopsをカンマ区切りで指定（スペースなし）
$ rubocop --only Rails/Blank,Layout/HeredocIndentation,Naming/FileName
```

## パフォーマンス最適化

### キャッシュの活用

RuboCopはデフォルトでキャッシュを有効にし、大規模プロジェクトの検査を高速化します。

```yaml
AllCops:
  # キャッシュを有効化（デフォルト: true）
  UseCache: true
  # キャッシュのルートディレクトリを指定
  CacheRootDirectory: tmp/rubocop_cache
```

キャッシュの保存場所（優先度順）：

1. `--cache-root`コマンドラインオプション
2. `$RUBOCOP_CACHE_ROOT`環境変数
3. `AllCops: CacheRootDirectory`設定パラメータ
4. デフォルト: `$XDG_CACHE_HOME/$UID/rubocop_cache`または`$HOME/.cache/rubocop_cache/`

### プロファイリング

パフォーマンスボトルネックを特定するためのプロファイリング：

```bash
$ rubocop --profile
```

## Cops（チェックルール）

RuboCopでは、様々なコードチェックを「cops」と呼びます。各copは特定の違反を検出する責任を持ちます。

### Copsの部門

#### Style

スタイルコードの一貫性をチェックします。多くはRuby Style Guideに基づいています。

```yaml
Style/StringLiterals:
  EnforcedStyle: double_quotes
```

#### Layout

インデント、配置、空白の一貫性をチェックします。

```yaml
Layout/IndentationStyle:
  EnforcedStyle: spaces
  IndentationWidth: 2
```

#### Lint

コードの曖昧さや潜在的なエラーをチェックします。RubyのMRI lint（`ruby -wc`）のすべてのチェックをポータブルに実装し、さらに多くの追加チェックを提供します。

**重要**: Lint copsを無効化することは一般的に推奨されません。

```yaml
Lint/ShadowingOuterLocalVariable:
  Enabled: true
```

#### Metrics

クラスの長さ、メソッドの長さなど、測定可能なソースコードの特性をチェックします。

```yaml
Metrics/MethodLength:
  Max: 30
  CountComments: false
```

#### Naming

メソッド名、定数名、ファイル名などの命名規則をチェックします。

```yaml
Naming/FileName:
  Enabled: true
  ExpectMatchingDefinition: false
```

#### Security

セキュリティ上の問題と関連するメソッド呼び出しや構造をチェックします。

```yaml
Security/Eval:
  Enabled: true
```

#### Bundler

Gemfileなど、Bundlerファイルのスタイルと悪い慣習をチェックします。

#### Gemspec

gemspecファイルのスタイルと悪い慣習をチェックします。

### Copsの有効化・無効化

特定のcopを無効化：

```yaml
Style/FrozenStringLiteralComment:
  Enabled: false
```

特定のファイルやディレクトリで無効化：

```yaml
Style/Documentation:
  Exclude:
    - 'spec/**/*'
    - 'test/**/*'
```

コード内で一時的に無効化：

```ruby
# rubocop:disable Style/FrozenStringLiteralComment
def some_method
  # ...
end
# rubocop:enable Style/FrozenStringLiteralComment
```

## セキュリティ

### Security Cops

Security部門のcopsは、既知のセキュリティ問題に関連するメソッド呼び出しや構造をチェックします。

```yaml
Security/Eval:
  # eval()の使用を検出
  Enabled: true

Security/YAMLLoad:
  # YAML.load()の安全でない使用を検出
  Enabled: true

Security/Open:
  # Kernel#openの安全でない使用を検出
  Enabled: true
```

### 推奨事項

1. **Security copsは常に有効に保つ**: セキュリティ関連のチェックを無効化しないでください
2. **定期的なアップデート**: RuboCopを最新バージョンに保ち、新しいセキュリティチェックを活用してください
3. **拡張機能の活用**: プロジェクトに応じて`rubocop-rails`などのセキュリティ拡張を追加してください

## 拡張機能とプラグイン

### 公式拡張

RuboCopのコアチームがメンテナンスする拡張機能：

- **rubocop-performance**: パフォーマンス最適化の解析
- **rubocop-rails**: Rails固有の解析
- **rubocop-rspec**: RSpec固有の解析
- **rubocop-minitest**: Minitest固有の解析
- **rubocop-rake**: Rake固有の解析
- **rubocop-sequel**: Sequel gemのコードスタイルチェック
- **rubocop-thread_safety**: スレッドセーフティ解析
- **rubocop-capybara**: Capybara固有の解析
- **rubocop-factory_bot**: factory_bot固有の解析
- **rubocop-rspec_rails**: RSpec Rails固有の解析

### 拡張機能のインストールと設定

Gemfileに追加：

```ruby
group :development do
  gem 'rubocop', require: false
  gem 'rubocop-rails', require: false
  gem 'rubocop-rspec', require: false
  gem 'rubocop-performance', require: false
end
```

`.rubocop.yml`で有効化：

```yaml
require:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance
```

### 拡張機能の推奨機能

RuboCopは、プロジェクトで使用しているgemに基づいて、関連する拡張機能を自動的に提案します。この機能を無効化するには：

```yaml
AllCops:
  SuggestExtensions: false

  # または特定の拡張のみ提案を無効化
  SuggestExtensions:
    rubocop-rake: false
```

## エディタ・IDE統合

### Visual Studio Code

[Ruby拡張](https://marketplace.visualstudio.com/items?itemName=rebornix.Ruby)を使用してRuboCopを統合できます。

### RubyMine / IntelliJ IDEA

2017.1リリース以降、RuboCopサポートが標準で利用可能です。

### その他のエディタ

- **Emacs**: rubocop.el、flycheck
- **Vim**: syntastic、neomake、ale、vim-rubocop
- **Sublime Text**: SublimeLinter-rubocop、Sublime RuboCop
- **Atom**: linter-rubocop
- **Helix**: Solargraph（LSP）およびフォーマッター設定

## CI/CD統合

### Git pre-commitフック

#### overcommitの使用

`.overcommit.yml`:

```yaml
PreCommit:
  RuboCop:
    enabled: true
```

#### pre-commitフレームワークの使用

`.pre-commit-config.yaml`:

```yaml
- repo: https://github.com/rubocop/rubocop
  rev: v1.81.1
  hooks:
    - id: rubocop
      additional_dependencies:
        - rubocop-rails
        - rubocop-rspec
```

### 自動コードレビューサービス

RuboCopを活用する主要なSaaSサービス：

- **Code Climate**: テストカバレッジ、複雑度、重複、セキュリティなどの自動レビュー
- **Codacy**: スタイル、セキュリティ、重複、複雑度のチェック（オープンソースは無料）
- **Hound**: GitHub pull requestでのスタイル違反のコメント（オープンソース）
- **Pronto**: GitHub、GitLab、Bitbucketと統合する自動コードレビュー
- **ReviewDog**: GitHub Actionsとの優れた統合を持つProntoの代替

## よくある落とし穴

### 自動修正の注意点

1. **すべての違反が自動修正できるわけではない**: 一部の違反は手動での修正が必要です
2. **非安全な自動修正**: `-A`（`--autocorrect-all`）は、コードのセマンティクスをわずかに変更する可能性のある修正も実行します
3. **テストの実行を忘れない**: 自動修正後は必ずテストスイートを実行してください

```bash
# 安全な修正のみ（推奨）
$ rubocop -a

# すべての修正（注意が必要）
$ rubocop -A
```

### 設定ファイルの探索

- RuboCopは検査対象ファイルのディレクトリから親ディレクトリへ遡って設定ファイルを探索します
- 特定の設定ファイルを使用する場合は`--config`フラグを使用してください

```bash
$ rubocop --config config/custom_rubocop.yml
```

### Lint copsの無効化

**推奨されません**: Lint copsは潜在的なバグを検出するため、無効化は避けるべきです。

### キャッシュの無効化が必要な場合

外部依存関係を変更した場合、キャッシュの無効化が必要になることがあります：

```bash
$ rubocop --cache false
```

または、カスタムcopで外部依存関係のチェックサムを実装：

```ruby
def external_dependency_checksum
  # カスタムチェックサムロジック
end
```

## バージョニングとアップグレード

### セマンティックバージョニング

RuboCopはセマンティックバージョニングに従っていますが、以下の点に注意：

- **新しいcops**: マイナーバージョンで追加される可能性があります（デフォルトでは無効）
- **pending cops**: 次のメジャーバージョンでデフォルト有効になるcopsのプレビュー

### pending copsの管理

```yaml
AllCops:
  # pending copsを有効化（次のメジャーバージョンの挙動を先取り）
  NewCops: enable

  # または無効化（現在の挙動を維持）
  NewCops: disable
```

### アップグレード時の推奨手順

1. 変更履歴（changelog）を確認
2. `--auto-gen-config`で新しい違反を特定
3. 段階的に違反を修正
4. テストスイートを実行して回帰を確認

## 参考リンク

### 公式ドキュメント

- [RuboCop公式サイト](https://docs.rubocop.org/rubocop/)
- [インストールガイド](https://docs.rubocop.org/rubocop/installation.html)
- [基本的な使い方](https://docs.rubocop.org/rubocop/usage/basic_usage.html)
- [設定ガイド](https://docs.rubocop.org/rubocop/configuration.html)
- [Copsドキュメント](https://docs.rubocop.org/rubocop/cops.html)

### コミュニティスタイルガイド

- [Ruby Style Guide](https://rubystyle.guide/)
- [Rails Style Guide](https://rails.rubystyle.guide/)
- [RSpec Style Guide](https://rspec.rubystyle.guide/)
- [Minitest Style Guide](https://minitest.rubystyle.guide/)

### 拡張機能

- [公式拡張一覧](https://docs.rubocop.org/rubocop/extensions.html)
- [RuboCop Performance](https://docs.rubocop.org/rubocop-performance/)
- [RuboCop Rails](https://docs.rubocop.org/rubocop-rails/)
- [RuboCop RSpec](https://docs.rubocop.org/rubocop-rspec/)

### その他

- [GitHubリポジトリ](https://github.com/rubocop/rubocop)
- [変更履歴](https://docs.rubocop.org/rubocop/changelog.html)
- [貢献ガイド](https://docs.rubocop.org/rubocop/contributing.html)
