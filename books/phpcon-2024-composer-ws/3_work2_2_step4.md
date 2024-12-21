---
title: "├ W-②: STEP-4 composer.lockの作成"
---

# STEP-4 composer.lockの作成

## A. 課題について

### 背景

`composer.lock`も作らないとですよね！

Tinyposerの場合、「パッチバージョンまで`composer.json`で指定しているなら、`.lock`がなくても問題がないのではないか？」と考える方もいるかも知れません。(もしかしたら、 WORK-1での作業を通じて「なぜ必要か」に察しが付く人もいるかも知れませんね！)

`.lock` ファイルの仕事はただ単に「プロジェクトとして、依存の完全な列挙を行う」だけにとどまりません。少なくとも`dist`の情報を持たせておくことで、処理を効率化できます。

`composer.json` が `require(-dev)`フィールドで`{パッケージ名: バージョン(の範囲)}`をマップ形式で記述していたのに対して、`composer.lock`は`packages(-dev)` フィールドで`[パッケージ情報]`のリストを持ちます。
Tinyposerでは、”それっぽい”フィールドに絞って `.lock` ファイルを作っていくことにしましょう。

最後に、`conent-hash`を追加します。Tinyposerの範疇では使われることはない値ですが、やっぱりComposerといえば「content-hashがコンフリクトした！！」という体験ですよね。誰しも見覚えがあるのではないでしょうか。これがないと、実際のComopserで`composer install`実行時にちゃんと動かなくなってしまいます。

完成形のイメージは、次のとおりです。

```json
{
    "packages": [
        {
            "name": "aura/cli",
            "version": "2.2.0",
            "source": {
                "url": "https://github.com/auraphp/Aura.Cli.git",
                "type": "git",
                "reference": "d69cfa6d81e94e13d831e38ad9e3d53aa0f08a8b"
            },
            "dist": {
                "url": "https://api.github.com/repos/auraphp/Aura.Cli/zipball/d69cfa6d81e94e13d831e38ad9e3d53aa0f08a8b",
                "type": "zip",
                "shasum": "",
                "reference": "d69cfa6d81e94e13d831e38ad9e3d53aa0f08a8b"
            },
            "license": [
                "BSD-2-Clause"
            ]
        },
        {
            "name": "psr/log",
            "version": "3.0.2",
            "source": {
                "url": "https://github.com/php-fig/log.git",
                "type": "git",
                "reference": "f16e1d5863e37f8d8c2a01719f5b34baa2b714d3"
            },
            "dist": {
                "url": "https://api.github.com/repos/php-fig/log/zipball/f16e1d5863e37f8d8c2a01719f5b34baa2b714d3",
                "type": "zip",
                "shasum": "",
                "reference": "f16e1d5863e37f8d8c2a01719f5b34baa2b714d3"
            },
            "license": [
                "MIT"
            ]
        }
    ],
    "packages-dev": [],
    "content-hash": "11446e25354a12ec9f85f7a3160ae6c4"
}
```



### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/2_require/step4.php`    |
| 実装サンプル   | `/opt/example/2_require/step4.php` |

1. 受け取ったパッケージのメタデータから、`.lock`用のパッケージ情報を組み立てる
2. `.lock`ファイルの内容とマージする
   * 既に `.lock`の中に指定パッケージが含まれていれば上書き
   * まだ含まれていなければ追記
3. プロジェクトのデータ(`composer.lock`)の内容から、`content-hash`を更新。ファイルに書き出す

### 参考: 関連の深いComposerのファイル・コード

* https://github.com/composer/composer/blob/2.8.3/src/Composer/Package/Locker.php#L69-L70



## B. 実装例とコードの解説

作業前のファイルの全景

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_4` が呼ばれます
}


function procedure2_4(
    array   $packageMeta,
    ?string $lockFilePath = null,
): void
{
    $lockFilePath ??= __DIR__ . '/composer.lock';

    if (!file_exists($lockFilePath)) {
        file_put_contents($lockFilePath, json_encode(['packages' => [], 'packages-dev' => []]));
    }

    $rootPackageLock = loadJsonFile($lockFilePath);

    /* === STEP-4 ココから === */



    /* === STEP-4 ココまで === */
}

```

### 1. 受け取ったパッケージのメタデータから、`.lock`用のパッケージ情報を組み立てる

まずはパッケージデータを組み立てます。こちらは、とてもシンプルなので「百聞は一見にしかず」でしょう。コードを御覧ください。

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_4` が呼ばれます
}


function procedure2_4(
    array   $packageMeta,
    ?string $lockFilePath = null,
): void
{
    $lockFilePath ??= __DIR__ . '/composer.lock';

    if (!file_exists($lockFilePath)) {
        file_put_contents($lockFilePath, json_encode(['packages' => [], 'packages-dev' => []]));
    }

    $rootPackageLock = loadJsonFile($lockFilePath);

    /* === STEP-4 ココから === */

    // ココから追記
    $packageLock = [
        'name' => $packageMeta['name'],
        'version' => $packageMeta['version'],
        'source' => $packageMeta['source'],
        'dist' => $packageMeta['dist'],
        'license' => $packageMeta['license'],
    ];
    // ココまで追記

    /* === STEP-4 ココまで === */
}
```



### 2.`.lock`ファイルの内容とマージする

`.lock`の `packages`へのマージを行います。このフィールドは(マップでなく)連想配列なので、`composer.json`に比べると少し複雑かもしれません。
まずは、パッケージ名でリストの中を探索してください。該当があればそのインデックスを使います。なければ、リストの末尾に追記します。

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_4` が呼ばれます
}


function procedure2_4(
    array   $packageMeta,
    ?string $lockFilePath = null,
): void
{
    $lockFilePath ??= __DIR__ . '/composer.lock';

    if (!file_exists($lockFilePath)) {
        file_put_contents($lockFilePath, json_encode(['packages' => [], 'packages-dev' => []]));
    }

    $rootPackageLock = loadJsonFile($lockFilePath);

    /* === STEP-4 ココから === */

    $packageLock = [
        'name' => $packageMeta['name'],
        'version' => $packageMeta['version'],
        'source' => $packageMeta['source'],
        'dist' => $packageMeta['dist'],
        'license' => $packageMeta['license'],
    ];

  

    // ココから追記
    $packageName = $packageLock['name'];
    $packageIndex = array_find_key(
        $rootPackageLock['packages'],
        fn($package) => $package['name'] === $packageName,
    );
    if ($packageIndex === null) {
        $packageIndex = count($rootPackageLock['packages']);
    }
    $rootPackageLock['packages'][$packageIndex] =  $packageLock;
  
      // ココまで追記

    /* === STEP-4 ココまで === */
}
```



### 3. プロジェクトのデータ(`composer.lock`)の内容から、`content-hash`を更新。ファイルに書き出す

`.lock`の内容から作ったハッシュ値を、`content-hash`に加えます。
この際に、重要なのは「依存しているパッケージが変化していないこと」です。それ以外の内容はノイズになりかねません。ハッシュ値を算出する前には既存の`contnet-hash`を除去するのを忘れないでください。

最後に、例によって`file_put_contents`でファイルに出力します。

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_4` が呼ばれます
}


function procedure2_4(
    array   $packageMeta,
    ?string $lockFilePath = null,
): void
{
    $lockFilePath ??= __DIR__ . '/composer.lock';

    if (!file_exists($lockFilePath)) {
        file_put_contents($lockFilePath, json_encode(['packages' => [], 'packages-dev' => []]));
    }

    $rootPackageLock = loadJsonFile($lockFilePath);

    /* === STEP-4 ココから === */

    $packageLock = [
        'name' => $packageMeta['name'],
        'version' => $packageMeta['version'],
        'source' => $packageMeta['source'],
        'dist' => $packageMeta['dist'],
        'license' => $packageMeta['license'],
    ];
    $packageName = $packageLock['name'];
    $packageIndex = array_find_key(
        $rootPackageLock['packages'],
        fn($package) => $package['name'] === $packageName,
    );
    if ($packageIndex === null) {
        $packageIndex = count($rootPackageLock['packages']);
    }
    $rootPackageLock['packages'][$packageIndex] =  $packageLock;

  

    // ココから追記
    if (array_key_exists('content-hash', $rootPackageLock)) {
        unset($rootPackageLock['content-hash']);
    }
    unset($rootPackageLock['content-hash']);
    $rootPackageLock['content-hash'] = hash('md5', trim(json_encode($rootPackageLock)));

    file_put_contents($lockFilePath, json_encode($rootPackageLock, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));
    // ココまで追記

    /* === STEP-4 ココまで === */
}
```



---

STEP-4はコレで完了です。

終わったら、課題の内容をクリアする実装ができているかを確認しましょう。
用意されている`make` コマンドで検査できます。コンテナ内ではなく、コンテナの外側（ホストマシン）で実行することに注意してください。

**ホストから**

```
make work2
```

もし`make`が使えないなど、コンテナ内で行う場合には下記のコマンドを利用してください。

**コンテナ内で**

```sh
php /opt/work/2_require/main.php
```

『✅ StepXが完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/3_work2_2_step4/finish.gif)

## 🎉 Tinyposerの”require"を味わう

さて、2つ目のコマンドも実装が完了しました。これにより我々は、`composer.json` `comopser.lock` が手に入るようになった訳です。

この2ファイルがあると、何が出来るでしょうか？・・・そうです、`comopser install`が出来るはずです！遊んでみましょう💪

ホストマシンから、次のどちらかのコマンドを実行してください。(コンテナに入って、bashを使います)

```sh
// Makefileのタスクを使う場合
make sh

// Docker Composeを直接使う場合
docker compose run --rm -it app bash
```

次にコンテナ内で、 `work2`のディレクトリに移動してから `composer install`を実行です。
うまくコマンドが通ったら、`ls`や`tree`で`vendor`ディレクトリの中身を覗いてみてください。

```sh
cd /opt/work/2_require
composer install

ls vendor
```

もちろん、うまく修正すればWORK-1 で作ったスクリプトとうまく組み合わせることも可能です。楽しさが何倍にも膨れ上がるはずなので、是非挑戦してみてください！

![](/images/3_work2_2_step4/demo.gif)
*vendorにパッケージが入ってきてくれて、嬉しい瞬間*
