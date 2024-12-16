---
title: "├ W-③: STEP-4 オートローダーの完成と配置"
---

# STEP-4 オートローダーの完成と配置

## A. 課題について

### 背景

ここまで「クラスやファイルを読み込むための設定」「クラスのオートローディングの仕組み」を作ってきました。これを、プロジェクトのアプリケーションから利用可能にする必要があります。

本家のComposerでいうところの `vendor/autoload.php` に相当する処理を用意しましょう！
注意しなければならないのは、「ファイルやクラスを読み込む処理を書く」のではなく「それを行うファイルを生み出す処理を書く」という点です。

### 課題

|                | コンテナ内のパス                         |
| -------------- | ---------------------------------------- |
| 作業用ファイル | `/opt/work/3_dump-autoload/step4.php`    |
| 実装サンプル   | `/opt/example/3_dump-autoload/step4.php` |

1. イーガーロード対象のファイルをすべて読み込む
2. PSR-4のクラスローダーを読み込み、それに名前空間マップの設定を注入する
3. PSR-4のクラスローダーを、`spl_autoload_register` で登録する
4. 1-3の内容を「スクリプトファイルとして書き出せる」ように、stringとして変数に格納する

これらの処理を、関数 `procedure3_4` の処理の一部として実装していきます。
なお、関数呼び出し時に引数として次の情報を受け取っています

- `string $vendorDirPath`: できあがった `autoload.php` を配置するディレクトリ
- `string $psr4ClassMapPath`: STEP-1で作った名前空間マップファイルのパス
- `string $eagerLoadFilesPath`: STEP-2で作ったイーガーロード対象リストファイルのパス 
- `string $psr4ClassLoaderPath`: PSR-3で作ったPSR-4のクラスローダー

## B. 実装例とコードの解説

作業前のファイルの全景

```php
<?php
processDumpAutoload(__DIR__ . '/vendor'); // この内部で `procedure3_1` が呼ばれます

function procedure3_4(
    string $vendorDirPath,
    string $psr4ClassMapPath,
    string $eagerLoadFilesPath,
    string $psr4ClassLoaderPath,
): void
{
    $autoloaderScript = <<<CODE
<?php
/* === STEP-4 ココから === */




/* === STEP-4 ココまで === */
CODE;

    file_put_contents("{$vendorDirPath}/autoload.php", $autoloaderScript);
}
```

### 1. イーガーロード対象のファイルをすべて読み込む

後ほど「処理を行うのではなく、処理を行うファイルを生成する」ための加工を行うとして、まずはシンプルに実装してしまいましょう。

`$eagerLoadFilesPath` は引数で受け取っています。この中身を、逐一requireしていけば良いでしょう。

```php
$eagerLoadFiles = require '{$eagerLoadFilesPath}';
foreach ($eagerLoadFiles as $file) {
    require_once $file;
}
```

### 2. PSR-4のクラスローダーを読み込み、それに名前空間マップの設定を注入する

こちらも、必要な知識は引数として注入されています。

まずは、`$psr4ClassLoaderPath` を読み取ってクラスを手に入れましょう。

このクラスローダーは、名前空間マップをコンストラクタで受け取るように設計されています。そこで使うのが、`$psr4ClassMapPath` です。

**参考**

```php
class Psr4ClassLoader
{
    private array $psr4ClassMap;

    public function __construct(
        string $psr4ClassMapPath,
    )
    {
        // `return` を持っているクラスをrequireしているので、そのまま変数に代入できる
        $this->psr4ClassMap = require $psr4ClassMapPath;
    }

    public function loadClass(string $class): void
    {
    }
}
```

これを用いて、次に書くコードはこのようになります。

```php
$eagerLoadFiles = require '{$eagerLoadFilesPath}';
foreach ($eagerLoadFiles as $file) {
    require_once $file;
}

// ココから追記
require_once '{$psr4ClassLoaderPath}';

\$psr4ClassLoader = new Psr4ClassLoader('{$psr4ClassMapPath}');
```

### 3. PSR-4のクラスローダーを、`spl_autoload_register` で登録する

最後に、`Psr4ClassLoader::loadClass()` を、 `spl_autoload_register`に渡します。

```php
spl_autoload_register([\$psr4ClassLoader, 'loadClass']);
```

### 4. 1-3の内容を「スクリプトファイルとして書き出せる」ように、stringとして変数に格納する

`step4.php` には、あらかじめ「渡されたコンテンツを、 `autoload.php` として書き出す」手続きが設置されています。

```php
file_put_contents("{$vendorDirPath}/autoload.php", $autoloaderScript);
```

1-3で実装した内容を、 変数 `$autoloaderScript` に格納できるように変形しましょう。

```php
    $autoloaderScript = <<<CODE
<?php
/* === STEP-4 ココから === */
\$eagerLoadFiles = require '{$eagerLoadFilesPath}';
foreach (\$eagerLoadFiles as \$file) {
    require_once \$file;
}

if (!class_exists('Psr4ClassLoader')) {
    require_once '{$psr4ClassLoaderPath}';
}

\$psr4ClassLoader = new Psr4ClassLoader('{$psr4ClassMapPath}');
spl_autoload_register([\$psr4ClassLoader, 'loadClass']);
/* === STEP-4 ココまで === */
CODE;

file_put_contents("{$vendorDirPath}/autoload.php", $autoloaderScript);
```

ポイントは、変数の前の `$` をバックスラッシュでエスケープしていることでしょうか。
その他は、特別な処理は加えていません。

---

STEP-4はコレで完了です。

終わったら、課題の内容をクリアする実装ができているかを確認しましょう。
用意されている`make` コマンドで検査できます。コンテナ内ではなく、コンテナの外側（ホストマシン）で実行することに注意してください。

**ホストから**

```
make work3
```

もし`make`が使えないなど、コンテナ内で行う場合には下記のコマンドを利用してください。

**コンテナ内で**

```sh
php /opt/work/3_dump-autoload/main.php
```

『✅ StepXが完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/4_work3_2_step4/finish.gif)

## 🎉 Tinyposerの”require"を味わう

では、出来上がったautoloaderでクラスや関数を読み込める喜びを味わいましょう！

ホストマシンから、次のどちらかのコマンドを実行してください。(コンテナに入って、bashを使います)

```sh
// Makefileのタスクを使う場合
make sh

// Docker Composeを直接使う場合
docker compose run --rm -it app bash
```

次にコンテナ内で、

1.  PHPを対話モードで立ち上げて
2. `autoload.php`を読み込んで
3. `composer install`したクラスを何かしら使ってみる

という操作をしてみます。

```sh
34a5779e2b7f:/opt# php -a
Interactive shell

php > require '/opt/work/3_dump-autoload/vendor/autoload.php';
php > $logger = new \Psr\Log\NullLogger;
php > var_dump(get_debug_type($logger));
php shell code:1:
string(18) "Psr\Log\NullLogger"
```

ちゃんと読み込まれているのが確認できたでしょうか✨️

![](/images/4_work3_2_step4/demo.gif)
