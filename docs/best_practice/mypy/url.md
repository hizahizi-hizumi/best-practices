# mypy ベストプラクティス - 参考リンク集

mypyのベストプラクティスをまとめるために収集した公式ドキュメントのURLリストです。

## 目次

- [入門・導入](#入門導入)
- [設定ファイル](#設定ファイル)
- [型アノテーション](#型アノテーション)
- [よくある問題と解決方法](#よくある問題と解決方法)
- [エラーコード](#エラーコード)
- [既存コードベースへの導入](#既存コードベースへの導入)
- [パフォーマンス最適化](#パフォーマンス最適化)
- [高度な機能](#高度な機能)
- [その他の参考資料](#その他の参考資料)

---

## 入門・導入

### Getting started
- **URL**: https://mypy.readthedocs.io/en/stable/getting_started.html
- **説明**: mypy の基本的な使い方とインストール方法。静的型付けと動的型付けの違い、基本的な型アノテーション、strict モードの概要などを解説。
- **重要トピック**:
  - mypy のインストールと実行方法
  - 動的型付けと静的型付けの違い
  - Strict mode と設定オプション
  - 複雑な型の使い方
  - ライブラリの型情報の扱い方

### Type hints cheat sheet
- **URL**: https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html
- **説明**: Python 3 の型ヒントのクイックリファレンス。変数、関数、クラスの型アノテーションの書き方をまとめたチートシート。
- **重要トピック**:
  - 変数の型アノテーション
  - 便利な組み込み型
  - 関数とクラスの型ヒント
  - デコレータと非同期処理

---

## 設定ファイル

### The mypy configuration file
- **URL**: https://mypy.readthedocs.io/en/stable/config_file.html
- **説明**: mypy.ini、pyproject.toml、setup.cfg での設定ファイルの書き方。グローバルオプションとモジュール単位のオプション設定を詳細に解説。
- **重要トピック**:
  - 設定ファイルの形式（mypy.ini、pyproject.toml）
  - グローバルオプションとモジュール別オプション
  - 厳密性フラグの設定
  - インポート検出の設定
  - エラーメッセージのカスタマイズ
  - サンプル設定例

### The mypy command line
- **URL**: https://mypy.readthedocs.io/en/stable/command_line.html
- **説明**: mypy のコマンドラインオプションの完全なリファレンス。チェック対象の指定方法、厳密性の調整、エラーレポートの設定など。
- **重要トピック**:
  - チェック対象の指定方法
  - 動的型付けの禁止オプション
  - 型なし定義と呼び出しの設定
  - None と Optional の扱い
  - 警告の設定
  - 厳密性フラグ
  - CI/CD での使用方法

### Inline configuration
- **URL**: https://mypy.readthedocs.io/en/stable/inline_config.html
- **説明**: ソースコード内で型チェックの設定を行う方法。`# type: ignore` コメントや mypy 設定コメントの使い方。

---

## 型アノテーション

### Built-in types
- **URL**: https://mypy.readthedocs.io/en/stable/builtin_types.html
- **説明**: Python の組み込み型の扱い方。int、str、list、dict などの基本型と Any 型の説明。

### Type inference and type annotations
- **URL**: https://mypy.readthedocs.io/en/stable/type_inference_and_annotations.html
- **説明**: 型推論と明示的な型アノテーション。mypy がどのように型を推論するか、いつ明示的な型アノテーションが必要かを解説。

### Kinds of types
- **URL**: https://mypy.readthedocs.io/en/stable/kinds_of_types.html
- **説明**: 様々な型の種類（クラス型、Any 型、タプル型、Callable 型、Union 型、Optional 型、型エイリアスなど）の詳細な説明。

### Class basics
- **URL**: https://mypy.readthedocs.io/en/stable/class_basics.html
- **説明**: クラスの型チェックの基本。インスタンス変数、クラス変数、__init__ メソッドの型アノテーション方法。

### Annotation issues at runtime
- **URL**: https://mypy.readthedocs.io/en/stable/runtime_troubles.html
- **説明**: 型アノテーションの実行時の問題と解決方法。前方参照、循環インポート、PEP 563 の `from __future__ import annotations` の使い方など。

### Protocols and structural subtyping
- **URL**: https://mypy.readthedocs.io/en/stable/protocols.html
- **説明**: Protocol を使った構造的サブタイピング。ダックタイピングを型安全に実現する方法。

### Dynamically typed code
- **URL**: https://mypy.readthedocs.io/en/stable/dynamic_typing.html
- **説明**: Any 型を使った動的型付けコードの扱い方。Any と object の違い。

### Type narrowing
- **URL**: https://mypy.readthedocs.io/en/stable/type_narrowing.html
- **説明**: 型の絞り込み（Type narrowing）。isinstance チェックやユーザー定義 Type Guard の使い方。

### Generics
- **URL**: https://mypy.readthedocs.io/en/stable/generics.html
- **説明**: ジェネリック型の定義と使用方法。TypeVar、Generic、共変性・反変性の説明。

### More types
- **URL**: https://mypy.readthedocs.io/en/stable/more_types.html
- **説明**: その他の高度な型（NoReturn、NewType、関数オーバーロード、async/await の型付けなど）。

### TypedDict
- **URL**: https://mypy.readthedocs.io/en/stable/typed_dict.html
- **説明**: TypedDict を使った辞書の型付け。必須・オプション項目の指定、読み取り専用項目の定義。

### Final names, methods and classes
- **URL**: https://mypy.readthedocs.io/en/stable/final_attrs.html
- **説明**: Final デコレータを使った変更不可能な定義の作成方法。

### Stub files
- **URL**: https://mypy.readthedocs.io/en/stable/stubs.html
- **説明**: スタブファイル（.pyi）の作成と使用方法。型情報のないライブラリに型を追加する方法。

---

## よくある問題と解決方法

### Common issues and solutions
- **URL**: https://mypy.readthedocs.io/en/stable/common_issues.html
- **説明**: mypy 使用時によく遭遇する問題とその解決方法。型エラーの抑制、空のコレクション、型の再定義、共変性・反変性の問題など。
- **重要トピック**:
  - 明らかに間違ったコードでエラーが報告されない場合
  - エラーメッセージの抑制方法（type: ignore）
  - ファイル全体のエラー無視
  - 実行時のコード問題
  - mypy の実行速度の改善
  - 空のコレクションの型指定
  - 型の再定義
  - 共変性と反変性
  - 複雑な型テスト
  - Python バージョンとプラットフォームチェック

### Missing imports
- **URL**: https://mypy.readthedocs.io/en/stable/running_mypy.html#missing-imports
- **説明**: インポートが見つからない問題の解決方法。スタブのインストール、ignore_missing_imports の使い方。

---

## エラーコード

### Error codes
- **URL**: https://mypy.readthedocs.io/en/stable/error_codes.html
- **説明**: mypy のエラーコードシステムの概要。エラーコードを使った細かいエラー制御方法。

### Error codes enabled by default
- **URL**: https://mypy.readthedocs.io/en/stable/error_code_list.html
- **説明**: デフォルトで有効になっているエラーコードの一覧と説明。各エラーコードの意味と発生する状況を詳しく解説。

### Error codes for optional checks
- **URL**: https://mypy.readthedocs.io/en/stable/error_code_list2.html
- **説明**: オプションのエラーコード一覧。より厳密な型チェックを行いたい場合に有効にできるエラーコード。

---

## 既存コードベースへの導入

### Using mypy with an existing codebase
- **URL**: https://mypy.readthedocs.io/en/stable/existing_code.html
- **説明**: 既存の大規模コードベースに mypy を導入する方法。段階的な導入戦略、CI/CD への統合、strictモードへの移行方法。
- **重要トピック**:
  - 小さく始める（Start small）
  - mypy を一貫して実行し、リグレッションを防ぐ
  - CI/CD での mypy の実行
  - 特定モジュールのエラーを無視する
  - インポート関連のエラー修正
  - 広く使われているモジュールの優先的なアノテーション
  - コードを書きながらアノテーションを追加
  - レガシーコードの自動アノテーション（MonkeyType、autotyping、PyAnnotate）
  - より厳密なオプションの導入
  - strict モードへの移行戦略

### Running mypy and managing imports
- **URL**: https://mypy.readthedocs.io/en/stable/running_mypy.html
- **説明**: mypy の実行方法とインポートの管理。チェック対象の指定、インポートの追跡方法、不足しているインポートの処理。

---

## パフォーマンス最適化

### Mypy daemon (mypy server)
- **URL**: https://mypy.readthedocs.io/en/stable/mypy_daemon.html
- **説明**: mypy デーモン（dmypy）を使った高速な増分型チェック。大規模プロジェクトでの型チェック速度を劇的に改善する方法。

### Using a remote cache to speed up mypy runs
- **URL**: https://mypy.readthedocs.io/en/stable/additional_features.html#using-a-remote-cache-to-speed-up-mypy-runs
- **説明**: リモートキャッシュを使った mypy 実行の高速化。チーム開発や CI/CD での型チェック速度の改善。

---

## 高度な機能

### Extending and integrating mypy
- **URL**: https://mypy.readthedocs.io/en/stable/extending_mypy.html
- **説明**: mypy のプラグインシステム。カスタムプラグインを作成して mypy の機能を拡張する方法。

### Using installed packages
- **URL**: https://mypy.readthedocs.io/en/stable/installed_packages.html
- **説明**: PEP 561 準拠のパッケージの使用方法。型情報を含むパッケージの作成と使用。

### Automatic stub generation (stubgen)
- **URL**: https://mypy.readthedocs.io/en/stable/stubgen.html
- **説明**: stubgen を使ったスタブファイルの自動生成。既存のコードから .pyi ファイルを生成する方法。

### Automatic stub testing (stubtest)
- **URL**: https://mypy.readthedocs.io/en/stable/stubtest.html
- **説明**: stubtest を使ったスタブファイルのテスト。スタブファイルが実際の実装と一致しているか検証する方法。

### Additional features
- **URL**: https://mypy.readthedocs.io/en/stable/additional_features.html
- **説明**: その他の機能（Dataclasses のサポート、attrs パッケージの扱い、リモートキャッシュなど）。

---

## その他の参考資料

### Frequently Asked Questions
- **URL**: https://mypy.readthedocs.io/en/stable/faq.html
- **説明**: mypy に関するよくある質問と回答。動的型付けと静的型付けの併用、プロジェクトでの利用、ダックタイピングとの関係など。

### Supported Python features
- **URL**: https://mypy.readthedocs.io/en/stable/supported_python_features.html
- **説明**: mypy がサポートしている Python の機能と制限事項。

### Mypy Release Notes
- **URL**: https://mypy.readthedocs.io/en/stable/changelog.html
- **説明**: mypy のバージョン履歴と変更点。各バージョンでの新機能、バグ修正、破壊的変更の情報。

### GitHub Repository
- **URL**: https://github.com/python/mypy
- **説明**: mypy の公式 GitHub リポジトリ。ソースコード、Issue、Pull Request、開発への貢献方法。

### Official Website
- **URL**: https://mypy-lang.org/
- **説明**: mypy の公式ウェブサイト。プロジェクトの概要と最新情報。

### PEP 484 – Type Hints
- **URL**: https://peps.python.org/pep-0484/
- **説明**: Python の型ヒントに関する PEP（Python Enhancement Proposal）。型ヒントの標準仕様。

### PEP 561 – Distributing and Packaging Type Information
- **URL**: https://peps.python.org/pep-0561/
- **説明**: 型情報の配布とパッケージ化に関する PEP。py.typed マーカーやスタブパッケージの作成方法。

---

## カテゴリ別重要度

### 最優先（必読）
- Getting started
- Using mypy with an existing codebase
- The mypy configuration file
- Common issues and solutions
- Type hints cheat sheet

### 高優先度
- The mypy command line
- Built-in types
- Type inference and type annotations
- Error codes enabled by default
- Running mypy and managing imports

### 中優先度
- Kinds of types
- Class basics
- Protocols and structural subtyping
- Mypy daemon (mypy server)
- Using installed packages

### 必要に応じて参照
- その他すべてのドキュメント

---

## ベストプラクティス作成のための推奨読書順序

1. **Getting started** - mypy の基本を理解
2. **Type hints cheat sheet** - 型ヒントの書き方を習得
3. **Using mypy with an existing codebase** - 導入戦略を学ぶ
4. **The mypy configuration file** - 設定方法を理解
5. **Common issues and solutions** - よくある問題への対処法を学ぶ
6. **Error codes enabled by default** - エラーメッセージを理解
7. **The mypy command line** - コマンドラインオプションを把握
8. その他必要に応じて該当するトピックを参照

---

## 更新履歴

- 2026-01-03: 初版作成（mypy 1.19.1 ドキュメントを基に作成）
