---
title: "├ W-③: STEP-2 ファイルのイーガーロード"
---

# STEP-2 ファイルのイーガーロード

## A. 課題について

### 背景

Tinyposerで、もう1つ対応したいローディング機構が「イーガーロード」です。`composer.json`の`autolaod.files`に該当します。

https://getcomposer.org/doc/04-schema.md#files

関数や定数など、オートローディングできないものに関しては手動でincludeしていくしかありません。パッケージによっては、クラスだけでなく関数を提供するものもあります。そうした場合に利用する機能です。polyfillなどは、典型的な例でしょう。

**symfony/polyfillの例(一部割愛)**

```json
  "autoload": {
      "psr-4": { "Symfony\\Polyfill\\": "src/" },
      "files": [
          "src/bootstrap.php",
          "src/Apcu/bootstrap.php",
          "src/Ctype/bootstrap.php",
          "src/Uuid/bootstrap.php",
          "src/Iconv/bootstrap.php",
          "src/Intl/Grapheme/bootstrap.php",
          "src/Intl/Idn/bootstrap.php",
          "src/Intl/Icu/bootstrap.php",
          "src/Intl/MessageFormatter/bootstrap.php",
          "src/Intl/Normalizer/bootstrap.php",
          "src/Mbstring/bootstrap.php"
      ]
  },
```

対応方法としては、`vendor/autoload.php`読み込み時に該当ファイルを読み込んでしまえばOKです。
そのために、後ほど作るオートローダーに「即時で読み込んで欲しいファイルのパス一覧」を書き出す仕組み作りましょう。

### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/3_require/step2.php`    |
| 実装サンプル   | `/opt/example/3_require/step2.php` |

1. `composer.json` を読み込んでパッケージ情報を解釈し、`autoload.files`の内容を取得する
   * 存在しなければcontinue
2. 取得した内容から、フィルパスのリストを作成する
   * 後続のステップで、これは「foreachで回してrequireする」ような使い方をされます

### 参考: 関連の深いComposerのファイル・コード

* https://github.com/composer/composer/blob/2.8.3/src/Composer/Autoload/AutoloadGenerator.php#L446-L451

## B. 実装例とコードの解説

作業前のファイルの全景

```php
processDumpAutoload(__DIR__ . '/vendor'); // この内部で `procedure3_1` が呼ばれます

function procedure3_2(string $vendorDirPath): string
{
    $eagerLoadFiles = [];

    $packageFiles = glob("{$vendorDirPath}/*/*/composer.json");
    foreach ($packageFiles as $packageFile) {
        /* === STEP-2 ココから === */



        /* === STEP-2 ココまで === */
    }

    $eagerLoadFilesPath = $vendorDirPath . '/autoload_files.php';

    file_put_contents(
        $eagerLoadFilesPath,
       '<?php return '.var_export($eagerLoadFiles, true).';',
    );

    return $eagerLoadFilesPath;
}
```

### 1. `composer.json` を読み込んでパッケージ情報を解釈し、`autoload.files`の内容を取得する

ものすごくざっくりいうと「さっきPSR-4対応とやったのと似た感じのコード」が必要です。

```php
processDumpAutoload(__DIR__ . '/vendor'); // この内部で `procedure3_1` が呼ばれます

function procedure3_2(string $vendorDirPath): string
{
    $eagerLoadFiles = [];

    $packageFiles = glob("{$vendorDirPath}/*/*/composer.json");
    foreach ($packageFiles as $packageFile) {
        /* === STEP-2 ココから === */

        // ココから追記
        $package = json_decode(file_get_contents($packageFile), true);
        $packageEagerLoadFiles = $package['autoload']['files'] ?? null;
        if (!$packageEagerLoadFiles) {
            continue;
        }
        // ココまで追記
      

        /* === STEP-2 ココまで === */
    }

    $eagerLoadFilesPath = $vendorDirPath . '/autoload_files.php';

    file_put_contents(
        $eagerLoadFilesPath,
       '<?php return '.var_export($eagerLoadFiles, true).';',
    );

    return $eagerLoadFilesPath;
}


```



### 2. 取得した内容から、フィルパスのリストを作成する

こちらも複雑なことはしなくて大丈夫です。

名前空間マップと異なり、ファイル(パス)に関しては単純なリストで問題ありません。なので、見つけたものをどんどん末尾に追加していきましょう。

```php
processDumpAutoload(__DIR__ . '/vendor'); // この内部で `procedure3_1` が呼ばれます

function procedure3_2(string $vendorDirPath): string
{
    $eagerLoadFiles = [];

    $packageFiles = glob("{$vendorDirPath}/*/*/composer.json");
    foreach ($packageFiles as $packageFile) {
        /* === STEP-2 ココから === */

        $package = json_decode(file_get_contents($packageFile), true);
        $packageEagerLoadFiles = $package['autoload']['files'] ?? null;
        if (!$packageEagerLoadFiles) {
            continue;
        }

      
          // ココから追記
          $packageDir = dirname($packageFile);
        foreach ($packageEagerLoadFiles as $eagerLoadFile) {
            $eagerLoadFiles[] = "{$packageDir}/{$eagerLoadFile}";
        }
                // ココまで追記
      

        /* === STEP-2 ココまで === */
    }

    $eagerLoadFilesPath = $vendorDirPath . '/autoload_files.php';

    file_put_contents(
        $eagerLoadFilesPath,
       '<?php return '.var_export($eagerLoadFiles, true).';',
    );

    return $eagerLoadFilesPath;
}
```

---

STEP-2はコレで完了です。

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

『✅ Step3が完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/4_work3_2_step2/finish.gif)
