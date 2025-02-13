---
title: "WORK-①: パッケージの取得処理を作ってみよう(install)"
---

ここからハンズオンの本編です！

本物のComposerに比べると遥かに小規模で貧相ですが、自分だけの「ちっちゃくて可愛い」「おもちゃのような」オリジナルツール **『Tinyposer』** を作り上げていきましょう！

## ワークの概要とゴール

最初は _install_ フェーズの処理を実装します。

課題用レポジトリの `work` ディレクトリには、予め`compoiser.lock`が用意されています。本物のComposerを利用して作成されたものです。このファイルを読み取り、`vendor`ディレクトリにパッケージファイルを配置するところまでを扱います。

![](/images/2_work1_0_intro/2/intro/tree.png)
*ワークの終了後には、vendorディレクトリが出来上がっている状態になります*

## ワーク1で抑えたいComposerの仕組み

Composerのエッセンスについて、このワークに取り組むことで学べるポイントは次のとおりです。

1. `composer.lock` に記されているパッケージ情報の利用方法
2. パッケージのコンテンツ(ソースコード)は、どのように配布されいてるか/ComposerがGitHubとどう関わっているか

## 本物との差分の例

本物のComposerと異なる点(機能が欠落している要素)は、次のとおりです。

* GitHub以外のホスティングに対応していない
* `package-type:composer` 以外のレポジトリに対応していない
* Zip以外のアーカイブ形式に対応していない
* インストール時のシステム要件のチェックに対応していない
* `--no-install` `--prefer-install`などの各種オプションや、それに対応する機能
* ⚠️**アーカイブファイルの扱いに関するセキュリティ事項の考慮**

## 用語の整理

ハンズオン中に出てくる用語について、以下のように定義します。
一部、Composer本来の用語とは(矛盾しない範囲で)異なりますので注意してください。

* レポジトリ: パッケージのメタ情報の一覧を管理しているシステム。今回の場合は、Packagist
* レポジトリのメタ情報: レポジトリ自体の機能や公開情報を扱う情報
* パッケージ: インストール対象となるライブラリ
* パッケージの本体・パッケージのコンテンツ: ソースコードを含む、パッケージの実体
* パッケージのメタ情報: レポジトリ上に管理されているパッケージ情報。各バージョン一式の情報をまとめたもの
* パッケージ定義(ファイル): 各パッケージの特定のバージョンに対応する、完結した定義情報。`composer.json`。
* プロジェクト: Composerによってパッケージを追加する、自分自身のパッケージ。(root package)
