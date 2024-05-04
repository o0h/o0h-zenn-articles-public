---
title: "リクエストを受け付け、素朴な画面の描画をする"
---

## この章で行うこと

- PSR-7のHttp Messageを用いて、簡単な出力を行えるようにする
- FW全体の基礎となるApplicationクラスを用意する

## この章で作るものの全体像

まずは、「FWとアプリケーションの基礎を作ってみる」「ブラウザ上で画面に何かしら表示できる」という所から始めましょう。
実装する内容は、「リクエスト等の状態によらず、固定文字列を出力する」だけの素朴な機能です。この段階では、アプリケーションとしては面白みにかけるかも知れませんが、小さく一歩ずつ進めていきましょう！

3つのクラスと、それを利用・補助する2つのクラスを作成していきます。
![](/./images/books/kantan-fw-lite/emitter/2.png)

1. 受け付けたリクエストの処理の基礎を担う `BaseApplication`クラス
   - それを継承する、アプリケーション側の `Application` クラス
2. PSR-7メッセージに準拠した、レスポンスを表現する`Response`クラス
   - それを簡単に生成するための`ResponseFactory` クラス
3. レスポンス内容をHTTP出力に変換し、出力する `Emitter`クラス

これらを連携させた全体的な流れが次のシーケンス図の通りになります。
![](/./images/books/kantan-fw-lite/emitter/1.png)

1. `index.php` で、 `Application`の生成と距離の起動を行う。その実務は、親クラスとなる`BasApplication`によって担う
   - これにより、「制御の逆転」が実現します
1. `BaseApplication`で、`ResponseFactory`を利用したレスポンス内容の組み立てと、`Emitter`を利用したHTTP出力を行う
   - PSR-7の世界に入門しましょう！

## まずは、すべての起点となるApplicationから

それではコードを書いていきましょう。

「最低限の実装でブラウザに `Hello, world!` を表示する」というのを、1番最初のゴールに設定します。

まずはFWのFWたる理由、「制御の逆転」を実現する起点を作ります。`Application`クラスです。
アプリケーション側の `Application`(具象クラス)と、その継承元となるFW側の `BaseApplication`(抽象クラス)を作成します。

`app/fw/src/BaseApplication.php`に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite;

abstract class BaseApplication
{
    final public function run()
    {
        echo 'Hello, world!';
    }
}

```

これで、「インスタンス化して、メインとなるメソッドである`run()`を呼び出せば、`Hello, world!`を出力してくれる」という仕事が出来上がります。

しかし、これは「FW側の住人」ですので、自分たちのアプリケーション側で利用するために`Application`を作成します。
現時点では、ロジックは必要ありません。`app/src/Application.php`に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KiribanBbs;

use O0h\KantanFwLite\BaseApplication;

final class Application extends BaseApplication {}

```

これで、この章で目指す最低限のロジックが出来ました！！

仕上げとして、HTTPのメッセージを最初に受け取る地点である「フロントコントローラー」と結合します。
`app/public/index.php`に、次のように記述してください。

```diff
<?php

-phpinfo();
+declare(strict_types=1);
+require dirname(__FILE__, 2) . '/vendor/autoload.php';
+
+$app = new \O0h\KiribanBbs\Application();
+
+$app->run();

```

やっていることは、

1. autoloaderを読み込む
2. `Application`をインスタンス化する
3. `run()`を実行する

という仕事です。

さぁ！これで、我々の最初のアプリケーションとFWが出来上がりました。
試しに、ブラウザから `http://localhost:8082` や `http://localhost:8082/hoge/fuga/piyo` にアクセスしてみてください。
しっかりHello Worldできているでしょうか？

## Applicationクラスから出力の仕事を分離

`Application`クラスは、「リクエストの内容を受け付けて、レスポンスの出力を行う」という責務を持ちますが、通常は出力部分は別のクラスに委譲することになります。
現段階の素朴なコンテンツを出力するだけのアプリケーションでは、そうした分離も少し過剰に感じる面を否めないですが、思い切って進めてしまいましょう。

ここで導入される概念が「エミッター(emitter)」です。エミットは「放出する・送る」といった意味を持つ単語で、FWの内では「HTTPレスポンスを出力する」という役割を担います。

まず、Emitterクラスを実装します。
`app/fw/src/Http/Emitter.php`に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

final readonly class Emitter
{
    public function emit(): void
    {

    }
}

```

(最終的には、Responseオブジェクトを受け付けることになります。すぐに出てくるので、今は我慢してください！)

そして、`BaseApplication`クラスにて担っていた出力部分を、`Emitter::emit()`に移しましょう。

```diff
--- a/app/fw/src/Http/Emitter.php
+++ b/app/fw/src/Http/Emitter.php
@@ -8,6 +8,6 @@
 {
     public function emit(): void
     {
-
+        echo 'Hello, world!';
     }
 }
--- a/app/fw/src/BaseApplication.php
+++ b/app/fw/src/BaseApplication.php
@@ -8,6 +8,5 @@
 {
     final public function run(): void
     {
-        echo 'Hello, world!';
     }
 }

```

仕上げに、`BaseApplication` クラスから`Emitter::emit()`を呼ぶようにします。

```diff
--- a/app/fw/src/BaseApplication.php
+++ b/app/fw/src/BaseApplication.php
@@ -4,9 +4,13 @@

 namespace O0h\KantanFwLite;

+use O0h\KantanFwLite\Http\Emitter;
+
 abstract class BaseApplication
 {
     final public function run(): void
     {
+        $emitter = new Emitter();
+        $emitter->emit();
     }
 }

```

これで、`BaseApplication`クラスが少し綺麗になりました！

## PSR-7のResponseを使う

これだけでChapterが終わるのも物足りないかと思うので、もう一捻りしてみましょう。

Emitterが「自分自身で出力されるコンテンツを決めている」のには違和感があります。やはり、表示するべき結果は外部(Application)から受け取るべきでしょう。

そこで、`Emitter::emit()`にデータを入力できるようにします。そのオブジェクトは、PSR-7のResponseです！

### PSR-7: HTTP message interfacesについて

実装に入る前に、現代的なPHP製FWの「リクエスト」「レスポンス」を表す際に一般的な企画となっている、PSR-7のインターフェイスについて簡単に解説します。

PSR-7は、「HTTPで扱われるメッセージを、PHPのクラスとしてどのように表現するか？」に関心を持ち、指針を示している規格です。
正式な仕様は、PHP-FIGのホームページから確認ができます。

https://www.php-fig.org/psr/psr-7/

通常、PHPの内部で「受け付けたリクエスト」を扱おうとするとスーパーグローバル変数等から内容を読み取ることになります。また、「出力するレスポンス」については、`header()` `echo`で操作すればOKなのですが、取り扱い方法や禁忌については規定がありません。「アプリケーションサーバーから外部のサービスにHTTPリクエストを送る/受け取る」といった際にも、利用する関数やライブラリによって、その形式はまちまちです。
これらに対して、統一的な指針を示すことで、FWやライブラリにおける相互運用性を高めようという狙いのもとでPSR-7が策定されました。

中核となるのが `MessageInterface`で、それを継承した `RequestInterface`・`ResponseInterface`・`ServerRequestInterface`の3つのインターフェイスが定義されています。

![](/./images/books/kantan-fw-lite/emitter/4.png)

「HTTPメッセージ」の基礎として、`MessageInterface`があります。
ざっくりと言うと、「ヘッダー、ボディと、プロトコルバージョンを扱うメソッド」の集合です。
それを継承して「クライアントからサーバー送信するもの」の表象として、 `RequestInterface`があります。
これは、HTTPメッセージに加えて「リクエストの送信先、送信メソッド」に関する操作が含まれます。
`ResponseInterface`はどうでしょうか？「サーバーからクライアントに送信するもの」です。あるいは、「クライアントがサーバーから受け取るもの」とも言えます。
HTTPメッセージに加えて、「レスポンスのステータス(code, reason phrase)」に関する操作が含まれます。
3つ目、`ServerRequestInterface`は、「サーバーがクライアントから受けとったもの」です。
「サーバー情報(`$_SERVER`をイメージしてください)、Cookie情報(`$_COOKIE`をイメージしてください)、リクエストボディを扱いやすくするもの、ファイルアップロードに関する情報(`$_FILES`)、自由に情報を付帯できるフィールド」に関する操作が含まれます。

基本的な情報は以上ですが、PSR-7は他にも特徴を持っています。

- リクエストヘッダー・ボディを扱うためのインターフェイス群が存在する点
  - リクエスト先情報等を扱うための `UriInterface`
  - リクエストボディを扱うための`StreamInterface`
  - ファイルアップロード情報を扱うための `UploadedFileInterface`
- これらのオブジェクトが、全て不変オブジェクト(イミュータブル)として定義されている点

このbookでは詳細に踏み込むのを避けますが、詳しく知りたい方には以下のリソースをおすすめします。

https://www.slideshare.net/sasezaki/httpphp

https://sasezaki.hatenablog.com/entry/2015/03/07/195908

https://speakerdeck.com/tanakahisateru/17ninatutafalseka

また、これらのオブジェクトの生成についても、後続のPSRで規格が策定されました。
PSR-17です。

https://www.php-fig.org/psr/psr-17/

これは、`RequestFactoryInterface`・`ResponseFactoryInterface`・`ServerRequestFactoryInterface`の3つのインターフェイスが含まれます。

ここまでを前提知識として、私たちは `ResponseInterface`を実装したクラスを作成していきましょう

### PSR-7のResponseを作成する

まず、PSRインターフェイスを利用できるようにパッケージを追加します。

`fw`ディレクトリ以下で`composer require`を実施するか、`--working-dir`オプションを利用して`fw` PJへパッケージを追加してください。

追加するパッケージは以下のとおりです。

- `psr/http-message`: PSR-7のパッケージ。`@stable`か `^2.0`を指定することをおすすめします
- `psr/http-factory`: PSR-17のパッケージ。`@stable`を指定することをおすすめします
- `fig/http-message-util`: PSR-7の実装を補助するパッケージ。`@stable`を指定することをおすすめします

```
root@fa8c293ef1d0:/var/www/html/app/fw# composer require --prefer-stable psr/http-message psr/http-factory fig/http-message-util
./composer.json has been updated
Running composer update psr/http-message psr/http-factory fig/http-message-util
Loading composer repositories with package information
Updating dependencies
Lock file operations: 3 installs, 0 updates, 0 removals
  - Locking fig/http-message-util (1.1.5)
  - Locking psr/http-factory (1.0.2)
  - Locking psr/http-message (2.0)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 3 installs, 0 updates, 0 removals
  - Downloading fig/http-message-util (1.1.5)
  - Downloading psr/http-message (2.0)
  - Downloading psr/http-factory (1.0.2)
  - Installing fig/http-message-util (1.1.5): Extracting archive
  - Installing psr/http-message (2.0): Extracting archive
  - Installing psr/http-factory (1.0.2): Extracting archive
Generating autoload files
No security vulnerability advisories found.
Using version ^2.0 for psr/http-message
Using version ^1.0 for psr/http-factory
Using version ^1.1 for fig/http-message-util
```

:::message
FW側でパッケージの追加をした場合、アプリケーション側のプロジェクトに対して「間接依存が新しく発生した」という情報を伝える必要があります。
`/app`ディレクトリ以下で、`composer update`を実行してください。
:::

```
root@fa8c293ef1d0:/var/www/html/app# composer update
Loading composer repositories with package information
Updating dependencies
Lock file operations: 3 installs, 1 update, 0 removals
  - Locking fig/http-message-util (dev-master 9d94dc0)
  - Upgrading o0h/kantan-fw-lite (dev-main 9c6e1e9 => dev-main 0c7a4db)
  - Locking psr/http-factory (dev-master 7037f4b)
  - Locking psr/http-message (dev-master 402d35b)
Writing lock file
Installing dependencies from lock file (including require-dev)
Package operations: 3 installs, 1 update, 0 removals
  - Downloading psr/http-factory (dev-master 7037f4b)
  - Installing psr/http-message (dev-master 402d35b): Extracting archive
  - Installing psr/http-factory (dev-master 7037f4b): Extracting archive
  - Installing fig/http-message-util (dev-master 9d94dc0): Extracting archive
  - Upgrading o0h/kantan-fw-lite (dev-main 9c6e1e9 => dev-main 0c7a4db): Source already present
Generating autoload files
No security vulnerability advisories found.

```

パッケージが追加できたら、一気に `Response`クラスと`Stream`クラスを作ってしまいましょう！
コードとしては単純なメソッドの集合になるのですが、少し分量が多いので気をつけてください。

`app/fw/src/Http/Response.php` と`app/fw/src/Http/Stream.php` に、次のように記述してください。
各メソッドの実装については説明を割愛します。もし、メソッドごとの機能が気になるようであれば、PSRの仕様を参照してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Fig\Http\Message\StatusCodeInterface;
use Psr\Http\Message\MessageInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\StreamInterface;

class Response implements ResponseInterface
{
    private int $statusCode = StatusCodeInterface::STATUS_OK;

    private string $reasonPhrase = 'OK';

    private array $headers = [];

    private StreamInterface $body;

    private string $protocolVersion;

    public function getStatusCode(): int
    {
        return $this->statusCode;
    }

    public function withStatus($code, $reasonPhrase = ''): self
    {
        $new = clone $this;
        $new->statusCode = $code;
        $new->reasonPhrase = $reasonPhrase;

        return $new;
    }

    public function getReasonPhrase(): string
    {
        return $this->reasonPhrase;
    }

    public function getProtocolVersion(): string
    {
        return $this->protocolVersion;
    }

    public function withProtocolVersion(string $version): MessageInterface
    {
        $new = clone $this;
        $new->protocolVersion = $version;

        return $new;
    }

    public function getHeaders(): array
    {
        return $this->headers;
    }

    public function hasHeader(string $name): bool
    {
        return isset($this->headers[$name]);
    }

    public function getHeader(string $name): array
    {
        $value = $this->headers[$name] ?? null;
        return  (array)$value;
    }

    public function getHeaderLine(string $name): string
    {
        return implode(',', $this->getHeader($name));
    }

    public function withHeader(string $name, $value): MessageInterface
    {
        $new = clone $this;
        $new->headers[$name] = (array)$value;

        return $new;
    }

    public function withAddedHeader(string $name, $value): MessageInterface
    {
        $values = $this->getHeader($name);
        $values[] = $value;

        return $this->withHeader($name, $values);
    }

    public function withoutHeader(string $name): MessageInterface
    {
        if (!$this->hasHeader($name)) {
            return $this;
        }

        $new = clone $this;
        unset($new->headers[$name]);

        return $new;
    }

    public function getBody(): StreamInterface
    {
        return $this->body;
    }

    public function withBody(StreamInterface $body): MessageInterface
    {
        $new = clone $this;
        $new->body = $body;

        return $new;
    }
}

```

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\StreamInterface;

class Stream implements StreamInterface
{
    /** @var resource $resource */
    private $resource;

    /**
     * @param resource|string|null $contents
     */
    public function __construct($contents = null)
    {
        if (is_resource($contents)) {
            $this->resource = $contents;
            return;
        }

        $resource = tmpfile();
        $written = fwrite($resource, $contents ?? '');
        if ($written === false) {
            throw new \RuntimeException('読み込みに失敗しました');
        }
        $this->resource = $resource;
    }

    public function __toString(): string
    {
        return $this->getContents();
    }

    public function close(): void
    {
        if (!$this->resource) {
            return;
        }

        fclose($this->resource);
    }

    public function detach()
    {
        $resource = $this->resource;

        $this->close();
        $this->resource = null;

        return $resource;
    }

    public function getSize(): ?int
    {
        if (!$this->resource) {
            return null;
        }

        $stats = fstat($this->resource);

        return $stats['size'] ?? null;
    }

    public function tell(): int
    {
        if (!$this->resource) {
            throw new \RuntimeException('リソースが存在しません');
        }
        $tell = ftell($this->resource);
        if (!is_int($tell)) {
            throw new \RuntimeException('読み込みに失敗しました');
        }

        return $tell;
    }

    public function eof(): bool
    {
        return $this->resource && feof($this->resource);
    }

    public function isSeekable(): bool
    {
        return $this->resource && $this->getMetadata('seekable');
    }

    public function seek($offset, $whence = SEEK_SET): void
    {
        if (!$this->resource) {
            throw new \RuntimeException('リソースが存在しません');
        }
        if (!$this->isSeekable()) {
            throw new \RuntimeException('リソースはシークできません');
        }

        fseek($this->resource, $offset, $whence);
    }

    public function rewind(): void
    {
        $this->seek(0);
    }

    public function isWritable(): bool
    {
        if (!$this->resource) {
            return false;
        }
        $mode = $this->getMetadata('mode');
        if (!$mode) {
            return false;
        }

        return preg_match('/[wacx+]/', $mode);
    }

    public function write($string): int
    {
        if (!$this->isWritable()) {
            throw new \RuntimeException('書き込みできません');
        }

        $written = fwrite($this->resource, $string);
        if ($written === false) {
            throw new \RuntimeException('書き込みに失敗しました');
        }

        return $written;
    }

    public function isReadable(): bool
    {
        if (!$this->resource) {
            return false;
        }
        $mode = $this->getMetadata('mode');

        return preg_match('/[r+]/', $mode) === 1;
    }

    public function read($length): string
    {
        if (!$this->isReadable()) {
            throw new \RuntimeException('読み込みできません');
        }

        $read = fread($this->resource, $length);
        if ($read === false) {
            throw new \RuntimeException('読み込みに失敗しました');
        }

        return $read;
    }

    public function getContents(): string
    {
        if (!$this->isReadable()) {
            throw new \RuntimeException('読み込みできません');
        }

        $this->rewind();
        $contents = stream_get_contents($this->resource);
        if ($contents === false) {
            throw new \RuntimeException('読み込みに失敗しました');
        }

        return $contents;
    }

    public function getMetadata($key = null): mixed
    {
        if (!$this->resource) {
            return null;
        }
        $meta = stream_get_meta_data($this->resource);

        if ($key === null) {
            return $meta;
        }

        return $meta[$key] ?? null;
    }
}

```

続いて、`Response`・`Stream`オブジェクトを生成するファクトリを実装します。

`app/fw/src/Http/ResponseFactory.php` `app/fw/src/Http/StreamFactory.php`・に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;

class ResponseFactory implements ResponseFactoryInterface
{
    public function createResponse(int $code = 200, string $reasonPhrase = ''): ResponseInterface
    {
        return (new Response())->withStatus($code, $reasonPhrase);
    }
}

```

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\StreamFactoryInterface;
use Psr\Http\Message\StreamInterface;

class StreamFactory implements StreamFactoryInterface
{
    public function createStream(string $content = ''): StreamInterface
    {
        return new Stream($content);
    }

    public function createStreamFromFile(string $filename, string $mode = 'r'): StreamInterface
    {
        $resource = fopen($filename, $mode);
        if (!$resource) {
            throw new \RuntimeException('ファイルのオープンに失敗しました');
        }
        return new Stream($resource);
    }

    public function createStreamFromResource($resource): StreamInterface
    {
        return new Stream($resource);
    }
}

```

これで、PSR-7デビューが完了です！！

### Responseから出力内容を決める

それでは、本来の目的である`Emitter`からのコンテンツ決定の分離を進めていきましょう。

まず、`Emitter::emit()`が`Response`を受け付けられるようにして、その内容をもとに出力をするようにします。

```diff
--- a/app/fw/src/Http/Emitter.php	(revision ebf52fbb9208e1a26e74e4c9237086cde6767f36)
+++ b/app/fw/src/Http/Emitter.php	(date 1701973481255)
@@ -4,10 +4,22 @@

 namespace O0h\KantanFwLite\Http;

+use Psr\Http\Message\ResponseInterface;
+
 final readonly class Emitter
 {
-    public function emit(): void
+    public function emit(ResponseInterface $response): void
     {
-        echo 'Hello, world!';
+        header(
+            header: sprintf('HTTP/1.1 %d %s', $response->getStatusCode(), $response->getReasonPhrase()),
+            response_code: $response->getStatusCode(),
+        );
+        foreach ($response->getHeaders() as $headerField => $headerValues) {
+            foreach ((array)$headerValues as $headerValue) {
+                header("{$headerField}: {$headerValue}");
+            }
+        }
+
+        echo $response->getBody();
     }
 }

```

さほど長くないコードですが、`Emitter`の役割としてはコレで十分です。FWの完成まで、もう手を加えることはないでしょう。
(もちろん、もっと厳密・高性能に実装することはできるのですが、今回は学習目的でFWを作成しているので細かいことは省略です！)

やっていることは3つです。

1. ステータスの出力(header関数)
   - ステータスコードと、リーズンが必要です
2. レスポンスヘッダーの出力
   - 行ごとにheader関数を呼びだし、もし複数の値を持つフィールドの場合には2回目以降は上書き(replace)しないようにします
3. レスポンスボディの出力
   - `Response::getBody()`は、`StreamInterface`を返しますが、これは`__toString`メソッドを実装していますので、そのまま出力して構いません

新しくなったEmitterを、Applicaitonクラスから利用するようにします。
そのために、「`Response`を生成する」「`Emitter::emit()`に`Response`を渡す」という変更が必要になります。
また、折角なので少し出力するコンテンツも変更してみましょう！

`Emitter`と`BaseApplication`を、次のように変更してください。

```diff
--- a/app/fw/src/Http/Emitter.php	(revision 415d65a5f9fedb5dd3a80d8be4a75a70af058346)
+++ b/app/fw/src/Http/Emitter.php	(date 1701974024207)
@@ -4,10 +4,24 @@

 namespace O0h\KantanFwLite\Http;

+use Psr\Http\Message\ResponseInterface;
+
 final readonly class Emitter
 {
-    public function emit(): void
+    public function emit(ResponseInterface $response): void
     {
-        echo 'Hello, world!';
+        header(
+            header: sprintf('HTTP/1.1 %d %s', $response->getStatusCode(), $response->getReasonPhrase()),
+            response_code: $response->getStatusCode(),
+        );
+        foreach ($response->getHeaders() as $headerField => $headerValues) {
+            $toReplace = true;
+            foreach ((array)$headerValues as $headerValue) {
+                header("{$headerField}: {$headerValue}", replace: $toReplace);
+                $toReplace = false;
+            }
+        }
+
+        echo $response->getBody();
     }
 }
--- a/app/fw/src/BaseApplication.php	(revision 415d65a5f9fedb5dd3a80d8be4a75a70af058346)
+++ b/app/fw/src/BaseApplication.php	(date 1701977802321)
@@ -5,12 +5,21 @@
 namespace O0h\KantanFwLite;

 use O0h\KantanFwLite\Http\Emitter;
+use O0h\KantanFwLite\Http\ResponseFactory;
+use O0h\KantanFwLite\Http\StreamFactory;
+use Psr\Http\Message\ResponseFactoryInterface;

 abstract class BaseApplication
 {
     final public function run(): void
     {
+        $contents = sprintf('Hello, World!<br> %sは、ただいま%sです', date('e'), date('D'));
+        $body = (new StreamFactory())->createStream($contents);
+        $response = (new ResponseFactory())->createResponse()
+            ->withHeader('Content-Type', 'text/html')
+            ->withBody($body);
+
         $emitter = new Emitter();
-        $emitter->emit();
+        $emitter->emit($response);
     }
 }

```

変更できたら、ブラウザから `http://localhost:8082`にアクセスしてみましょう。  
新しくなったHello,world!はちゃんと表示されたでしょうか？
![](/./images/books/kantan-fw-lite/emitter/5.png)

無事に表示されたら、成功です！！  
これであなたは、「FWを作って」「PSR-7のオブジェクトを使って」「コンテンツをブラウザで解釈可能な形で表示する」ところまで、実装を完了しました。
