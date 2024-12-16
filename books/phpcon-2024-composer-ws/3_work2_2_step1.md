---
title: "├ W-②: STEP-1 パッケージ情報の取得"
---

# STEP-1 パッケージ情報の取得

## A. 課題について

### 背景

Comopserは`require`コマンドで「このパッケージを追加してください」と命令を受けとり、パッケージに関する必要な情報をレポジトリに問い合わせます。そこから、まずはパッケージごとの`composer.json`相当の情報を済ませられるように処理を進めるのです。

そのためには、Packasitの用意しているAPIにリクエストを送ります。
エンドポイントは ` https://repo.packagist.org/p2/[vendor]/[package].json`です。例として、`monolog/monolog`のデータを見てみましょう。

URL: https://repo.packagist.org/p2/monolog/monolog.json

![](/images/3_work2_2_step1/package-json-sample.png)
_実際のURLを開いて、Chrome拡張でJSONデータを整形して表示したもの_

このAPIに関する概要は、公式ドキュメントの記載を参照してください。

https://packagist.org/apidoc#get-package-data

取得されるデータの中身については、次のステップ以降で見ていきます。
まずは取得するところまでのコードを書き進めてしまいましょう！

### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/2_require/step1.php`    |
| 実装サンプル   | `/opt/example/2_require/step1.php` |

step1のファイルにある `procedure2_1`関数の内部を実装してきます。
この関数は、パッケージ名をstringとして受け取り、APIのレスポンスコンテンツをJSON文字列として返すものです。

1. 受け取ったパッケージ名から、リクエスト先エンドポイントのURLを組み立てる
2. リクエストを実行して、戻ってきたデータをJSON文字列として返す

### 参考: 関連の深いComposerのファイル・コード

* https://github.com/composer/composer/blob/2.8.3/src/Composer/Repository/CompositeRepository.php

## B. 実装例とコードの解説

作業前のファイルの全景

```php
<?php
const BASE_PACKAGE_ENDPOINT_TEMPLATE = 'https://repo.packagist.org/p2/%s.json';

$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_1` が呼ばれます
}

function procedure2_1 (string $requirePackageName): string
{
    /* === STEP-1 ココから === */




    return ''; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してください
    /* === STEP-2 ココまで === */
}
```

### 1. 受け取ったパッケージ名から、リクエスト先エンドポイントのURLを組み立てる

予め `BASE_PACKAGE_ENDPOINT_TEMPLATE` という定数で、エンドポイントの雛形が定義されています。これを置換することで、リクエスト先のURLが組み立てられます。

:::message
実際にこうした処理を考える際には、しっかりとリクエスト結果を見てハンドリングすることが欠かません。
このハンズオンではコードを簡易にするために端折っています。
:::

```php
<?php
const BASE_PACKAGE_ENDPOINT_TEMPLATE = 'https://repo.packagist.org/p2/%s.json';

$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_1` が呼ばれます
}

function procedure2_1 (string $requirePackageName): string
{
    /* === STEP-1 ココから === */
  
    // 追記ココから
    $packageEndpoint = sprintf(BASE_PACKAGE_ENDPOINT_TEMPLATE, $requirePackageName);
    $packageVersionedMetaListJson = file_get_contents($packageEndpoint);
    // 追記ココまで

    return ''; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してください
    /* === STEP-2 ココまで === */
}
```

### 2. リクエストを実行して、戻ってきたデータをJSON文字列として返す

TODO部分を消して、取得結果を returnします

```php
<?php
const BASE_PACKAGE_ENDPOINT_TEMPLATE = 'https://repo.packagist.org/p2/%s.json';

$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_1` が呼ばれます
}

function procedure2_1 (string $requirePackageName): string
{
    /* === STEP-1 ココから === */
  
    $packageEndpoint = sprintf(BASE_PACKAGE_ENDPOINT_TEMPLATE, $requirePackageName);
    $packageVersionedMetaListJson = file_get_contents($packageEndpoint);

    // 変更ココから
    return $packageVersionedMetaListJson；
    /* === STEP-2 ココまで === */
}
```



---

STEP-1はコレで完了です。

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

『✅ Step1が完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/3_work2_2_step1/finish.gif)
