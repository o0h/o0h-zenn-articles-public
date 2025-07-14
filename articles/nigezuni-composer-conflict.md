---
title: "今こそ徹底入門！Composerのコンフリクト"
emoji: "🤯"
type: "tech"
topics: ["composer", "パッケージ管理"]
published: true
---

```sh
$ composer require --dev -W "phpunit/phpunit:^12.2.7" "phpunit/php-code-co
verage:^12.3.1"

Your requirements could not be resolved to an installable set of packages.

  Problem 1
    - Root composer.json requires phpunit/php-code-coverage ^12.3.1 -> satisfiable by phpunit/php-code-coverage[12.3.1, 12.3.x-dev].
    - phpunit/php-code-coverage[12.3.1, ..., 12.3.x-dev] require nikic/php-parser ^5.4.0 -> found nikic/php-parser[v5.4.0, v5.5.0] but these were not loaded, likely because it conflicts with another require.
  Problem 2
    - Root composer.json requires phpunit/phpunit ^12.2.7 -> satisfiable by phpunit/phpunit[12.2.7, 12.2.x-dev, 12.3.x-dev].
    - phpunit/php-code-coverage[12.3.1, ..., 12.3.x-dev] require nikic/php-parser ^5.4.0 -> found nikic/php-parser[v5.4.0, v5.5.0] but these were not loaded, likely because it conflicts with another require.
    - phpunit/phpunit[12.2.7, ..., 12.3.x-dev] require phpunit/php-code-coverage ^12.3.1 -> satisfiable by phpunit/php-code-coverage[12.3.1, 12.3.x-dev].
```

これです！！

## はじめに
PHPアプリケーションを作成していて、Composerを利用していると時折「コンフリクト！！！！」が発生します。  
私はこれがとても苦手で、「なんか情報が沢山あるし、解決も面倒くさそうだし、嫌だなぁ」と思う日々でした。  

最近になって、いよいよ「苦手で分からない！だけでは良くないな」と感じるようになりました。  
この情報を読み解いて、「怖くない」を目指そうというのが本記事の狙いです。

## 蛇足
Composerがコンフリクトを起こすのは、そもそもどんな目的で・何の作業をしている際のエラーなの？を知る必要がありますよね。  

「いい感じなパッケージを選択し、最終決定する(整合性をとる)」作業の流れについて、先日のPHP Conferenceの発表で触れました。よろしければ御覧ください。
https://speakerdeck.com/o0h/phpcon-2025?slide=62

実際、この記事を書こうと思ったのは  
「あんなに内部実装について深堀りしたんだから、昔からずっと苦手だったコンフリクト問題も、今なら分かるのではないか！？」  
という背景もあります。

資料内でも触れている「Rule」や「DependencySolverの問題解決手順」について知ると、なんとなくコンフリクト時のメッセージが読みやすく感じられるはずです。

## 前提の理解・「パッケージを決定する」とは何か？
※ 「Packageとは何か？」については、例えば公式ドキュメントの[Libraries \- Composer](https://getcomposer.org/doc/02-libraries.md#every-project-is-a-package)を参照してください

Composerにおいて、利用するパッケージを変更する作用を持つのは `require` `update` です。

* require:
    * 要求されたパッケージを導入する
* update:
    * 既に `require(-dev)` にあるパッケージについて、制約を満たす範囲でバージョンアップする

言い換えると、この2コマンドが `composer.lock` に記述されるパッケージリストを更新することになります[^update-lockfile]。  
正にこの 「`composer.lock`に記述されるパッケージリスト」こそ、(実装レベル・アーティファクトの観点で言えば)「パッケージを決定する」ことに他なりません。
重要なのは、 `composer.json` と `composer.lock` の責務の違いです。

* composer.json:
    * パッケージ(プロジェクトでもライブラリでも)が **直接要求するパッケージ** を扱う
    * 依存(要求)パッケージの **バージョンの有効範囲** を指定する
* composer.lock
    * パッケージとその依存パッケージを含めた **全ての直接・間接要求されたパッケージ** を扱う
    * 依存パッケージの **具体的なバージョン番号(semverやコミットハッシュなど)** を指定する

`composer.json` で管理しているのは抽象的な内容であり、それを現実レベルで解決した具象的な内容を管理するのが `composer.lock` という関係になります。

例を出すと、

1. `aaa/hoge`, `bbb/fuga`というパッケージを直接要求していて
2. それぞれ、 `^2.5.0`, `^3.2.0` の範囲を指定していて
    *  `^3.2.0` の場合、有効な範囲は `>=3.2.0 <4.0.0-0` になる
3. かつ、いずれかもしくは両方のパッケージが `aaa/piyo` `aaa/hogera` を(間接的に)依存していて、また同様に何らかの有効範囲もしていて・・・

となった場合、 `composer.json` と `composer.lock` のパッケージリストの例は次のようになります。
![](/images/nigezuni-composer-conflict/2.png)

しばらく経った後に `composer update` を実行したら、`composer.lock` にあるパッケージの指定バージョンが変わるかも知れません。


このように、 **ルートレベルにある要求(=`composer.json`の`require(-dev)`)を満たして、具体的なバージョンも込で全パッケージを選択すること** が、「パッケージを決定する」という作業のゴールです。  
この時に、「ルートと、依存先パッケージ」「間接パッケージ同士」の要求範囲が矛盾し、整合性を保てなくなると **コンフリクトが発生している** 状態になります。


[^update-lockfile]: 厳密に言えば、パッケージの変更を伴わなてくも `composer.lock` ファイルが更新されるケースはあります。今回は扱いません。


## 実例とともにコンフリクトを紐解く
いくつかの例を見ていきましょう

:::message
なお、ここからは `composer.json` のレベルで定義されるパッケージ(すなわち、いま作業しているプロジェクトです)を **rootパッケージ** 、そこで(直接的に)要求されている依存パッケージを **(ユーザーの)request** と呼ぶこととします
:::

### 1. requestと間接依存のコンフリクト
最初は、単純と言えそうな例です。

```sh
 $ cat composer.json
{
    "require": {
        "symfony/console": "^7.3"
    }
}

```

```sh
$ composer require psr/container:~1.0.0 -W
./composer.json has been updated
Running composer update psr/container --with-all-dependencies
Loading composer repositories with package information
Updating dependencies
Your requirements could not be resolved to an installable set of packages.

  Problem 1
    - Root composer.json requires psr/container ~1.0.0, found psr/container[1.0.0] but these were not loaded, likely because it conflicts with another require.
  Problem 2
    - symfony/console is locked to version v7.3.1 and an update of this package was not requested.
    - symfony/console v7.3.1 requires symfony/service-contracts ^2.5|^3 -> satisfiable by symfony/service-contracts[v3.6.0].
    - symfony/service-contracts v3.6.0 requires psr/container ^1.1|^2.0 -> found psr/container[1.1.0, 1.1.1, 1.1.2, 2.0.0, 2.0.1, 2.0.2] but it conflicts with your root composer.json require (~1.0.0).
```

まず、 _Problem_ が2つ報告されています。  
ここで言う問題とは、突き詰めれば **but these were not loaded** の部分です。  
「何かしらのパッケージが、(直接/間接)的な要求通りに配置できなかった」ことを報告しているのです。パッケージ間の整合性を取ることを責務としているComposerが、要求に答えるために解かなければいけない処理に回答ができなかった、だからこそ「Problem」だと言えます。
 
このProblemの「番号」は、大体においては(rootの)requestに近い問題が若い方に来ることが多いです。ただし、これは仕様ではなく「単に都合の問題」と捉えてください。計算量の最適化の結果によっては、入れ替わる可能性が大いにありえます。

では、内容を見てみましょう
#### Problem 1
```
Root composer.json requires psr/container ~1.0.0, found psr/container[1.0.0] but these were not loaded, likely because it conflicts with another require.
```

`require` コマンドで `psr/container ~1.0.0` を要求したので、これはrootの依存(request)として処理されるべきものです。そのため、`Root composer.json requires` と表記されます。  
バージョン指定は `~1.0.0` となっています。`>=1.0.0 <1.1.0` が、要求を満たす範囲です。
:::message
`~`と`^`の違いに注意。
semverの指定に使える記法を確認する時に、筆者は https://jubianchi.github.io/semver-check/ をよく使っています。
:::

どうやら、 `likely because it conflicts with another require` ということで、`psr/container`について他の要求との整合性が取れなかったみたいです。

まとめると、Problem1は「requestにあるpsr/container(の~1.0.0)が満たせない」のだということがわかりました。

#### Problem 2

```
- symfony/console is locked to version v7.3.1 and an update of this package was not requested.
- symfony/console v7.3.1 requires symfony/service-contracts ^2.5|^3 -> satisfiable by symfony/service-contracts[v3.6.0].
- symfony/service-contracts v3.6.0 requires psr/container ^1.1|^2.0 -> found psr/container[1.1.0, 1.1.1, 1.1.2, 2.0.0, 2.0.1, 2.0.2] but it conflicts with your root composer.json require (~1.0.0).
```

今度は最後の行が「問題そのもの」なのですが、それを遡って2行目→1行目と「なぜその問題が生じ地ているのか」というバックトレースとなります。

* (いま `.lock` ファイルで指定されている) `symfony/service-contracts v3.6.0` が、 `psr/container`の `^1.1|^2.0` を要求している
  * packagistからの情報も踏まえて、これを満たし得る候補バージョンは `[1.1.0, 1.1.1, 1.1.2, 2.0.0, 2.0.1, 2.0.2]` となる
* そうすると、 `composer.json` が要求している、 `psr/container` の `~1.0.0` と整合性が取れない

これが今回のコンフリクトを引き起こしている問題です。  
なぜ、そうなっているのか？・・・すなわち、どこをいじればコンフリクトが解消できるのか？のヒントを、上の行へと遡ることで見つけていきます。

* (いま `.lock` ファイルで指定されている) `symfony/console v7.3.1` は、 `symfony/service-contracts ^2.5|^3` を要求している
  * psr/containerの場合は「requireコマンドで指定されたパッケージだった」ことから、候補となる複数のバージョンを検証した
  * が、 `symfony/service-contracts` は要求の変化や追加がないので、 `.lock` にあるバージョンで確定。よって、「この要求は、 `v3.6.0` で満足している」と判断

更に遡って、1行目の内容を確認します

* `symfony/console` はロックされている(`.lock` ファイルで、既に具体的なバージョンの指定を受けている)パッケージで、その対象バージョンは`7.3.1`
* かつ、このパッケージの変更(`update`)は、今は要求されていない
  * Composerが勝手に変更するわけにいかない

結果的に、「要求を全て丸く収めるための選択肢がない」、すなわちコンフリクト状態に陥っているわけです。

#### 別の調べ方
「このパッケージの、このバージョン(範囲)を入れようとして、だめだった」という状況まで判明しているなら、もっと手早く調べられます。  
`composer prohibits` もしくはエイリアスの `composer why-not` を利用する方法です。

```sh
$ composer prohibits psr/container "~1.0.0"
symfony/service-contracts v3.6.0 requires psr/container (^1.1|^2.0) 
Not finding what you were looking for? Try calling `composer update "psr/container:~1.0.0" --dry-run` to get another view on the problem.
$ composer why-not psr/container "~1.0.0"
symfony/service-contracts v3.6.0 requires psr/container (^1.1|^2.0) 
Not finding what you were looking for? Try calling `composer update "psr/container:~1.0.0" --dry-run` to get another view on the problem.
```

再帰的に解析するフラグもあります
```sh
$ composer prohibits -r psr/container "~1.0.0"
__root__                  -      requires symfony/console (^7.3)              
symfony/console           v7.3.1 requires symfony/service-contracts (^2.5|^3) 
symfony/service-contracts v3.6.0 requires psr/container (^1.1|^2.0)           
Not finding what you were looking for? Try calling `composer update "psr/container:~1.0.0" --dry-run` to get another view on the problem.
```

#### 解決
これで、「結局は `symfony/console` のバージョンが動かせないことで問題が起きていそうだ」と判明しました。  
あまり現実的ではないですが、要するに「`symfony/console` を変更して良ければ解決するかも」という手段も選択できます。


```sh
$ composer require "psr/container:~1.0.0" "symfony/console:*" -W
./composer.json has been updated
Running composer update psr/container symfony/console --with-all-dependencies
Loading composer repositories with package information
Updating dependencies
Lock file operations: 0 installs, 4 updates, 0 removals
  - Downgrading psr/container (2.0.2 => 1.0.0)
  - Downgrading symfony/console (v7.3.1 => v6.2.13)
  - Downgrading symfony/service-contracts (v3.6.0 => v2.2.0)
  - Downgrading symfony/string (v7.3.0 => v6.4.21)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 0 installs, 4 updates, 0 removals
  - Downloading symfony/string (v6.4.21)
  - Downloading psr/container (1.0.0)
  - Downloading symfony/service-contracts (v2.2.0)
  - Downloading symfony/console (v6.2.13)
  - Downgrading symfony/string (v7.3.0 => v6.4.21): Extracting archive
  - Downgrading psr/container (2.0.2 => 1.0.0): Extracting archive
  - Downgrading symfony/service-contracts (v3.6.0 => v2.2.0): Extracting archive
  - Downgrading symfony/console (v7.3.1 => v6.2.13): Extracting archive
Generating autoload files
8 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
No security vulnerability advisories found.
$ composer update --bump-after-update
Loading composer repositories with package information
Updating dependencies
Nothing to modify in lock file
Writing lock file
Installing dependencies from lock file (including require-dev)
Nothing to install, update or remove
Generating autoload files
8 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
No security vulnerability advisories found.
Bumping dependencies
./composer.json has been updated (1 changes).
$ cat composer.json 
{
    "require": {
        "symfony/console": ">=6.2.13",
        "psr/container": "~1.0.0"
    }
}
```

### 2. 間接依存同士のコンフリクト
```sh
$ cat composer.json
{
    "require-dev": {
        "psy/psysh": "^0.11.0"
    }
}
```

```sh
$ composer require --dev -W "phpunit/phpunit:^12.2.7"
./composer.json has been updated
Running composer update phpunit/phpunit --with-all-dependencies
Loading composer repositories with package information
Updating dependencies
Your requirements could not be resolved to an installable set of packages.

  Problem 1
    - Root composer.json requires phpunit/phpunit ^12.2.7 -> satisfiable by phpunit/phpunit[12.2.7].
    - phpunit/php-code-coverage 12.3.1 requires nikic/php-parser ^5.4.0 -> found nikic/php-parser[v5.4.0, v5.5.0] but these were not loaded, likely because it conflicts with another require.
    - phpunit/phpunit 12.2.7 requires phpunit/php-code-coverage ^12.3.1 -> satisfiable by phpunit/php-code-coverage[12.3.1].


Installation failed, reverting ./composer.json and ./composer.lock to their original content.
```
PHPUnitを入れようとしたら、コンフリクトが起きて駄目でした。  
今回はProblemは1つだけですね

#### Problem 1
```
- Root composer.json requires phpunit/phpunit ^12.2.7 -> satisfiable by phpunit/phpunit[12.2.7].
```

これは、(requestとして)「指定された`phpunit/phpunit` の `^12.2.7` に合致するのは、 `12.2.7` がある」ことを意味します。

```
- phpunit/php-code-coverage 12.3.1 requires nikic/php-parser ^5.4.0 -> found nikic/php-parser[v5.4.0, v5.5.0] but these were not loaded, likely because it conflicts with another require.
```
_but these were not loaded_ が出てきたので、ここが今回の問題の正体です。  
どうやら `phpunit/php-code-coverage:12.3.1` は、 `nikic/php-parser:^5.4.0` を要求している様子(それを満たすのは、`v5.4.0`か `v5.5.0`。このPJに入っている他の依存パッケージは全て、このいずれかを利用する必要がある)です。
更に次の行を読んでいきましょう。


```
- phpunit/phpunit 12.2.7 requires phpunit/php-code-coverage ^12.3.1 -> satisfiable by phpunit/php-code-coverage[12.3.1].
```
その `phpunit/php-code-coverage:12.3.1` はどこからきているのか、がこの行に表れています。  
PHPUnitのバージョン(=`12.2.7`)が、この先の問題を引き起こしていました。


ただし、これだけだと「なぜ `nikic/php-parser:^5.4.0` を満たせないのか？」の正確な情報がありません。深く追跡するには、 `prohibits` コマンドを利用する必要があります。
```
$ composer why-not  nikic/php-parser "^5.4.0"
psy/psysh v0.11.22 requires nikic/php-parser (^4.0 || ^3.1) 
```

`psy/psysh` の、現行バージョンとの相性が悪いみたいですね。

構造化してまとめると、次のようになります。

![](/images/nigezuni-composer-conflict/3.png)

## まとめ: コンフリクトが起きたらどうすれば良いのか

コンフリクトが起きるのは、「requestで追加された要求と、それ以外の要求(`.lock`されているパッケージや、`.json`で指定されている範囲)の整合性がとれない」ことで発生します。

そのための解決の手順は

1. 最終的にどの要求がコンフリクトしているのかを調べる
2. その要求の大元となっている(上流の)要求を調べる
3. 大元の要求かコンフリクト先のパッケージバージョンを、緩めたり調整できないかを判断する

ということになります。  
コンフリクトが起きた時には表示されるメッセージは、「少し複雑で怖いな・・・よく分かりにくいぞ！」という印象を持つかも知れませんが、まずは落ち着いて _these were not loaded_ が書かれている行を見つけてください。

rootからのrequestになっていないパッケージは、 `-W`(` --update-with-all-dependencies`)フラグで自動的に「要求に合致するバージョンに更新する」ことが可能です。これを踏まえて、 「本当に直接的なrequestとして、`composer.json` に指定する必要があるのか？」を見直していくことも大事です。

それでもコンフリクトが発生したときには、 `prohibits` コマンドを上手く利用しながら、適切な依存関係を構築していきましょう。