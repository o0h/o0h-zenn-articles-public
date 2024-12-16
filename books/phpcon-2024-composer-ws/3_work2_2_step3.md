---
title: "├ W-②: STEP-3 composer.jsonの作成"
---

# STEP-3 composer.jsonの作成

## A. 課題について

### 背景

プロジェクトが直接的に依存する(必要としている)パッケージを、 `composer.json`に書き出します。

Tinyposerでは、本当にミニマムなところまで削ぎ落として実装します。 `requires`だけを持つJSONファイルとして、`composer.json`を定義してしまいましょう！

最終的なアウトプットは、次のとおりです。

```json
{
    "require": {
        "aura/cli": "2.2.0",
        "psr/log": "3.0.2"
    }
}
```

本物の `composer.json`の機能と比べると、かなり削ぎ落とされています。

* `require`以外のフィールドを持っていない
* 各パッケージのバージョン指定が、パッチバージョンまで完全指定している。許容範囲の「幅」を持たせない

非常に低機能なものになりますが、おかげで実装も物凄く簡潔なものになります。

先ほど読み込めるようになった「パッケージのメタデータを組み合わせて、特定バージョンの完全な定義ファイル」の中から、`name` `version`を読み取ります。これをkey-valueとしたマップデータを、`composer.json`の `require`フィールドに保持していけばOKです。

### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/2_require/step3.php`    |
| 実装サンプル   | `/opt/example/2_require/step3.php` |

1. パッケージのデータを受け取って、必要なフィールドの値を抜き出し、「パッケージ名」「バージョン」のマップを`require`に持たせる
   * `name`
   * `version`
2. composer.json` として書き込む(既存ファイルがあれば更新する)

## B. 実装例とコードの解説

作業前のファイルの全景

※ 今回は、事前にファイル内に用意されているコードが多いので、気をつけてください。いつもと変わらず、「STEP-3で編集するエリア」だけ変更すればOKです。

※ `loadJsonFile()`はこのハンズオン用課題レポジトリの中で独自定義している関数です。これを用いずに、 `file_get_contents` + `json_decode`で実装しても、もちろん問題ありません。

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_2` が呼ばれます
}

function procedure2_3(
    array   $packageMeta,
    ?string $jsonFilePath = null,
): void
{
    $jsonFilePath ??= __DIR__ . '/composer.json';

    if (!file_exists($jsonFilePath)) {
        file_put_contents($jsonFilePath, json_encode(['require' => new stdClass()]));
    }

    $rootPackage = loadJsonFile($jsonFilePath);

    /* === STEP-3 ココから === */


    /* === STEP-3 ココまで === */
}
```

今回のSTEPでは、 `procedure2_3()` の中身を完成させていきます。
第1引数 `$packageMeta`は、STEP-2で組み立てたパッケージデータが入っています。すなわち、追加されるパッケージのデータをここから取り出すことができます。
上手く読み取って、既に存在している `composer.json`のデータとマージしましょう。`$rootPackage` に追加すればOKです。

### 1. パッケージのデータを受け取って、必要なフィールドの値を抜き出してマップを組み立てる

特に解説も必要ないくらい単純なものなので、コードをご覧ください

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_2` が呼ばれます
}


function procedure2_3(
    array   $packageMeta,
    ?string $jsonFilePath = null,
): void
{
    $jsonFilePath ??= __DIR__ . '/composer.json';

    if (!file_exists($jsonFilePath)) {
        file_put_contents($jsonFilePath, json_encode(['require' => new stdClass()]));
    }

    $rootPackage = loadJsonFile($jsonFilePath);

    /* === STEP-3 ココから === */

  
    // ココから追記
    $packageName = $packageMeta['name'];
    $rootPackage['require'][$packageName] = $packageMeta['version'];
    // ココまで追記
  
  
      /* === STEP-3 ココまで === */
}
```

### 2. composer.json` として書き込む(既存ファイルがあれば更新する)

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_2` が呼ばれます
}


function procedure2_3(
    array   $packageMeta,
    ?string $jsonFilePath = null,
): void
{
    $jsonFilePath ??= __DIR__ . '/composer.json';

    if (!file_exists($jsonFilePath)) {
        file_put_contents($jsonFilePath, json_encode(['require' => new stdClass()]));
    }

    $rootPackage = loadJsonFile($jsonFilePath);

    /* === STEP-3 ココから === */
    $packageName = $packageMeta['name'];
    $rootPackage['require'][$packageName] = $packageMeta['version'];

    // ココから追記  
    file_put_contents($jsonFilePath, json_encode($rootPackage, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));
    // ココまで追記


    /* === STEP-3 ココまで === */
}
```



---

STEP-3はコレで完了です。

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

![](/images/3_work2_2_step3/finish.gif)
