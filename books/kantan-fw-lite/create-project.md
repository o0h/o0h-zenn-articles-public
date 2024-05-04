---
title: "Composerプロジェクトを準備する"
---

## この章で行うこと

- Composerプロジェクトを初期化(新規作成)する
- 作成してComposerプロジェクトを利用できるようにする

## プロジェクトのディレクトリ構造の全体像

この章から、本格的にフレームワークの開発に入っていきます！
(以降、Webアプリケーションフレームワークを指して **FW** と表記することにします。)

開発を進めていくに当たって、Composerプロジェクトの初期化を行います。

このbookでは、「FW」と「アプリケーション」の2つのプロジェクトを並行して扱っていくことになります。

以降、アプリケーション側のプロジェクト名を _o0h/kiriban-bbs_ 、FW側のプロジェクトを *o0h/kantan-framework-lite*として説明を続けていきます。これは、作成者=筆者をvendor名として _o0h_、パッケージ名をそれぞれ _kiriban-bbs_・*kantan-framework-lite*と設定するための命名です。

最終的なディレクトリの構造と制御・依存の関係は、次の図のようになります。

![](/./images/books/kantan-fw-lite/create-project/1.png)

## Composerプロジェクトの初期化

それでは、実際にComposerプロジェクトを作成していきましょう。

Chapter3で作成した環境には、 `make sh` タスクを実行することでアプリケーションサービスの中に入りbashを実行できます。

まずは、アプリケーション側から始めます。
appディレクトリ直下にcomposer.jsonを作成しましょう。initコマンドを実行することで、対話的にComposerプロジェクトを作成することが出来ます。

```shell
composer init
```

パッケージ名を聞かれるので、任意の名前を入力してください。
vendor名 + chain-caseでの命名が多く見られますが、他プロジェクトからの依存を前提としていなければ実際の動作上の影響もないので、厳密な規則に沿った名前にする必要はありません。
引き続き、Description・Auhtor・Minimum Stability・Package Type・Licenseについて記入をしていってください。
(このサンプルコードはGitHub上にOSSとして公開する予定なので、ここではMITライセンスを選択しています。個人のプロジェクトとして利用する場合には、OSSライセンスをする必要もないでしょう。不要であれば、この項目は削除することも出来ます)

ここまでの全体像は、以下のようになります。

```
root@fa8c293ef1d0:/var/www/html/app# composer init


  Welcome to the Composer config generator



This command will guide you through creating your composer.json config.

Package name (<vendor>/<name>) [root/app]: o0h/kiriban-bbs
Description []:
Author [n to skip]: n
Minimum Stability []: dev
Package Type (e.g. library, project, metapackage, composer-plugin) []: project
License []: MIT

```

続いて、依存性の定義をしていくことになります。今のところは不要なので、`no` を入力してスキップします。

```
Define your dependencies.

Would you like to define your dependencies (require) interactively [yes]? no
Would you like to define your dev dependencies (require-dev) interactively [yes]? no
```

続いて、オートローディングの設定に入ります。
PSR-4のルート名前空間と対応するパスを入力する必要があります。今回は、`O0h\KiribanBbs`を`src/`に対応させたいので、デフォルトのまま(何も入力せずにエンター)で大丈夫です。

最後に、出来上がる `composer.json`の内容についてプレビューが表示されます。問題がなければ、エンターを押してください。

```
{
    "name": "o0h/kiriban-bbs",
    "type": "project",
    "license": "MIT",
    "autoload": {
        "psr-4": {
            "O0h\\KiribanBbs\\": "src/"
        }
    },
    "minimum-stability": "dev",
    "require": {}
}

Do you confirm generation [yes]?
Generating autoload files
Generated autoload files
PSR-4 autoloading configured. Use "namespace O0h\KiribanBbs;" in src/
Include the Composer autoloader with: require 'vendor/autoload.php';
```

今度は `app/fw`ディレクトリに移動して、同様にプロジェクトの初期化を行っていきます。

先程との違いとしては、FWは「他のプロジェクトから利用される(依存先となる)モジュール」になるので、*Package Type*を`project`にしています。[^composer-package-type]

```
root@fa8c293ef1d0:/var/www/html/app/fw# composer init


  Welcome to the Composer config generator



This command will guide you through creating your composer.json config.

Package name (<vendor>/<name>) [root/fw]: o0h/kantan-fw-lite
Description []:
Author [n to skip]: n
Minimum Stability []: dev
Package Type (e.g. library, project, metapackage, composer-plugin) []: library
License []: MIT

Define your dependencies.

Would you like to define your dependencies (require) interactively [yes]? no
Would you like to define your dev dependencies (require-dev) interactively [yes]? no
Add PSR-4 autoload mapping? Maps namespace "O0h\KantanFwLite" to the entered relative path. [src/, n to skip]:

{
    "name": "o0h/kantan-fw-lite",
    "type": "library",
    "license": "MIT",
    "autoload": {
        "psr-4": {
            "O0h\\KantanFwLite\\": "src/"
        }
    },
    "minimum-stability": "dev",
    "require": {}
}

Do you confirm generation [yes]?
Generating autoload files
Generated autoload files
PSR-4 autoloading configured. Use "namespace O0h\KantanFwLite;" in src/
Include the Composer autoloader with: require 'vendor/autoload.php';
```

これで、2つのプロジェクトの初期化が完了しました。

忘れないうちに、vendorディレクトリを`.gitignore`に追加しておきましょう。
プロジェクトルートに `.gitignore`ファイルを作成し、次の行を追加してください

```.gitignore
**/vendor/
```

これで、サブディレクトリ下のvendorディレクトリもGit管理から自動的に外れるようになりました。

## FWの取り込み

アプリケーションのプロジェクトは、FWのプロジェクトに依存して作成されることになります。つまり、Composerでrequireする必要があります。
アプリケーション側の``composer.json` を編集して、

1. repositoriesにローカルファイルを参照可能になるように設定を追加する
2. requireにFWのプロジェクトを追加する

の2つの設定追加を行う必要があります。(実際には、2については`composr require`コマンドで行います)

`app/composer.json`に、次のように  `repositories`フィールドを追加してください。[^composer-repositories]

```diff
         }
     },
     "minimum-stability": "dev",
+    "repositories": [
+      {
+        "type": "path",
+        "url": "./fw",
+        "symlink": true
+      }
+    ],
     "require": {}
 }
```

これは「パッケージ情報を参照可能な情報ソースを追加する」という指示を出すものであり、今回は「FWディレクトリ」の場所をC omposerが覗いてくれれば「何かの利用可能なパッケージがあるよ！」と情報を提供している(参照するように指示している)ことになります。
また、開発中はパッケージの内容が頻繁に変わることを考慮して、vendorディレクトリ下への実ファイルのコピーではなくシンボリックリンクを利用するように `symlink:true`のフラグを立てています。

パッケージの在り処をComposerが把握可能になったので、`composer require`を実施します。
requireするのは、FW側のプロジェクトのnameに指定したパッケージ名です。
次のような実行結果が表示されます。

```
root@fa8c293ef1d0:/var/www/html/app# composer require o0h/kantan-fw-lite
./composer.json has been updated
Running composer update o0h/kantan-fw-lite
Loading composer repositories with package information
Updating dependencies
Lock file operations: 1 install, 0 updates, 0 removals
  - Locking o0h/kantan-fw-lite (dev-main)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 1 install, 0 updates, 0 removals
  - Installing o0h/kantan-fw-lite (dev-main): Symlinking from ./fw
Generating autoload files
No security vulnerability advisories found.
Using version dev-main for o0h/kantan-fw-lite
```

お疲れ様でした、これでアプリケーション側のプロジェクトにFWがインストールされました！

lsコマンドを利用して、どんな様子になっているのかを確認できます。

```
root@fa8c293ef1d0:/var/www/html/app# ls -l vendor/o0h/
total 0
lrwxr-xr-x 1 root root 9 Dec  5 21:25 kantan-fw-lite -> ../../fw/
```

次の章からPHPのコードを書いていきます。

[^composer-package-type]: 多くの場合、作成するプロジェクトのPackage Typeに何を選択しても動作上の問題はありません。興味のある方は、[composer/installers: A Multi\-Framework Composer Library Installer](https://github.com/composer/installers) などのリソースを参照してください。
[^composer-repositories]: 詳細は https://getcomposer.org/doc/05-repositories.md を参照してください
