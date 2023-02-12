---
title: "1章 利用する環境の準備"
free: true
---

# PHPアプリケーションの開発環境を用意する

それでは、いよいよ話を始めましょう。

まずは、本書で説明している内容を手元で簡単に試してみるためにも、アプリケーションを開発し実行するための環境を構築します。
・・・とは言っても、何も複雑なことはありません。Docker/Docker Composeを利用した開発用環境を用意してあるので、そのまま利用することができます。

下記のアーカイブファイルをダウンロードして、展開してください。

https://github.com/o0h/o0h-zenn-articles-public/raw/main/books/re-introduction-php-waf-development/sample/part1.zip

展開したファイルのざっくりとした利用方法は、README.mdに記載されいている通りです。

https://github.com/o0h/o0h-zenn-articles-public/blob/main/books/re-introduction-php-waf-development/sample/part1/README.md

もう少々だけ詳しく見ていきましょう。

# 全体的な構成

ディレクトリは以下のような構成になっています。

```
.
├── .gitignore
├── .runtime
│   ├── app
│   ├── database
│   ├── docker-compose.yml
│   └── nginx
├── Makefile
├── README.md
├── app
│   └── webroot
├── composer.json
└── composer.lock
```

アプリケーションの実行環境(Docker周りのファイル)をまとめた `.runtime`ディレクトリ、アプリケーションの中身が含まれる`app`ディレクトリ、PHPプロジェクトを定義する`comopser.json` `composer.lock`、その他にこの環境を利用する手助けとしての`README.md`と`Makefile`が含まれています。

ただし、`app`ディレクトリは今の所は特に言及すべき内容はありません。`phpinfo()`を実行するだけになっています。

# 起動と動作確認

`docker-compose.yml`でサービスを構成済みのため、これを起動すればローカル環境を立ち上げることが出来ます。
そうは言えども、設定ファイルの指定などを指定するのは手間になるので、タスクランナーとして``Makefile`を用意しています。

アーカイブファイルを展開したディレクトリにて、`make`を実行してください。Dockerイメージのビルドと、コンテナの作成・起動が行われます。

`make ps`を実行することで、起動しているコンテナを確認することができます。

```shell
$ make ps
COMPOSE_PROJECT_NAME=reintroduction-php-fw-app docker compose -f .runtime/docker-compose.yml ps
NAME                          IMAGE                           COMMAND                  SERVICE             CREATED             STATUS              PORTS
reintroduction-php-fw-app     reintroduction-php-fw-app-app   "docker-php-entrypoi…"   app                 39 seconds ago      Up 37 seconds       9000/tcp
reintroduction-php-fw-db      mysql:8                         "docker-entrypoint.s…"   db                  39 seconds ago      Up 37 seconds       33060/tcp, 0.0.0.0:3341->3306/tcp
reintroduction-php-fw-nginx   nginx:alpine                    "/docker-entrypoint.…"   nginx               39 seconds ago      Up 37 seconds       0.0.0.0:8080->80/tcp
```

上記のように、「アプリケーションサーバー」「Webサーバー(Nginx)」「データベースサーバー(MySQL)」が起動していれば正常です。

続いて、ブラウザなどから`http://localhost:8080/` にアクセスしてください。ここでは`app/webroot`ディレクトリをドキュメントルートとしてPHPスクリプトが実行されます。現在は、`phpinfo()`を実行するための`index.php`が設置されているだけなので、PHP の設定情報が出力されていることを確認できればOKです。次の章から、`app`ディレクトリにCakePHPアプリケーションを配置していくことになります。

![phpinfo](/images/image-20230212035724516.png)

# ごく簡単な内容の紹介

立ち上がったサービスは、ごくシンプルなNginx+PHP-FPM＋MySQLで構成されています。PHP利用者にとっては、一般的な構成の1つと言えるでしょう。

具体的な内容は、`docker-compose.yml`を参照することで確認ができます。

https://github.com/o0h/o0h-zenn-articles-public/blob/main/books/re-introduction-php-waf-development/sample/part1/.runtime/docker-compose.yml

## 利用しているPHPについて

PHPについては、執筆時点の最新バージョンである`8.2`を利用しています。パッチバージョンまでは指定をしていませんが、本書で扱う内容に関しては特に問題とはならないはずです。

CakePHPは動作要件としていくつかの拡張モジュールの導入を必要とします。そのため、公式のDockerイメージをベースとして、いくつかのPHPの拡張モジュールを追加する形で独自のDockerイメージを構築しています。
必要とされるPHP拡張モジュールについては、公式ドキュメントの「システム要件」を参照してください。

https://book.cakephp.org/4/ja/installation.html#id2

これらに加えて、あると便利な`APUc` ・`Xdebug`をインストールしているのが、今回利用しているアプリケーションサーバーのイメージです。

また、Composerの実行ファイルも配置してPATHを通しています[^install-composer]。詳細はDockerfileを直接確認してください。

https://github.com/o0h/o0h-zenn-articles-public/blob/main/books/re-introduction-php-waf-development/sample/part1/.runtime/app/Dockerfile

Xdebugの設定については、コンテナ起動時にバインドマウントすることで注入しています。設定ファイルは`.runtime/app/shared_files/mount`以下に、コンテナ上のパスと同じ構造になるようにディレクトリを設けています。

https://github.com/o0h/o0h-zenn-articles-public/blob/main/books/re-introduction-php-waf-development/sample/part1/.runtime/app/shared_files/mount/usr/local/etc/php/conf.d/xdebug.ini



[^install-composer]: Composerのインストールについては、第4章で取り扱います。
