---
title: "開発環境を用意する"
---

## この章で行うこと

- ローカルでの開発環境を進められるように、Dockerコンテナを用意する
- 用意した環境でPHPアプリケーションが動く事を確認する

## 環境構築用のファイルを用意する

まずは、手元の環境でPHPのWebアプリケーションが動作な状態を作ります。
Dockerを利用しますので、インストールと起動を行っておいてください(Dockerについての説明は割愛します)

ただし、環境の構築についてはこのbookの本題ではありませんので、詳しい解説は取り扱いません。

この章で構築する環境は、以下の要件を満たします。

- Webサーバー
  - Dockerオフィシャルイメージ / php:8.3-apache-bookworm
- データベース

  - Dockerオフィシャルイメージ / mysql:8

- PHP Extension
  - Xdebug(3.3)
- PHP Tool
  - Composer(latest)

サポートレポジトリの[tag:Chapter3-localenv](https://github.com/o0h/kantan-fw-lite/releases/tag/Chapter3-localenv)から、必要なファイルを入手できます。  
もしくは、https://github.com/o0h/kantan-fw-lite/archive/refs/tags/Chapter3-localenv.zip をダウンロードして、解凍してください。

以下のディレクトリ構成になっています。

```
.
├── .editorconfig
├── .runtime
│   ├── .env
│   ├── app
│   │   ├── Dockerfile
│   │   └── shared_files
│   │       ├── copy
│   │       └── mount
│   ├── compose.yml
│   └── database
│       └── shared_files
│           └── docker-entrypoint-initdb.d
├── LICENSE
├── Makefile
├── README.md
└── app
    └── public
        └── index.php
```

PHPの開発は`app`ディレクトリ配下で行います。

これは、よくある「(``vendor` 等との対比で)アプリケーションソースを配置する場所」ではないことに注意してください。このディレクトリがDockerコンテナ内の`/var/www/html`にマウントされます。3章以降で、フレームワーク本体を開発する `fw` ディレクトリや、アプリケーションを開発する `src` ディレクトリを追加することになります。

`.runtime` ディレクトリは、実行環境に関する設定や出力されるファイルを格納しています。

ルートに`compose.yml`が設置されているのと、同階層でサービスごとにディレクトリを設置しています。`.runtime/app`はアプリケーションサーバーに関する内容、`.runtime/databse`はデータベースサーバーに関する内容を扱っています。
`shared_files`以下は、Dockerイメージにコピーされるファイル(`copy`ディレクトリ下)やコンテナにマウントされるファイル(`mount`ディレクトリ下)を設置する場所です。これらは、イメージないしコンテナ内の設置パスと同じ構造となるように命名されています。データベースサーバーに関しては、独自のイメージをビルドしないので、`copy` /`mount`を区別していません。

ファイルを設置できたら、makeコマンドを利用して環境の構築・起動が可能です。

詳しい内容は`Makefile`の中身をご覧いただくとして、ここでは主なコマンドだけ掻い摘んで説明します。

- `make` : イメージのビルドとコンテナの起動(`init`と`up`を組み合わせたもの)
- `make init`: イメージのビルド
- `make up` : コンテナの起動
- `make down`: コンテナの停止

## 環境を起動する

ファイルが適切に設置できたら、以下のコマンドを実行してください

```console
make
```

`kantan-fw-app-lite`と`kantan-fw-lite-db`の2つのコンテナについて、`Started`と表示されれば成功です。

続いて、ブラウザから `http://localhost:8082/hoge/fuga/piyo`にアクセスしてみてください。
`app/public/index.php`の内容として、`pnpinfo()`の結果が表示されていれば成功です。PHPが実行できない・404が返ってくると言った場合は、サーバーの設定やアクセス権限などが誤っている可能性があるので、設定ファイルを見直してみましょう。

![](/./images/books/kantan-fw-lite/setup/1.png)
_正常に起動が完了すれば、phpinfo()の実行結果が表示されます_
