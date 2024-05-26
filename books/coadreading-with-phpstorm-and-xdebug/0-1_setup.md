---
title: "準備: 利用する環境の準備"
free: true
---

本書で利用する環境とその構築について、説明をします。

## ソースコードの取得

PhpStormのプロジェクト設定を含めて、ソースコードがGitHubに展開されています。  
まずは、これをcloneしてきてください。  
(もちろん、PhpStormのメニューを利用しても良いですね！[^pj-from-vcs])

https://github.com/o0h/xdebug-phpstorm-handson

[^pj-from-vcs]: GitHubからプロジェクトを取得・開始する方法は、こちらの動画に詳しいです。 https://www.youtube.com/watch?v=aBVOAnygcZw

## PHP実行環境の確認

レポジトリに含まれる情報によって、PHPの実行環境に関する設定が完了しているはずです。  
具体的に、どのような内容となっているのかを確認しましょう。

設定(`Settings`) > `PHP` を開くと、「プロジェクトで利用するPHPのバージョン」「PhpStormがPHPを実行する際に利用する環境」が表示されます。

- `PHP Language Level`: `8.3`
  - これは`composer.json` の `require` 句の指示に同期されています
- `CLI Interpreter`: `app`
  - Dockerコンテナを利用すること、及びその際の設定情報(名)についての指示が、 `.idea/php.xml` 内で記述されています
    - (`component[name="PhpInterpreters"] > interpreters > interpreter` 要素)
  - この `app` は、 `.runtime/compose.yaml` で定義されているサービス名です

![](/images/0-1_setup/php-interpreter-and-level.png)

`CLI Interpreter` の選択ボックスの右隣にあるメニュー(「・・・」ボタン)をクリックすと、コンテナでのPHP情報を詳しく見ることが出来ます。  
実行されるPHPのバージョン情報や、Xdebugの情報がここで確認できます。

![](/images/0-1_setup/phpenv.png)

## PHPの疎通確認と、Composerパッケージの取得

取得してきたコードには、PhpStormのプロジェクト設定ファイルである [`.idea`ディレクトリ](https://github.com/o0h/xdebug-phpstorm-handson/tree/main/.idea) を梱包しています。  
これによって、基本的な設定が共有されています。

例えば、PHPの実行情報は「設定済み」な項目の1つです。  
それを確かめるのと同時に、ハンズオンを進めるのに不可欠となる準備を進めるため、Composerで管理されているパッケージ群を取得しましょう。

`composer.json` をエディタで開いて
![](/images/0-1_setup/open-composer.png)

画面上部に、基本的なComposerコマンドを利用するメニューが表示されているので、 `install` をクリックします
![](/images/0-1_setup/composer-install.png)

問題なく設定が適用されていれば、依存パッケージが取得されていく様子が表示されるはずです。
![](/images/0-1_setup/composer-install-execution.png)

もし、「Composerコマンドの実行環境の指示が必要」といった旨の表示がされる場合は、設定(`Settings`) > `PHP` > `Composer` メニューを開いて、

- `Execution` が `Remote Interpreter` を指定していること
- `CLI Interpreter` が `app` を指定していること

を確認してください。

![](/images/0-1_setup/setting-composer-interpreter.png)
