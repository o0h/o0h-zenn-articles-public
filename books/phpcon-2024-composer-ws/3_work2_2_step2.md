---
title: "├ W-②: STEP-2 利用バージョンの決定とパッケージ情報の構築"
---

# STEP-2 利用バージョンの決定とパッケージ情報の構築

## A. 課題について

### 背景

#### minifyされたメタデータを解釈する

STEP-1で取得したパッケージのメタ情報は、少し特殊な構造になっています。これをうまく組み立てると、晴れて「依存パッケージの正式な `composer.json`」が入手できるのです！

ということで、このSTEPではパッケージのメタ情報を読み解いていきましょう。

まず注目すべきは `packages`フィールドです。ここは、マップ構造になります。キーはパッケージ名になります。[^json-cut]

[^json-cut]: ※ 見やすさを考え、一定の階層以下の内容は省略しています。実際には空配列でや空オブジェクトはなく、データが入っています。

```json
{
  "packages": {
        "monolog/monolog": []
  }
}
```

このAPI自体が「特定のパッケージに関するメタ情報」なので、入ってくるキーは1つしか存在しない前提で良いでしょう。
マップのバリュー部分は、オブジェクトのリストです。1つ1つのオブジェクトが、バージョンごとの定義情報を持ったものになります。 `version` フィールドに注目してみてください。[^json-cut]
    

```json
"packages": {
"monolog/monolog": [
  {
    "name": "monolog/monolog",
    "description": "Sends your logs to files, sockets, inboxes, databases and various web services",
    "keywords": [],
    "homepage": "https://github.com/Seldaek/monolog",
    "version": "3.8.1",
    "version_normalized": "3.8.1.0",
    "license": [],
    "authors": [],
    "source": {},
    "dist": {},
    "type": "library",
    "support": {},
    "funding": [],
    "time": "2024-12-05T17:15:07+00:00",
    "autoload": {},
    "extra": {},
    "require": {},
    "require-dev": {},
    "suggest": {},
    "provide": {}
  },
  {
    "version": "3.8.0",
    "version_normalized": "3.8.0.0",
    "source": {},
    "dist": {},
    "support": {},
    "time": "2024-11-12T13:57:08+00:00"
  },
  {
    "version": "3.7.0",
    "version_normalized": "3.7.0.0",
    "source": {},
    "dist": {},
    "support": {},
    "time": "2024-06-28T09:40:51+00:00",
    "require-dev": {}
  },
 // 以下、たくさん続く
```

さて、特徴的なのはここからです！

バージョンごとの定義情報は、備えているフィールドが異なる場合があります。
上記の例だと、0番目に出てきたバージョン(`3.8.1`)には `name` `description` `keywords`などのフィールドがありますが、1番目以降(`3.8.0`, `3.7.0`)のバージョンには見当たりません。逆に、 1番目(`3.8.0`)にはなかった `require-dev`が、2番目(`3.7.0`)には再び存在しています。

これは、「リストを先頭から見ていって、前のデータと差分がなければフィールドを省略する」という仕組みからくるものです。(一部のフィールドを除きます)
パッケージのメタ情報が小さくなることは、ネットワークをまたいだ転送のコストを初めとしたメリットがあります。_mifify_されたデータをやり取りしている形です。

ハンズオンのスコープ外ではありますが、この処理を実現しているのは `composer/metadata-minifier` というパッケージです。気になる方は、ソースコードを覗いてみるときっと楽しいですよ。

https://packagist.org/packages/composer/metadata-minifier

Toyposerの実装にあたっては、この仕様に則った「復元」処理を考える必要があります。これが、このステップの課題の1つです。

#### 都合にあったバージョンを見つける

もう1つの課題は、「どのバージョンを使えばいいか」の取捨選択を行うことです。

パッケージのメタデータに含まれているのは、「(GitHubで)タグを付けられたバージョン」になります。裏を返すと、「安定版」「ベータ版」など様々な安定性のものがフラットに入って来るのです。

今回は、実際のComposerとの差分として「安定版だけを使う、最新版だけを使う」ことにしましょう。そのためには、バージョンごとのメタ情報を検査して「安定版であること」を確認する必要があります。

Packagistから配布されるバージョンには、次のリストに上げるシンタックスがあります(例)

**versionフィールドに入る値の有効な形式**

- 0.2.5
- v2.0.4 (prefix "v")
- 1.0.0-dev (suffix)
- 3.0.0-patch (suffix)
- 3.0.0-RC2 (suffix with number)

suffixが alpha, beta, rc, dev のいずれかの場合は、「安定版(stable)ではない」と判断することにします。それ以外のsuffix(`patch`など)は、利用可能です。



### 課題

|                | コンテナ内のパス                   |
| -------------- | ---------------------------------- |
| 作業用ファイル | `/opt/work/2_require/step2.php`    |
| 実装サンプル   | `/opt/example/2_require/step2.php` |

(実装の都合で、順番を入れ替えています)

1. STEP-1で取得したパッケージのメタデータから、「最新・安定版」のバージョンを見つける
2. minifyされているデータを復元して、正当なパッケージ定義データとする

### 参考: 関連の深いComposerのファイル・コード

* https://github.com/composer/metadata-minifier/blob/1.0.0/src/MetadataMinifier.php#L22-L46

### 参考: このステップの実装で役に立ちそうな関数etc

* https://php.net/ja/str_contains
* https://php.net/ja/current

## B. 実装例とコードの解説

作業前のファイルの全景

```php
$requirePackageNameList = ['aura/cli', 'psr/log'];

foreach ($requirePackageNameList as $requirePackageName) {
    processRequirePackage($requirePackageName); // この内部で `procedure2_2` が呼ばれます
}

function procedure2_2(string $packageVersionedMetaList): array
{
    /* === STEP-2 ココから === */



    return []; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してください
    /* === STEP-2 ココまで === */
}
```

### 1. STEP-1で取得したパッケージのメタデータから、「最新・安定版」のバージョンを見つける

まずは、バージョンごとのメタ情報をループする中で「このバージョンは安定版か」の判定をする処理を実装していきます。

なお、「最新版であること」の要件については、リストの「先に出てきたもの(インデックスが若いもの)」を採用する方法で済ませたいと思います。

見方を変えると、これは「バージョンはいくつでも良くて、安定版かどうかだけを気にする。すなわち、version nameの後半部分(suffix)だけ気にしておけば良い」と言えます。このコード例では、`explode`を使ってsuffixを取り出しています。「不安定」を示すキーワードがsuffixから見つかったら、そのバージョンは利用できません。ループ文を`continue`します。

逆に、利用可能な「安定版」が見つかったら、その後のバージョンを参照する必要がなくなります。`break`してループを抜けましょう。

```php
function procedure2_2(string $packageVersionedMetaList): array
{
    /* === STEP-2 ココから === */

  
    // ココから追記・変更
    $packageVersionedMetaList = json_decode($packageVersionedMetaList, true);
    $packageVersionedMetaPackages = current($packageVersionedMetaList['packages']);

    foreach ($packageVersionedMetaPackages as $packageVersionedMetaPackage) {
        $versionNormalized = $packageVersionedMetaPackage['version'];
        if (str_contains($versionNormalized, '-')) {
            $suffix = explode('-', $versionNormalized)[1];
            if (preg_match('#('.implode('|', ['dev', 'alpha', 'beta', 'rc']).')#i', $suffix)) {
                continue;
            }
        }
        break;
    }

    return []; // 型宣言を満たすための仮置きのreturn。 @TODO 実装が完了したら削除してください
    // ココまで追記・変更


      /* === STEP-2 ココまで === */
}
```



### 2. minifyされているデータを復元して、正当なパッケージ定義データとする

minifyの復元は、容量さえ掴んでしまえば非常に単純なものです。「先頭にあるバージョンからループして見ていく」中で、「出会ったバージョンをメモして、次に出てき他バージョンをマージする」ことで実現できます。

コード中に3箇所ある「ココを追記」の部分を、書き写してください。

```php
function procedure2_2(string $packageVersionedMetaList): array
{
    /* === STEP-2 ココから === */
    $packageVersionedMetaList = json_decode($packageVersionedMetaList, true);
    $packageVersionedMetaPackages = current($packageVersionedMetaList['packages']);

    // ココを追記①
    $packageData = [];

      foreach ($packageVersionedMetaPackages as $packageVersionedMetaPackage) {
        // ココを追記②
        $packageData = array_merge($packageData, $packageVersionedMetaPackage);

        $versionNormalized = $packageVersionedMetaPackage['version'];
        if (str_contains($versionNormalized, '-')) {
            $suffix = explode('-', $versionNormalized)[1];
            if (preg_match('#('.implode('|', ['dev', 'alpha', 'beta', 'rc']).')#i', $suffix)) {
                continue;
            }
        }
        break;
    }

    // ココを変更③
    return $packageData;
    /* === STEP-2 ココまで === */
}
```



---

STEP-2はコレで完了です。

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

『✅ Step2が完了しました！』と出力されていれば、成功です。自信を持って、次のステップに進みましょう！

![](/images/3_work2_2_step2/finish.gif)
