---
title: "├ W-①: STEP-3 依存パッケージの取得"
---

# STEP-3 依存パッケージの取得

## A. 課題について

### 背景

ここから少しずつ、Composer”ならでは”の話題に入っていきます！

`.lock` ファイルからパッケージ情報を読み取って、その中身を1件ずつ処理していく土台が整いました。次は、実際に各パッケージの中身を取り寄せる処理を書いていきます。
(このハンズオンのスコープでは、「GitHubからファイルを取得する」「取得するファイルはZip形式」と決め打ちにしています。)

`.lock` から読み取った各パッケージの情報(= `$package`とします)には、パッケージ本体の取得元の情報が含まれています。それが``$package['dist'] `です。

**distの例(guzzlehttp/guzzle:7.9.2)**

```json
"dist": {
    "type": "zip",
    "url": "https://api.github.com/repos/guzzle/guzzle/zipball/d281ed313b989f213357e3be1a179f02196ac99b",
    "reference": "d281ed313b989f213357e3be1a179f02196ac99b",
    "shasum": ""
},
```

名前の通り、`url`フィールドにパッケージ本体(のアーカイブ)が手に入るURLが格納されています。

ここをみると、ホストが `api.github.com`になっています。GitHubのAPI経由でファイルをダウンロードするということですね。具体的には、『Download a repository archive (zip)』のAPIを利用します。

https://docs.github.com/en/rest/repos/contents?apiVersion=2022-11-28#download-a-repository-archive-zip

そのため、単純なHTTPリクエストを送るだけでは作業が完了できません。GitHub APIを利用するにあたっての要件を満たす必要があります。それが、次の2点です。

1. BearerトークンをAuthorizationフィールドにセットする
2. 何かしらのUser-Agentをセットする

ただし、「GitHubの使い方」については本ハンズオン内で中心的に扱いたいトピックではありません。そのため、予め`downloadWithGitHubAuth` という関数を用意してあります。

要点だけを掻い摘むと、次のような実装です。

```php
  $options = [
      'http' => [
          'header' => [
              'Authorization: Bearer ' . getenv('GITHUB_OAUTH_TOKEN'),
              'User-Agent: MyFirstComposer/0.3',
          ],
          'follow_location' => 1,
      ],
  ];
  $context = stream_context_create($options);
  $dist = @file_get_contents($url, context: $context);
```

詳細は `/opt/work/helper.php`を覗いてみてください。

これらの知識とヘルパーを利用して、実装に取り組んでいきましょう。
STEP-3では **パッケージ情報を使って、パッケージ本体の取得元からアーカイブファイルを取得する** ことがゴールとなります。

:::message alert

リクエスト処理を自作する場合は、リクエスト先がGitHubAPIであることを必ず確認してください。  
確認を怠った場合、意図しないサーバーにGitHubトークンが渡ってしまう危険性があります。

:::

### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/2_install/step3.php`    |
| 実装サンプル   | `/opt/example/2_install/step3.php` |

1. パッケージ情報から、ソースコード本体(アーカイブファイル)を入手するためのURLを取得する
2. 取得したURLからアーカイブファイルをダウンロードして、指定したパスに保存する
   - DLしたファイルは、 `/tmp/work1/` 以下に保存する
   - ファイル名は、パッケージ名＋拡張子とする
     - (例) `monolog/monolog` の場合、 `/tmp/work1/monolog.zip` というパスになる
     - **パッケージ名に `/` を含む(サブディレクトリが必要)ことに注意**
3. ファイルを保存した先のフルパスを、関数の `return` として返す

### 参考: このステップの実装で役に立ちそうな関数etc

ディレクトリの作成や存在確認には、次の関数が利用できます。

1. https://php.net/ja/file_exists
2. https://php.net/ja/is_dir
3. https://php.net/ja/mkdir

ファイルの書き出しには次の関数やクラスが利用できます。

1. https://php.net/ja/file-put-contents
2. https://php.net/ja/function.fwrite.php
3. https://php.net/ja/class.splfileobject.php

その他

1. https://php.net/ja/dirname
2. https://php.net/ja/is_dir

### 参考: 関連の深いComposerのファイル・コード

- https://github.com/composer/composer/blob/2.8.4/src/Composer/Package/Locker.php
- https://github.com/composer/composer/blob/2.8.3/src/Composer/Util/GitHub.php
-

## B. 実装例とコードの解説

作業前のファイルの全景

```php
<?php
require __DIR__ . '/step2.php';

function procedure1_3(array $package): string
{
    /* === STEP-3 ココから === */


    return ''; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してくだし
    /* === STEP-3 ココまで === */
}
```

今回は、全体処理(`make work1`)の中から「呼び出される側」の関数を完成させていくことになります。

### 1. パッケージ情報から、ソースコード本体(アーカイブファイル)を入手するためのURLを取得する

`$lockData`の`packages` ないし `packages-dev`から取り出した、パッケージ情報が引数として渡ってきます。

先述の通り、ここから`['dist']['url']` を抜き出せばOKです。

```php
function procedure1_3(array $package): string
{
    /* === STEP-3 ココから === */
    $url = $package['dist']['url'];
      echo "\tDL: {$url}\n"; // 要件にはないが、動作確認のためにechoさせる

    return ''; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してくだし
    /* === STEP-3 ココまで === */
}
```

### 2. 取得したURLからアーカイブファイルをダウンロードして、指定したパスに保存する

取り出したURLに、GETリクエストを送ってファイルを取得しましょう。これには `downloadWithGitHubAuth()`関数を使います。(①)

取得したファイルの保存先が `/tmp/work1/{パッケージ名}.zip` です。
ただし、パッケージ名は「ベンダー名」＋「/(スラッシュ)」＋「プロジェクト名」となるのが慣習です。そのため、ディレクトリを作成しておく必要があります。(②)
この時、「同一ベンダーから複数のパッケージを取得している」ケースの考慮が漏れがちなので、注意してくださいね。

保存先が確保できたら、ファイルを書き込みます(③)

```php
function procedure1_3(array $package): string
{
    /* === STEP-3 ココから === */
    $url = $package['dist']['url'];
      echo "\tDL: {$url}\n"; // 要件にはないが、動作確認のためにechoさせる

      // ココから追記
      // ①
      $packageContent = downloadWithGitHubAuth($package['dist']['url']);
    // ②
    $downloadTo = "/tmp/work1/{$package['name']}.zip";
    $downloadDir = dirname($downloadTo); // フルパスから1つ上のディレクトリ名までを取得
    if (!is_dir($downloadDir)) { // 与えられたパスがディレクトリではなかったら
        mkdir($downloadDir, 0777, true);
    }
    // ③
    file_put_contents($downloadTo, $packageContent);
      // 追記ココまで

    return ''; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してくだし
    /* === STEP-3 ココまで === */
}
```

### 3. ファイルを保存した先のフルパスを、関数の `return` として返す

仕上げに、関数が返す値を調整しましょう。

```php
function procedure1_3(array $package): string
{
    /* === STEP-3 ココから === */
    $url = $package['dist']['url'];
    echo "\tDL: {$url}\n"; // 要件にはないが、動作確認のためにechoさせる

    $packageContent = downloadWithGitHubAuth($package['dist']['url']);
    $downloadTo = "/tmp/work1/{$package['name']}.zip";
    $downloadDir = dirname($downloadTo); // フルパスから1つ上のディレクトリ名までを取得
    if (!is_dir($downloadDir)) { // 与えられたパスがディレクトリではなかったら
        mkdir($downloadDir, 0777, true);
    }
    file_put_contents($downloadTo, $packageContent);

    // 変更: ダミーで入れていたreturnを書き換える
    return $downloadTo;
    /* === STEP-3 ココまで === */
}
```

---

STEP-3はコレで完了です。

終わったら、課題の内容をクリアする実装ができているかを確認しましょう。
用意されている`make` コマンドで検査できます。コンテナ内ではなく、コンテナの外側（ホストマシン）で実行することに注意してください。

**ホストから**

```
make work1
```

もし`make`が使えないなど、コンテナ内で行う場合には下記のコマンドを利用してください。

**コンテナ内で**

```sh
php /opt/work/X_install/main.php
```

『✅ Step3が完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/2_work1_2/3/step3-finish.gif)
