---
title: "├ W-①: STEP-4 依存パッケージの展開・配置"
---

# STEP-4 依存パッケージの展開・配置

## A. 課題について

### 背景

ダウンロードされたファイルを、プロジェクトのPHPアプリケーションから使えるように配置しましょう。
`dist`からダウンロードした場合、通常はパッケージが圧縮された状態で配布されます。そのため、これを解凍する必要があります。

その際に「単純にアーカイブファイルを展開した」だけだとハマるポイントがあるので、注意が必要です。

まず、構造に注意しましょう。
感覚としては「パッケージを固めたアーカイブを取得してきた」ところなので、展開すると「パッケージのルートディレクトリ以下の内容が展開される」と想像するかも知れません。
しかし、実際には「パッケージ全体をラップしたディレクトリ」が取り出されます。すなわち、｀src` ディレクトリ等がそのまま出てくることはありません。一階層だけ深くなった所に、それらのファイルが存在します。
テキストで説明されてもピンと来ないかも知れないので、GUI上で実際に展開している様子のキャプチャしたものをご覧ください。

![](/images/2_work1_2/4/zipball.gif)
*パッケージを展開すると出てくるのは・・・*

また、その際に取り出されるディレクトリの名前も「そのままパッケージ名」にはっていません。プロジェクトのアプリケーションから使えるようにするには、適切な命名に揃えたいところです。

以上の点を踏まえて、STEP-4に取り組んでいきましょう！

### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/1_install/step4.php`    |
| 実装サンプル   | `/opt/example/1_install/step4.php` |

1. STEP-3で保存したアーカイブファイルを、ZipArchiveオブジェクトとして読み込む
3. アーカイブを展開し、 `/vendor/{$package}`ディレクトリに配置する`

### 注意

macOS上でDockerを利用している場合、PHPの `rename` コマンドは条件によって動作しない可能性があります。ファイルシステムが異なるパスへの移動は、その1つです。

実際、[Composerのソースコード](https://github.com/composer/composer/blob/2.8.3/src/Composer/Util/Filesystem.php#L441C1-L443C87)を読んでみても、こんな記述があります。

```php
// We do not use PHP's "rename" function here since it does not support
// the case where $source, and $target are located on different partitions.
$result = $this->getProcess()->execute(['mv', $source, $target], $output);
```

そのため、今回は課題レポジトリ上でLinuxの `mv` コマンドを実行する関数を用意しています。定義は `/opt/work/helper.php`にあります。

```php
mv($from, $to);
```

のように使ってください。

### 参考: このステップの実装で役に立ちそうな関数etc

- https://php.net/ja/ziparchive
- https://php.net/ja/exec
- https://php.net/ja/proc_open
- https://php.net/ja/glob

### 参考: 関連の深いComposerのファイル・コード

* https://github.com/composer/composer/blob/2.8.3/src/Composer/Downloader/ArchiveDownloader.php
* https://github.com/composer/composer/blob/2.8.3/src/Composer/Util/Filesystem.php

## B. 実装例とコードの解説

作業前のファイルの全景

```php
$archivePaths = glob('/tmp/work1/*/*.zip');
foreach ($archivePaths as $archivePath) {
    /* === STEP-4 ココから === */
    /* === STEP-4 ココまで === */
}
```

### 1. STEP-3で保存したアーカイブファイルを、ZipArchiveオブジェクトとして読み込む

課題ファイルに、予め「対象となるZIPファイルをリストアップして、1件ずつ処理をしていく」という記述は行われています。
渡ってくるのは、ZIPファイルの配置されているフルパスです。これを `ZipArchive`オブジェクトを用いて処理していきましょう。

```php
$archivePaths = glob('/tmp/work1/*/*.zip');
foreach ($archivePaths as $archivePath) {
    /* === STEP-4 ココから === */
  
    // 追記ココから
    $zip = new ZipArchive();
    if ($zip->open($archivePath) !== true) {
        throw new RuntimeException('アーカイブファイルのオープンに失敗しました');
    }
    $zip->close();
     // 追記ココまで
    /* === STEP-4 ココまで === */
}
```

### 2. アーカイブを展開し、 `/vendor/{$package}`ディレクトリに配置する`

読み込んだZIPを展開していきます。そのために、展開先ディレクトリ名も決定していきましょう。

`ZipArchive::extractTo()`メソッドでファイルを展開できます。(①)

STEP-3では「パッケージ名.zip」というファイル名で保存していたので、今回はこれを信頼して名前を取り出します。そのため、「フルパスから、ベースとなるディレクトリ部分と、ファイルの拡張子部分を除去する」というアプローチで実現できます。(②)

先述の理由で、「アーカイブを展開した後に、目的のファイルを取り出す」という手間が必要になります。これは、課題レポジトリ内で独自に用意してある関数 `mv()` を利用してしまいましょう。(③)
` $zip->getNameIndex(0)` を使うと、「アーカイブファイルの中の1番最初のファイル(ディレクトリ)」名を取り出すことができます。 

例えば、 `guzzlehttp/guzzle` のZIPファイルであれば、含まれているディレクトリ(1番外側のディレクトリ)は `guzzle-guzzle-d281ed3/`となります。試しにこんなコードを見てみましょう。

```php
$zip = new ZipArchive();
$zip->open($archivePath);
echo '読み込んだファイル: ' . $archivePath . PHP_EOL;
echo 'getNameIndex(0): ' . $zip->getNameIndex(0) . PHP_EOL;
```

出力結果は次のとおりです。

```
読み込んだファイル: /tmp/work1/guzzlehttp/guzzle.zip
getNameIndex(0) :guzzle-guzzle-d281ed3/
```



では、課題の実装に入っていきます。

```php
$archivePaths = glob('/tmp/work1/*/*.zip');
foreach ($archivePaths as $archivePath) {
    /* === STEP-4 ココから === */
    $zip = new ZipArchive();
    if ($zip->open($archivePath) !== true) {
        throw new RuntimeException('アーカイブファイルのオープンに失敗しました');
    }

    // 追記ココから
    // ①
    $zip->extractTo('/tmp/work1/');

    // 再び、追記ココから
    // ②
    $packageName = substr($archivePath , strlen('/tmp/work1/'), -4);
    $installTo = __DIR__ . '/vendor/' . $packageName;
    if (!is_dir(dirname($installTo))) {
        mkdir(dirname($installTo), 0777, true);
    }
  
    // ③
    mv('/tmp/work1/' . $zip->getNameIndex(0), $installTo .'/');
  
    // 追記ココまで
    $zip->close();
    /* === STEP-4 ココまで === */
}
```



---

STEP-4はコレで完了です。

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

『✅ StepXが完了しました！』と出力されていれば、成功です。
更に、ワーク全体の工程を完了したので『💯 WORK COMPLETE 💯』とも表示されます。

![](/images/2_work1_2/4/step4-finish.gif)



## 🎉 Tinyposerの”install"を味わう

さて、今回の目標である「コマンドを作ってみる」シリーズの第1弾がこれで完了した訳です。
果たして仕上がりはいかがでしょうか？

作業ファイルのある `work/1_install` 以下に作成した`vendor`ディレクトリとその中身が、現在の我々の作ろうとしていたものです。その中身を覗いてみてください！なんだか見覚えのありそうな構成で、ファイルが並んでいませんか？

![](/images/2_work1_2/4/vendor-dekitane.gif)
*My First Tinyposer -install-*

この調子で。次に行ってみましょう！

## ちょっとコラム

Zip Bombと言われる攻撃手法があります。こういうのを見ると、「危険性があるものを安易に扱わないようにしないとな」と意識が高まりますよね。

https://ja.wikipedia.org/wiki/%E9%AB%98%E5%9C%A7%E7%B8%AE%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E7%88%86%E5%BC%BE

Composerでこの問題に対処しているコードがあります。こういった工夫を知れるのも、「中身をじっくり読んでみた人の特典」なのかも知れませんね

https://github.com/composer/composer/blob/2.8.3/src/Composer/Downloader/ZipDownloader.php#L224-L238
