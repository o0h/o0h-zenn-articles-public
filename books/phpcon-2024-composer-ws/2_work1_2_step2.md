---
title: "├ W-①: STEP-2 依存パッケージ取得の基礎枠作り"
---

# STEP-2 依存パッケージ取得の基礎枠作り

## A. 課題について

### 背景

パッケージごとの処理を行っていくための土台を作りましょう。

`composer.lock`の中にある、パッケージ情報を読み取れるようにする必要があります。
`packages` `packages-dev` というフィールドがあるので、これらを取り出していきます。名前の通り`packages-dev`は `composer require --dev`でインストールされたパッケージ群です。
(今回のハンズオンではdevを区別しません。そのため、これら2つのフィールドを同じように扱ってください)

STEP-1で作った`$lockData` の中にあるパッケージ情報のフィールド2つは、それぞれパッケージの情報が格納された添字配列になっています。
JSON風のシンタックスで表すと、次のようなイメージです。

**構造のイメージ**

```
{
    "packages": [
        {@package_a},
        {@package_b},
        {@package_c}
    ],
    "packages-dev": [
        {@package_d},
        {@package_e},
        {@package_f}
    ]
}
```

具体的な中身は、次のSTEPで触れていきます。

ここでは、パッケージごとの処理を行える状態を目指しましょう。

### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/1_install/step2.php`    |
| 実装サンプル   | `/opt/example/1_install/step2.php` |

ここでは、先ほどSTEP1で作った内容がそのまま引き継がれているので、ローカル変数`$lockData` を利用できます。

1. `$lockData` から、パッケージの情報を取り出してください
2. 取り出した情報に対して、`processPackage()` 関数を呼び出してください
   - `processPackage()`は、単体のパッケージ情報を引数に取ります

## B. 実装例とコードの解説

作業前のファイルの全景

```php
assert(isset($lockData));

/* === STEP-2 ココから === */
// foreach ( ・・・

/* === STEP-2 ココまで === */
```

:::message

動作確認用コマンド(`make work1`)実行時に、自動的にSTEP-1の内容が引き継がれるようになっています。
そのため、「STEP-1の内容をコピー＆ペーストして持ってくる」や「requireで先のファイルを読み込むように追加する」という作業は不要です

:::

### 1.`$lockData` から、パッケージの情報を取り出してください

例によって簡易な実装で充分なので、フィールド2つ対してそれぞれforeachループを回せば充分でしょう。

```php
foreach ($lockData['packages'] as $package) {
  // TODO: $packageの処理
}

foreach ($lockData['packages-dev'] as $package) {
  // TODO: $packageの処理
}
```

### 2. 取り出した情報に対して、`processInstallPackage()` 関数を呼び出してください

ループ文の中で、指定された関数を呼びます。TODOコメントを記した部分を変更していきましょう。

```php
foreach ($lockData['packages'] as $package) {
  processInstallPackage($package);
}

foreach ($lockData['packages-dev'] as $package) {
  processInstallPackage($package);
}
```

---

STEP-2はコレで完了です。

終わったら、課題の内容をクリアする実装ができているかを確認しましょう。
用意されている`make` コマンドで検査できます。コンテナ内ではなく、コンテナの外側（ホストマシン）で実行することに注意してください。

**ホストから**

```
make work1
```

もし`make`が使えないなど、コンテナ内で行う場合には下記のコマンドを利用してください。

**コンテナ内で**

```sh
php /opt/work/1_install/main.php
```

『✅ Step2が完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/2_work1_2/2/step2-finish.gif)
