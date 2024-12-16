---
title: "2章 まずはCakePHPを動かしてみる"
free: true
---

# CakePHP アプリケーションの雛形を用意する

1 章で用意した環境を使って、早速 CakePHP アプリケーションの作成を始めていきましょう。
本章では、CakePHP の公式ドキュメントに含まれている「コンテンツ管理チュートリアル」を題材とします。

https://book.cakephp.org/4/ja/tutorials-and-examples.html

## プロジェクトの雛形を設置する

`Makefile` に定義されているタスクを利用して、全てのサービスを立ち上げてください。

```sh
$ make
docker-compose -f .runtime/docker-compose.yml build
# 省略
[+] Running 3/3
 ⠿ Container reintroduction-php-fw-db     Started                                                                                                                                                                 2.8s
 ⠿ Container reintroduction-php-fw-app    Started                                                                                                                                                                 2.7s
 ⠿ Container reintroduction-php-fw-nginx  Started
$ make ps
COMPOSE_PROJECT_NAME=reintroduction-php-fw-app docker compose -f .runtime/docker-compose.yml ps
NAME                          IMAGE                           COMMAND                  SERVICE             CREATED              STATUS              PORTS
reintroduction-php-fw-app     reintroduction-php-fw-app-app   "docker-php-entrypoi…"   app                 About a minute ago   Up About a minute   9000/tcp
reintroduction-php-fw-db      mysql:8                         "docker-entrypoint.s…"   db                  About a minute ago   Up About a minute   33060/tcp, 0.0.0.0:3341->3306/tcp
reintroduction-php-fw-nginx   nginx:alpine                    "/docker-entrypoint.…"   nginx               About a minute ago   Up About a minute   0.0.0.0:8080->80/tcp
```

立ち上がったアプリケーションコンテナ内で、`make sh`で対話的なシェル操作を実行できます。

```sh
$ make sh
COMPOSE_PROJECT_NAME=reintroduction-php-fw-app docker compose -f .runtime/docker-compose.yml exec app sh
/opt/project # ls
Makefile       README.md      app            composer.json  composer.lock  vendor
/opt/project #
```

このコンテナの中では、既に Composer が利用可能な状態で配置されています。

```sh
/opt/project # which composer
/usr/local/bin/composer
/opt/project # composer --version
Composer version 2.5.3 2023-02-10 13:23:52
```

では、チュートリアルにある通り Composer のプロジェクト作成コマンドを利用して、CakePHP プロジェクトの雛形を設置します。

1 章の時点では PHP の動作確認と Nginx の設定用に `app/webroot/index.php`を設置していました。これを置換する形でアプリケーションを展開していこうと思います。
そのために、不要なファイルとディレクトリを削除します。`app`ディレクトリと、ルートに置いてある Composer の設定ファイルが不要となります。

```
/opt/project # ls
Makefile    README.md    app    composer.json    composer.lock    vendor
/opt/project # rm -rf app composer.* vendor
/opt/project # ls
Makefile   README.md
```

準備が出来たので、CakePHP プロジェクトを配置します。

```sh
/opt/project # composer create-project --prefer-dist cakephp/app:4.* app
Creating a "cakephp/app:4.*" project at "./app"
Info from https://repo.packagist.org: #StandWithUkraine
Installing cakephp/app (4.4.2)
  - Downloading cakephp/app (4.4.2)
  - Installing cakephp/app (4.4.2): Extracting archive
Created project in /opt/project/app
Loading composer repositories with package information
Updating dependencies
Lock file operations: 88 installs, 0 updates, 0 removals
# 省略
22 package suggestions were added by new dependencies, use `composer suggest` to see details.
Generating autoload files
57 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
PHP CodeSniffer Config installed_paths set to ../../cakephp/cakephp-codesniffer,../../slevomat/coding-standard
No security vulnerability advisories found
> App\Console\Installer::postInstall
Created `config/app_local.php` file
Created `/opt/project/app/logs` directory
Created `/opt/project/app/tmp/cache/views` directory
Set Folder Permissions ? (Default to Y) [Y,n]? y
Permissions set on /opt/project/app/tmp/cache/views
Permissions set on /opt/project/app/logs
Updated Security.salt value in config/app_local.php
```

Composer によるパッケージの配置処理を終えると _Set Folder Permissions ? (Default to Y) [Y,n]?_ と質問されるので、`y`と答えてください。これは、CakePHP が更新するファイルについてアプリケーションからの書き込み権限を与えるものです。

ここまで出来たら、`http://localhost:8080`を開いてみてください。次のような可愛いトップページが表示されたらプロジェクトの設置作業は完了です。

![image-20230213004052980](/images/image-20230213004052980.png)

余談ですが、🍓 は CakePHP4 系のコードネームです。3 系は「[Red Velvet](https://ja.wikipedia.org/wiki/%E3%83%AC%E3%83%83%E3%83%89%E3%83%BB%E3%83%B4%E3%82%A7%E3%83%AB%E3%83%B4%E3%82%A7%E3%83%83%E3%83%88%E3%83%BB%E3%82%B1%E3%83%BC%E3%82%AD)」でした。

https://twitter.com/cakephp/status/877532376334782464

https://twitter.com/cakephp/status/877888565442560000

:::message

**修正が必要な点**

ディレクトリ構造が変わり、`./composer.json`が`./app/composer.json`に移動しました。
これに伴って、Makefile に定義してある Composer インストールを実施するタスク`make init`が正常に動作しなくなります。

対応するために、アプリケーションコンテナに環境変数を設定してください。ワーキングディレクトリを`app`下にしてしまうことで、簡単に対処できます。(もちろん、Makefile 側を修正する方法でも問題ありません。)

```diff
--- docker-compose.yml
+++ docker-compose.yml
@@ -13,7 +13,7 @@ services:
       - ../:/opt/project:cached
       - ./app/shared_files/mount/usr/local/etc/php/conf.d/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini:cached
       - ./app/shared_files/mount/var/log/xdebug:/var/log/xdebug
-    working_dir: /opt/project
+    working_dir: /opt/project/app
     environment:
       LANG: ja_JP.UTF-8
       TZ: Asia/Tokyo
```

:::

## 出来上がったファイルの確認

Composer の`create-project`については、4 章で扱います。
実行結果を見ると、雛形に含まれないファイルやディレクトリが作成されていることに気付きます。[^post-install-src]

```sh
> App\Console\Installer::postInstall
Created `config/app_local.php` file
Created `/opt/project/app/logs` directory
Created `/opt/project/app/tmp/cache/views` directory
```

これらは、`.gitignore`にて Git 管理しないように指定されているファイル群です。

```sh
 $ cat app/.gitignore
# CakePHP specific files #
##########################
/config/app_local.php
/config/.env
/logs/*
/tmp/*
/vendor/*
# (省略)
```

このタイミングで、簡単に役割について触れておきます。

#### `/config/app_local.php`

`app.php` はアプリケーションの設定を管理するファイルですが、この `_local` ファイルでは環境固有の設定内容を扱います。
「全ての環境で、設定項目としては共通するが、その値が違う」といった場合は、環境変数で値を注入することで`app.php`にて扱うことが可能です。一方で、設定項目自体に差分があるものに関しては、`_local`のファイルに記述することで管理の煩雑さから開放されます。例えば「テスト用データベースの設定」などは、開発環境や CI 環境以外では必要とされない項目となることででしょう。

`bootstrap.php`内で、`app.php`の後に読み込まれます(つまり、`app.php`の設定内容を`_local`ファイルで上書きすることができます)。

https://github.com/cakephp/app/blob/4.4.2/config/bootstrap.php#L86-L92

両者のファイルには明確な使い分けのルールはなく、機能的にも同等なため、各々のプロジェクトに応じた基準で利用することが出来ます。同種のファイルが複数存在することに煩雑さを感じる場合は、`_local`ファイルを削除して利用しないのも 1 つの手かも知れません。実際、`_local`ファイルは CakePHP 3.9 になるまでは雛形に含まれていませんでした。[^begin-app-local]

[^begin-app-local]: 実装時の PR[Generate and load app_local\.php by inoas · Pull Request \#713 · cakephp/app](https://github.com/cakephp/app/pull/713)

#### `/config/.env`

これも環境固有のファイルです。名前の通り、環境変数をファイルで管理します。`josegonzalez/dotenv`が利用されています。

https://packagist.org/packages/josegonzalez/dotenv

`app.php`と同様に`bootstrap.php`内で読み込まれており、CakePHP のコアが持つ設定の次に実行されています。
デフォルトではコメントアウトされているので、`.env`ファイルによる環境変数の管理を利用するには、手動で有効化する(コメントアウトを外す)必要があります。

https://github.com/cakephp/app/blob/4.4.2/config/bootstrap.php#L49-L69

#### `logs`と`tmp`

これらは名前の通り、ログファイルやキャッシュなどの一時ファイルを扱うためのディレクトリです。

`/tmp/cache`ディレクトリには、`views`以外にもディレクトリが存在しているのですが、``view`ディレクトリだけが事後的に(プロジェクトの雛形設置時に)作成されるようになっています。これは、`views`のキャッシュについては、その他の(予め`.gitkeep`されている)ディレクトリと異なり、CakePHP のコアの機能を利用しただけではキャッシュが作成されないことが理由なのかと思います。`views`のキャッシュについては、必要に応じて作成・読み込みロジックを任意の箇所に実装してください。
詳細は公式ドキュメントの「ビュー」の項に記載があります。

https://book.cakephp.org/4/ja/views.html

[^post-install-src]: この処理の実態は https://github.com/cakephp/app/blob/4.4.2/src/Console/Installer.php#L89-L105 にあります

# チュートリアルを実施する

ここまでで[コンテンツ管理チュートリアル](https://book.cakephp.org/4/ja/tutorials-and-examples/cms/installation.html#)のページの内容は完了したので、「データベース作成」から進めていきます。

https://book.cakephp.org/4/ja/tutorials-and-examples/cms/database.html

本書で用いている開発環境では、既に「データベースの作成」「ユーザーへの権限付与」「アプリケーションからデータベースへの接続情報の設定」を完了しています。(そのため、「データベースの設定」の節はスキップすることができます)
これは、Docker の MySQL イメージの機能と`docker-compose.yml`上から設定している環境変数での制御によるものです。詳細が気になる方は、各ファイルを確認してみてください。

アプリケーションからみたデータベースへの接続情報は以下の通りになります。

| 項目       | 設定値                   |
| ---------- | ------------------------ |
| ホスト     | reintroduction-php-fw-db |
| ユーザー名 | app_user                 |
| パスワード | secret                   |
| スキーマ名 | app_db                   |

これらの情報を組み立てた DSN を、`DATABASE_URL`という環境変数名で渡すことで CakePHP は接続情報を得ることが出来ます。
これは、`app_local.php` で設定値に環境変数を読み込ませていることで機能しています。

https://github.com/cakephp/app/blob/4.4.2/config/app_local.example.php#L60

## テーブルの用意

チュートリアルに記載されている DDL 文を実行します。(利用するスキーマ名が異なることに注意してください)

Makefile 中に、データベースコンテナでのシェル実行のためのタスクが定義されているので、これを利用することで簡単に SQL ジック環境に入ることが出来ます。

```sh
$ make sh-db
COMPOSE_PROJECT_NAME=reintroduction-php-fw-app docker compose -f .runtime/docker-compose.yml exec db sh
# mysql -u app_user -psecret app_db
(省略)
mysql> CREATE TABLE users (
    ->     id INT AUTO_INCREMENT PRIMARY KEY,
    ->     email VARCHAR(255) NOT NULL,
    ->     password VARCHAR(255) NOT NULL,
    ->     created DATETIME,
    ->     modified DATETIME
    -> );
NT NOT NULL,
    PRIMARY KEY (article_id, tag_id),
    FOREIGN KEY tag_key(tag_id) REFERENCES tags(id),
    FOREIGN KEY article_key(article_id) REFERENCES articles(id)
);

INSERT INTO users (email, password, created, modified)
VALUES
('cakephp@example.com', 'secret', NOW(), NOW());

INSERT INTO articles (user_id, title, slug, body, published, created, modified)
VALUES
(1, 'First Post', 'first-post', 'This is the first post.', 1, now(), now());Query OK, 0 rows affected (0.04 sec)

mysql>
mysql> CREATE TABLE articles (
    ->     id INT AUTO_INCREMENT PRIMARY KEY,
    ->     user_id INT NOT NULL,
    ->     title VARCHAR(255) NOT NULL,
    ->     slug VARCHAR(191) NOT NULL,
    ->     body TEXT,
    ->     published BOOLEAN DEFAULT FALSE,
    ->     created DATETIME,
    ->     modified DATETIME,
    ->     UNIQUE KEY (slug),
    ->     FOREIGN KEY user_key (user_id) REFERENCES users(id)
    -> ) CHARSET=utf8mb4;
Query OK, 0 rows affected (0.03 sec)

mysql>
mysql> CREATE TABLE tags (
    ->     id INT AUTO_INCREMENT PRIMARY KEY,
    ->     title VARCHAR(191),
    ->     created DATETIME,
    ->     modified DATETIME,
    ->     UNIQUE KEY (title)
    -> ) CHARSET=utf8mb4;
Query OK, 0 rows affected (0.03 sec)

mysql>
mysql> CREATE TABLE articles_tags (
    ->     article_id INT NOT NULL,
    ->     tag_id INT NOT NULL,
    ->     PRIMARY KEY (article_id, tag_id),
    ->     FOREIGN KEY tag_key(tag_id) REFERENCES tags(id),
    ->     FOREIGN KEY article_key(article_id) REFERENCES articles(id)
    -> );
Query OK, 0 rows affected (0.03 sec)

mysql>
mysql> INSERT INTO users (email, password, created, modified)
    -> VALUES
    -> ('cakephp@example.com', 'secret', NOW(), NOW());
Query OK, 1 row affected (0.01 sec)

mysql>
mysql> INSERT INTO articles (user_id, title, slug, body, published, created, modified)
    -> VALUES
    -> (1, 'First Post', 'first-post', 'This is the first post.', 1, now(), now());
Query OK, 1 row affected (0.01 sec)
```

## アプリケーションの作成

下準備が整ったので、CMSアプリケーションを作成していきましょう。

「データベース作成」の続きの内容(最初のモデルの作成)と、次ページの「Articles コントローラーの作成」の内容を実施することで、シンプルなCMSが出来上がります。

https://book.cakephp.org/4/ja/tutorials-and-examples/cms/articles-controller.html



![image-20230213024837638](/images/image-20230213024837638.png)
*記事一覧のビュー*

![image-20230213024926145](/images/image-20230213024926145.png)
*個別記事の表示のビュー*

![image-20230213024959651](/images/image-20230213024959651.png)
*編集のビュー*

![image-20230213025102597](/images/image-20230213025102597.png)
*記事の更新の成功*

続く「タグとユーザー」では、CLIによるファイル自動生成機能が利用されています。
実行すると以下のような結果になります。

```sh
$ make sh
COMPOSE_PROJECT_NAME=reintroduction-php-fw-app docker compose -f .runtime/docker-compose.yml exec app sh
/opt/project/app # bin/cake bake model users
One moment while associations are detected.

Baking table class for Users...

Creating file /opt/project/app/src/Model/Table/UsersTable.php
Wrote `/opt/project/app/src/Model/Table/UsersTable.php`
Deleted `/opt/project/app/src/Model/Table/.gitkeep`

Baking entity class for User...

Creating file /opt/project/app/src/Model/Entity/User.php
Wrote `/opt/project/app/src/Model/Entity/User.php`
Deleted `/opt/project/app/src/Model/Entity/.gitkeep`

Baking test fixture for Users...

Creating file /opt/project/app/tests/Fixture/UsersFixture.php
Wrote `/opt/project/app/tests/Fixture/UsersFixture.php`
Deleted `/opt/project/app/tests/Fixture/.gitkeep`
Bake is detecting possible fixtures...

Baking test case for App\Model\Table\UsersTable ...

Creating file /opt/project/app/tests/TestCase/Model/Table/UsersTableTest.php
Wrote `/opt/project/app/tests/TestCase/Model/Table/UsersTableTest.php`
Done
```

本書では、チュートリアルの内容に触れるのはここまでとします。少なくとも、用意した開発環境でCakePHPのアプリケーションが実際に開発できることが確認できました。

以降も、CakePHPの基本的な機能やFWの持つ力を引き出すために知っておくべき機能が丁寧に解説されています。
CakePHPを利用した開発が初めてという方は、本書の次の章に進む前に是非とも一通りのチュートリアルコンテンツをなぞってみてください。

:::message
**手動で作成したファイルでIDEに怒られるのが気になる場合**    

プロパティの未定義などで、IDEの持つ静的解析機能によって警告が出ているかと思います。気持ちよく開発するためには、アノテーションを付与するのが望ましいでしょう。

CakePHPのコアデベロッパーであるdereuromark氏が、「CakePHP IdeHelper Plugin」を提供しています。

@[card](https://github.com/dereuromark/cakephp-ide-helper)

このプラグインを利用することで、既存ファイルに自動的にアノテーションを追加することが出来ます。気になる方は、[ドキュメント](https://github.com/dereuromark/cakephp-ide-helper/tree/master/docs)を覗いてみてください。
こうした開発支援ツールは、「16章 CakePHPの開発効率を支えるためのツール」で取り上げていきます。

:::
