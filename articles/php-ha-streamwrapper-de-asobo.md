---
title: "streamWrapperでレガシーコードと少しうまく付き合う"
emoji: "📿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [PHP]
publication_name: "neinc_tech"
published: true
---

こんにちは、NE会社で働いております[きんじょう(@o0h\_)](https://twitter.com/o0h_)がお送りします。

みなさんは、「レガシーコード」と聞くとどんなイメージが浮かぶでしょうか？

「結実するあの時の約束、先人たちの夢と努力の結晶」「見果てぬ夢を追い求めた、フロンティア精神の足跡」「生きた歴史そのものであり、語り継がれる旗印」、色々とあると思います。
裏を返せば、絶対的な基準で、かつ定量化した評価で「これがあるとレガシー」「これがないとレガシー」とは言いにくいものであります。
文脈に応じて、すなわちレガシーなコードの何(どんな問題)を論じたいか？という主旨に応じて、着目するべき性質が変わってきますよね。[^nani-ga-legacy]

さて、この記事もタイトルに「レガシーコード」を冠しています。話を進めていくには「どんなレガシー」を定義していく必要があります。  
という訳で、 今回取り扱いたい話題に合わせての「例えばこんなレガシー」は次のような想定です。
なお、PHPの話をするものなので、スコープもPHPに絞ります。

- 依存の解決には、明示的にrequire/includeを使ったクラスファイルのロードを行っている
  - 色々なところにインクルード系の命令が出現してきます
  - DIコンテナはもちろんオートローディングなどの仕組みを用いておらず、依存者が自らの範疇で依存先の読み込み処理を用意している状態です
- 副作用のあるファイルが分離されておらず、トップレベルで = ファイルを読み込んだ時点で何かしらの処理が発生しかねない
  - これには「ファイル冒頭で記述された `require_once`」なども含まれます
- クラス名や関数名・定数名の重複定義が存在する
  - 名前空間を利用した重複回避が行われていない場合には、同様の命名が為される事もしばしばありました
  - 現在では、PSR-4/PSR-0のように「ベンダー名やファイルパスに応じて規則的な命名を行う」ことによって、(名前の専有という意味では)ちょっとしたファイルスコープのような意識が形成されています
- システムやプロセス全体に影響を及ぼす処理の散在
  - `exit()` を実行するコードであったり、`ini_set()` の呼び出しなどが、昨今のコードでは一部に集約させられているかと思います
    - 大局を統治するMiddlewareクラスであったり、初期設定を行うbootstrapファイルであったり、という雰囲気です
  - (たとえばPEAR時代くらいの)古くからあるコードに触れたことがある人の中には、「コントローラアクションの中でおもむろに `exit()` が呼ばれている」といった場面を懐かしく思い出せる人もいるのではないでしょうか

概して、 **「安全にファイルを読み込めるか、利用者側が簡単に判断をしにくい。予期していない呼称や中断のリスクが高い」ような状況が発生している**、と言えます。
これが今回の記事で向きあいたいレガシーです。

[^nani-ga-legacy]: 「レガシーとは」についていえば、最近の話題では個人的にhanhan1978さんの資料を参照することが多いです。素敵な。発表でした。 https://speakerdeck.com/hanhan1978/avoid-php-legacy

この記事では、そうした問題に対して「ファイル読み出しの危険性を退ける」方向でアプローチを試みるために、streamWrapperの実装例・使用方法を紹介します。

## お先に参考記事を

日常的には使わない(・・・・使われるべきでもない)機能ですが、日本語のリソースでも非常に有用で興味深いものが存在しています。  
自分としてもかなりお世話になったので、敬意を込めて記事の最初に紹介しておきます。

まずは公式ドキュメントです。
必要なAPIなどはココを見るのが網羅性が高いです。

https://www.php.net/manual/ja/class.streamwrapper.php
https://www.php.net/manual/ja/stream.streamwrapper.example-1.php
https://www.php.net/manual/ja/stream.examples.php

次が、PHP-VCRのコードリーディングと解説を行っているkunitさんの記事です。
自分がstreamWrapperの存在を認知したのは「PHP-VCRってどうやって動いているんだ・・？」と興味を持ち[^php-vcr]、たどり着いたこちらの記事がキッカケでした。
https://kunit.jp/archives/108

[^php-vcr]: PHP-VCRとの出会いは、またもやhanhanさんの発信となりますが、PHP勉強会@東京での発表でした https://tech.innovator.jp.net/entry/2019/03/01/172358

最後に紹介するのが、streamWrapperそのものを主題として扱っているtzm_freedomさんの記事です。
(込み入ったことを丁寧に踏み込んだ上で分かりやすく記述してくれている事が多く、時折&しばしばお世話になっているブログの1つです)

https://blog.freedom-man.com/php-mock-streamwrapper
https://blog.freedom-man.com/phpinput_streamwrapper

## ざっくり「streamWrapperってなんだ？」

話を戻して。

PHPが何らかを読み出したり書き出したりする際には、読み出し元の情報源であったり出力先のリソースを扱う必要があります。
そうした何かを抽象化して、「開く・閉じる・移動する」「読み出す・書き出す」といった振る舞いを提供しているのがストリームです。

> ストリームは、 ファイル、ネットワーク、データ圧縮などに関する、 共通した一連の関数群と利用法を持つ操作の一般化の手法です。 もっとも単純な定義では、ストリーム というのは、ストリーミング可能な動作を体現する resource オブジェクトといえます。つまり、ストリームには線的に読み出したり、 あるいは書き込んだりすることが可能で、かつ、 ストリーム上の任意の場所に fseek() できる場合もあります。
>
> https://www.php.net/manual/ja/intro.stream.php

たとえば `file_get_contents()` も「リソースからデータを読み出す」ものですし、 `fwrite()` も「リソースにデータを書き込む」ものです。

指定されたURIがどのようなストリームであるか、を判断するためにschemaを記述します。
'https://' でURIの記述が始まっていれば「HTTPSアクセスをしよう！」となりますし、 `file_put_contents('php://stdout', 'いええい!!');` のように書けば「PHPの持つ特殊なストリームであり、今回は標準出力に”いえええい”したいんだな」というコードになります。

そして、そんな仕組みのストリームについて、「(組み込みのプロトコルも含めて)任意の処理をユーザーランドから設定できる」という仕組みがあるのです！  
それが`streamWrapper` となります。

## streamWrapperを登録してみる

たとえば、「AWSのS3と気軽に通信したいので、`opendir('s3://hogehoge)` で開けるようにしよう！と言った場合。
`stream_wrapper_register()` 関数がありますので、コレを利用します。

```php
stream_wrapper_register('s3', AwsS3StreamWrapper::class, \STREAM_IS_URL);
```

こんな具合で済むので、さほど複雑なことはありません。

もし「組込みプロトコルなど、すでに登録済みのstreamWrapperを無効化したい」と言った場合には、先んじて登録解除を行います。

```php
stream_wrapper_unregister('file');
stream_wrapper_register('file', MyFileStreamWrapper::class);
```

これで、PHPが内部で使っているファイルの読み書きの挙動を乗っ取る手はずが整います！

## 実際に簡単なstreamWrapperを作ってみる

「実際にstreamWrapperはどう作るんだろう？」という理解を深めるために、実用的ではないですが興味深いstreamWrapperを作ってみましょう。

例えば、皆さんは次のPHPコードを見たときに、どんな結果になると想像しますか？

```php
<?php

declare(strict_types=1);

require 'fizzbuzz://1005';
```

実際に3v4lで動かしてみた結果がこちらです。

![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-03-44-50.png)

そうです！動くわけがありません。
コレが動くように、「fizzbuzzストリーム」を作っていきましょう

動作イメージとしては、以下のように「`fizzbuss` プロトコル + 任意の自然数」を開くと、任意の自然数までのFizzBuzzをechoするものになります。

![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-03-54-15.png)
https://3v4l.org/7XcYR
![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-03-54-41.png)
https://3v4l.org/APMn4

というわけで、FizzBuzzStreamWrapperを。実装していきます。

streamWrapperはPHP標準でのInterface定義がなく、必要なメソッドを実装したクラスを定義していくことになります。
Interfaceが存在せずにダックタイピングで動作する利点として、提供可能とされる全てのメソッドを定義しなくてもコンパイルエラーにならずに済む点が挙げられます。  
例えば、`dir_opendir()`や`unlink()` はFizzBuzzで遊びたい今の我々にとっては不必要な操作です。

先に、できあがる全体像を示します。

```php
<?php
final class FizzBuzzStreamWrapper
{
    /** @var resource */
    public $context;

    private int $counter = 1;

    private int $length = 15;

    public function stream_open($path, $mode, $options, &$opened_path): bool
    {
        preg_match('#([\d]+$)#', $path, $matches);
        $this->length = filter_var(
            $matches[1] ?? '',
            FILTER_VALIDATE_INT,
            ['options' => ['default' => 15]]
        );

        return true;
    }

    public function stream_stat(): array|false
    {
        return ['size' => $this->getSize()];
    }

    public function stream_read($count): string|false
    {
        $fizzbuzz = match (true) {
            $this->counter % 15 === 0 => 'FizzBuzz',
            $this->counter % 5 === 0 => 'Buzz',
            $this->counter % 3 === 0 => 'Fizz',
            default => (string)$this->counter
        };

        $contents = '';
        if ($this->counter === 1) {
            $contents = '<?php ' . PHP_EOL;
        }
        $contents .= "echo '{$fizzbuzz}' . PHP_EOL;";

        $this->counter++;

        return $contents;
    }

    public function stream_eof(): bool
    {
        return $this->counter ===$this->length;
    }

    public function stream_close(): bool
    {
        return true;
    }

    public function stream_set_option(int $option, int $arg1, int $arg2)
    {
        return false;
    }

    private function getSize()
    {
        $ret = strlen("<?php \n") + $this->length * strlen("echo '' . PHP_EOL;");
        $range = range(1, $this->length);
        $numbers = array_filter($range, fn ($n) => ($n % 3 !== 0 && $n % 5 !== 0));
        $ret += strlen(implode('', $numbers));
        $fizzOrBuzzs = array_filter($range, fn ($n) => ($n % 15 !== 0) && ($n % 3 === 0 || $n % 5 === 0));
        $ret += strlen('fizz') * count($fizzOrBuzzs);
        $fizzbuzzs = array_filter($range, fn ($n) => ($n % 15 === 0));
        $ret += strlen('fizzbuzz') * count($fizzbuzzs);

        return $ret;
    }
}
```

6つのメソッドと1つのパブリックメンバーが必須であり、これが最低限の実装になります(多分・・)

各メソッドの要件・機能については、凡そ自明な命名にもなっていますので、[公式ドキュメント](https://www.php.net/manual/ja/class.streamwrapper.php) にゆずります。
この記事では、いくつか実装上の注意点に言及します。

### パブリックメンバー `$context` を忘れずに

`streamWrapper::$conetxt` というメンバーがいます。これは、`file_get_contents()` の第3引数で指定できるようなコンテキストが自動でセットされるものです。
コンテキストについては、公式ドキュメントの下記ページにある説明を参照してください。
https://www.php.net/manual/ja/context.php

特にコンテキストの取り扱いがない場合(今回の例も該当します)でも、書込み可能なメンバーが定義されていないとエラーを吐くことになりますので、おまじない感覚で定義しておきましょう

### statとsize

ここは、本来であれば `stat()` と同じ構造の情報を返しておくのが筋です。が、簡易的にやるのであれば今回の例のように大幅に割愛できます。
あるいは、空配列を返しても問題がないケースもあるでしょう。

https://www.php.net/manual/ja/function.stat.php

ただし、ファイルが読み出された目的によっては、このメソッドの返すデータサイズ情報に依存して挙動が変化します。  
例えば `file_get_contents()` での読み出しは、 `stream_read()` さえ適切に動けば目的が達成できる可能性があります。その一方で、`require` `include` 等のPHPとしての評価を伴う形でのファイル読み出しの際には、ストリームが返すデータとsize情報が一致している必要があります。
(そのため、今回は「生成されるであろうソースコードの文字数を計算するロジック(`getSize()`)」を実装しています。)

実例を示します。
次のキャプチャは、「sizeを0で固定してstatを返すようにした」際のコード(差分)と動作結果です。
`Failed opening` となっていることが分かります。
これは「読み出せたデータがない」という挙動になっていると思われます[^src-yomenai]
![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-04-38-11.png)
https://3v4l.org/gjHtc

では、0より大きな値(ただし必要なサイズより小さい)を返すとどうなるでしょうか？
今度は `Parse error: syntax error, unexpected end of file` になりました！
おそらく `size` の値となる30bytesまでデータを読み込んで、それ以後が切り詰められた(メモリから落とされた)感じでしょうか。

![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-04-12-10.png)
https://3v4l.org/drbAL

ちなみに、実際のデータサイズよりも大きい値をsizeにセットした場合にも、`Failed opening required` になります。

そして、次の例が `file_get_contents()` での読み出しの例です。
`size` は0を受け取りつつも、内容は受信できていることが分かります。
![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-04-20-50.png)
https://3v4l.org/3Pf5Y

そうした訳で、`stream_stat` の実装の際には注意してみてください。

[^src-yomenai]: php-src内部まで追えていないので、間違っていたらご指摘いただけると喜びます・・

### 各メソッドの呼び出し順序

基本的には、
open -> set_option -> stat -> (read -> eof){while(!eof)} -> close
という順番になります。

最初にstatを読み込んでいるのは、メモリの確保等との兼ね合いかな？と推察しています。[^src-yomenai]
echoデバッグをしかけて実行してみた結果が、次の通りになりました。

![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-04-39-36.png)
https://3v4l.org/vNfpt

## やや実践: 「危険」なファイルを無害化してみる

冒頭で挙げた問題について、取り組んでみます。

先の例は独自のプロトコル `fizzbuzz` を利用しましたが、今回は実際に `require` に関与していくために、`file` プロトコルを乗っ取っていきます。

### まずは動作イメージの確認

サンプルとして、以下のような `danger.php` ファイルを用意しました。通常は、読み込まれると即死します。

```php
<?php
declare(strict_types=1);

echo basename(__FILE__) . 'が読み込まれました' . PHP_EOL;
exit();
```

これを読み込む `main.php` は次の内容になります。

```php
<?php

declare(strict_types=1);

//require_once __DIR__ . '/SafeExitStreamWrapper.php';
//stream_wrapper_unregister('file');
//stream_wrapper_register('file', SafeExitStreamWrapper::class);
set_error_handler(function($errno, $errstr, $errfile = '', int $errline = 0, array $errcontext = [] ) {
    printf('[ERROR (errno %d)] %s (in %s:%d)' . PHP_EOL, $errno, $errstr, basename($errfile), $errline);
});

echo __LINE__ . '行目まで実行されました' . PHP_EOL;

require __DIR__ . '/danger.php';

echo __LINE__ . '行目まで実行されました' . PHP_EOL;
```

複数ファイルを用いるために3v4lではなく手元で実行しました。
すると、ファイルを読み込んだ次の瞬間に処理が停止している様子が見て取れます。

![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-04-41-55.png)

では、`main.php` を修正してみましょう

```php
<?php

declare(strict_types=1);

require_once __DIR__ . '/SafeExitStreamWrapper.php';
stream_wrapper_unregister('file');
stream_wrapper_register('file', SafeExitStreamWrapper::class);
set_error_handler(function($errno, $errstr, $errfile = '', int $errline = 0, array $errcontext = [] ) {
    printf('[ERROR (errno %d)] %s (in %s:%d)' . PHP_EOL, $errno, $errstr, basename($errfile), $errline);
});

echo __LINE__ . '行目まで実行されました' . PHP_EOL;

require __DIR__ . '/danger.php';

echo __LINE__ . '行目まで実行されました' . PHP_EOL;
```

そして実行すると・・・先程は到達できなかった、16行目まで処理が継続されました！
嬉しいですね。

![](/images/php-ha-streamwrapper-de-asobo/2023-09-22-04-43-44.png)

### streamWrapperを作る

では、肝心のstreamWrapperの中身を見ていきましょう。

なお、今回は簡略化のためにコードの記述を端折っている箇所があります。
無限ループなどの危険な挙動を発生させる可能性もあるため、利用時には注意してください。  
(例えば、「リソースの読み出しが終端に達した」ということを適切に知らせるために、 `stream_read()`は空文字やnullを返す必要があります)

```php
<?php
declare(strict_types=1);

final class SafeExitStreamWrapper
{
    /** @var resource */
    public $context;

    /** @var resource|false */
    private $fp;

    private int $position = 0;

    private string|false $content = '';

    /** @var array<int|string, int|string|bool> */
    private array $stat = [];

    public function stream_open($path, $mode, $options, &$opened_path): bool
    {
        // nativeのfopen()を呼び出すために一旦退避
        stream_wrapper_restore('file');

        $this->fp = fopen($path, $mode);

        // stream wrapperの復活
        stream_wrapper_unregister('file');
        stream_wrapper_register('file', __CLASS__);

        return $this->fp !== false;
    }

    public function stream_read($count): string|false
    {
        $content = substr($this->content, 0, $count);
        $this->content = substr($this->content, $count);
        $this->position += strlen($content);

        return $content;
    }

    public function stream_close(): bool
    {
        return fclose($this->fp);
    }

    public function stream_eof(): bool
    {
        return $this->position >= $this->stat['size'];
    }

    public function stream_stat(): array
    {
        if ($this->stat) {
            return $this->stat;
        }

        // nativeのfstat()を呼び出すために一旦退避
        stream_wrapper_restore('file');

        $stat = fstat($this->fp);
        $content = str_replace(
            'exit(',
            __CLASS__ . '::sefeExit(',
            fread($this->fp, $stat['size'])
        );
        rewind($this->fp);

        // stream wrapperの復活
        stream_wrapper_unregister('file');
        stream_wrapper_register('file', __CLASS__);

        $this->content = $content;
        $stat = ['size' => strlen($content)];
        $this->stat = $stat;

        return $stat;
    }

    public function stream_set_option(int $option, int $arg1, int $arg2)
    {
        return false;
    }

    public static function sefeExit($status = 0)
    {
        trigger_error("exit({$status})を回避しました", E_USER_NOTICE);
    }
}
```

### 実装時のポイント

実装に際してのポイントを見ていきます。

#### 従来のストリーム関連の関数を利用したい場合には、一時的に登録を戻す

`fopen()` や `fstat()` を利用する場面で、streamWrapperの登録を解除・処理の実行後に再登録を行っています。
こうしないと、自メソッドを再帰的に呼び出して無限ループが発生してしまうためです。

#### サイズの確定のために、一旦すべてのデータを読み切ってしまう

先ほど、正確なサイズを返せないと不具合が生じてしまうことを説明しました。
それを回避するために、この実装ではサイズの取得時点 = `stream_stat()` 内で、一旦すべてのコンテンツを読み出してからそのサイズを数えるようにしています。
また、少しでも処理効率を軽くできると嬉しいので、サイズ計測用に利用したデータ全体が、インスタンスメンバーに保持されます。
`stream_read()` は、実ファイルを見に行くのではなくてメモリに読み込まれたデータをイテレーションしながら結果を返していくことになります。
そして、そのために `ftell()` のような実リソースからポインタの位置を取得する術が利用できなくなり、こちらも自前管理するようになっています。

ちなみに、 **「ファイルサイズを計算できないとファイルを読み込む際に困るので、ファイルを読み込む前にファイルをすべて読み込むことで、ファイルサイズの計算を実現しよう」・・・** と何とも頭の痛くなる処理になっていると思いませんか？  

ここで、先に言及した「ただのコンテンツ読み出しであれば `stream_stat()` の `size` 情報がおかしくても大丈夫で、PHPスクリプトとしての評価の際には困る」という挙動の差が効いてきます。
すなわち、「ファイルサイズを計算できないと `reuqire` の実行の際に困るので、ファイルを読み込む前に、size情報を扱わなくて済む方法で読み出しとカウントを実現しよう」と言い換えられます。

#### 実装をもっと端折ることができないか？ → できるにはできる

ストリームからのデータ読み込みのライフサイクルやカーソルの管理など、面倒臭さは増している気がしますよね。
この辺りは、「本来のファイルが持つサイズと実際に読み込ませるコンテンツ(ソースコード)のサイズに差異がある」という特殊性のために、どうしても複雑な対処が避けられないものになっています。

逆に言えば、「サイズが変わらなければもうちょい楽をできる」ということです。

例えば文字列置換の前と置換後の文字長が変わっていないのであれば、素の挙動をそのまま使えますよね。
`define()` と `\TMP\d()` であれば同じ文字数ですし、 `ini_set()` と `\TMP\is()` も同じです。
こうした制約を飲めるのであれば、実装は簡単になりそうです。

## リアルでの使い方を考える

面白いことが出来るんですね！！ということで、面白いおもちゃになります💪
・・・それだけで済ませると、記事タイトルの回収になりませんので、少し「実用性」に思いを馳せてみましょう

### file読み込み時の書き換えで、ファイル読み込み時にプログラムの動作を妨げるものを握りつぶす

`exit()` などの危険な処理に関しては、先ほど言及したとおりです。

同じく `stream_read()` でゴニョゴニョする方法として、`class_exists()` `function_exists()` `defined()` や重複したクラスなどFQSENの定義を無効化できます。
`preg_match()` で要素名を抽出して、`**_exists()` で重複をチェックして、その結果に応じて `stream_read()` から返す内容を変更するというものです。

- 要素名を変更する
- 一時的な名前空間の付与を行う
- ファイル内容を返さず、空文字を返す(ファイルの読み込みを無効化する)
- 上記の「無害化」に加えて、重複を知らせるためにログを吐いたりエラーを発火させる

といった手段がとれます。

これを利用することで、例えばテストの実行やクラスマップの作成といった「全てのファイルを読み込み、解釈する必要がある」といった処理の実装を推し進める手段になるかと思います。  
主目的となるアプリケーション実行(プロダクション環境)では起きないような、横断的・連続的なファイルの読み込みが発生する場面ですので、特にレガシー性の濃いコードですと、「いつもは問題ないけど実は問題があった」ことに気付く場面もあることでしょう。

(ただし、個人的には「クラス名の重複を調べる」というだけであれば、Composerのclassmap+dumpautoloadを使って・・を好みますが、それはまた別の機会に。)

### 独自のプロトコルの追加

先に「たとえば」としてS3に言及しましたが、AWSの公式SDKでは実際にstreamWrapperを登録しています。

https://github.com/aws/aws-sdk-php/blob/3.281.11/src/S3/StreamWrapper.php#L122

laravel/serializable-closureでも、実際に「独自のinlcudeを実現する」という処理を利用している箇所があります。

https://github.com/laravel/serializable-closure/blob/v1.3.1/src/Serializers/Native.php#L189

### テスト時のモッキング

ファイルや標準入出力・標準エラーなど、ストリームに関わる部分のテストをどうしましょうか？という際にも、使う余地があるかと思います。
(PHP-VCRも、似たような動機ではありますが)

mikey179/vfsstreamは、そうした課題を扱っているライブラリです。

https://medium.com/swlh/phpunit-mocking-the-real-file-system-f6da6d1802cf

## まとめ

とある沼に突っ込んだので、改めてstreamWrapperについて触ってみると同時に記事にまとめてみました！
色々な可能性があってワクワクを感じる一方で、この辺りを扱い始めるとどうしても闇っぽい感じになっていきそうなので、できることならあまり関わらずに生きていきたいですね
