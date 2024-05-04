---
title: "リクエストから動的なコンテンツを作成する"
---

## この章で行うこと

- コンテンツの作成ロジックを担うActionクラスを用意する
- PSR-7のHttp Messageを用いて、入力を受け付けられるようにする

## この章で作るものの全体像

`Emitter`で担っていたコンテンツの作成を、Actionのレイヤーに移していきます。
Actionは、MVCでいう「C」にあたります。

また、`ServerRequest`クラスを新設し、利用できるようにします。これを用いて、リクエスト内容に応じた動的なコンテンツを扱えるように実装していきましょう。

全体の流れは、以下のようになります。前の章で記述した内容は割愛してありますので、補完しながら呼んでください。

![](/images/books/kantan-fw-lite/action/1.png)

1. `ServerRequest`を、`index.php`から`Application`に注入する

2. `Applicaiton`は、`Action`の生成と呼び出しを行う
3. `Action`は、コンテンツを独自のロジックで生成し、`Response`の生成とコンテンツのセットを行う
   * 整理すると、`Application`から見た`Action`との関係は、「リクエストを渡してレスポンスを受け取る」というものになります

また、シーケンス図上では省略していますが、`Request`の生成はPSR-17のファクトリを利用して行うことになります。

## PSR-7のServerRequestを作成する

この節でいくつかのクラスを用意していくのですが、記述量が多めの部品の作成が続き、少し退屈かもしれません・・・
少々お付き合いください。

まずは`ServerRequest`クラスから作成していきましょう。
`Response`と同様で、1つ1つのメソッドは非常に単純なものになります。
・・・と、その前に！ `Message`インターフェイスに対する実装は、`Response`クラスでも既に扱いましたよね。そこそこの記述量があるので、この部分をtraitに切り出して、再利用してしまいましょう。

`MessageTrait`を作成し、次のように関連するメソッドをお引越ししてください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\MessageInterface;
use Psr\Http\Message\StreamInterface;

/**
 * @see \Psr\Http\Message\MessageInterface
 */
trait MessageTrait
{
    private array $headers = [];

    private string $protocolVersion;

    private StreamInterface $body;

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

```diff
--- a/app/fw/src/Http/Response.php
+++ b/app/fw/src/Http/Response.php
@@ -5,22 +5,17 @@
 namespace O0h\KantanFwLite\Http;
 
 use Fig\Http\Message\StatusCodeInterface;
-use Psr\Http\Message\MessageInterface;
+use O0h\KantanFwLite\Http\MessageTrait;
 use Psr\Http\Message\ResponseInterface;
-use Psr\Http\Message\StreamInterface;
 
 class Response implements ResponseInterface
 {
+    use MessageTrait;
+
     private int $statusCode = StatusCodeInterface::STATUS_OK;
 
     private string $reasonPhrase = 'OK';
 
-    private array $headers = [];
-
-    private StreamInterface $body;
-
-    private string $protocolVersion;
-
     public function getStatusCode(): int
     {
         return $this->statusCode;
@@ -44,74 +39,4 @@
     {
         return $this->protocolVersion;
     }
-
-    public function withProtocolVersion(string $version): MessageInterface
-    {
-        $new = clone $this;
-        $new->protocolVersion = $version;
-
-        return $new;
-    }
-
-    public function getHeaders(): array
-    {
-        return $this->headers;
-    }
-
-    public function hasHeader(string $name): bool
-    {
-        return isset($this->headers[$name]);
-    }
-
-    public function getHeader(string $name): array
-    {
-        $value = $this->headers[$name] ?? null;
-        return  (array)$value;
-    }
-
-    public function getHeaderLine(string $name): string
-    {
-        return implode(',', $this->getHeader($name));
-    }
-
-    public function withHeader(string $name, $value): MessageInterface
-    {
-        $new = clone $this;
-        $new->headers[$name] = (array)$value;
-
-        return $new;
-    }
-
-    public function withAddedHeader(string $name, $value): MessageInterface
-    {
-        $values = $this->getHeader($name);
-        $values[] = $value;
-
-        return $this->withHeader($name, $values);
-    }
-
-    public function withoutHeader(string $name): MessageInterface
-    {
-        if (!$this->hasHeader($name)) {
-            return $this;
-        }
-
-        $new = clone $this;
-        unset($new->headers[$name]);
-
-        return $new;
-    }
-
-    public function getBody(): StreamInterface
-    {
-        return $this->body;
-    }
-
-    public function withBody(StreamInterface $body): MessageInterface
-    {
-        $new = clone $this;
-        $new->body = $body;
-
-        return $new;
-    }
 }

```



続いて、`ServerRequest`の実装に入る前に、その依存対象である`UriInterface`の実装クラスを作成します。
`/app/fw/src/Http/Uri.php`に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\UriInterface;

class Uri implements UriInterface
{
    private string $scheme;
    private string $userInfo;
    private string $host;
    private ?int $port;
    private string $path;
    private string $query;
    private string $fragment;

    public function __construct(string $uri = '')
    {
        $parts = parse_url($uri);
        if (!$parts) {
            throw new \InvalidArgumentException(\sprintf('%sは不正なURIです', $uri));
        }

        $this->scheme = isset($parts['scheme']) ? strtolower($parts['scheme']) : '';
        $this->userInfo = $parts['user'] ?? '';
        if (isset($parts['pass'])) {
            $this->userInfo .= ':' . $parts['pass'];
        }
        $this->host = isset($parts['host']) ? strtolower($parts['host']) : '';
        $this->port = $parts['port'] ?? null;
        # `/`の有無や重複を丸める
        $this->path = isset($parts['path']) ? ('/' . ltrim($parts['path'], '/')) : '';
        $this->query = $parts['query'] ?? '';
        $this->fragment = $parts['fragment'] ?? '';
    }

    public function getScheme(): string
    {
        return $this->scheme;
    }

    public function getAuthority(): string
    {
        $authority = '';

        if ($this->userInfo) {
            $authority = $this->userInfo;
        }
        if ($this->host && $authority) {
            $authority .= '@' . $this->host;
        } elseif ($this->host) {
            $authority .= $this->host;
        }
        if ($this->port) {
            $authority .= ':' . $this->port;
        }

        return $authority;
    }

    public function getUserInfo(): string
    {
        return $this->userInfo;
    }

    public function getHost(): string
    {
        return $this->host;
    }

    public function getPort(): ?int
    {
        return $this->port;
    }

    public function getPath(): string
    {
        return $this->path;
    }

    public function getQuery(): string
    {
        return $this->query;
    }

    public function getFragment(): string
    {
        return $this->fragment;
    }

    public function withScheme(string $scheme): UriInterface
    {
        $scheme = strtolower($scheme);

        $new = clone $this;
        $new->scheme = $scheme;

        return $new;
    }

    public function withUserInfo(string $user, ?string $password = null): UriInterface
    {
        $info = $user;
        if ($password) {
            $info .= ':' . $password;
        }

        $new = clone $this;
        $new->userInfo = $info;

        return $new;
    }

    public function withHost($host): UriInterface
    {
        $new = clone $this;
        $new->host = $host;

        return $new;
    }

    public function withPort($port): UriInterface
    {
        $new = clone $this;
        $new->port = $port;

        return $new;
    }

    public function withPath($path): UriInterface
    {
        $new = clone $this;
        $new->path = $path;

        return $new;
    }

    public function withQuery($query): UriInterface
    {
        $new = clone $this;
        $new->query = $query;

        return $new;
    }

    public function withFragment($fragment): UriInterface
    {
        $new = clone $this;
        $new->fragment = $fragment;

        return $new;
    }

    public function __toString(): string
    {
        $uri = '';

        if ($this->scheme) {
            $uri .= $this->scheme . '://';
        }
        $authority = $this->getAuthority();
        if ($authority) {
            $uri .= $authority;
        }
        $path = $this->getPath();
        if ($path) {
            if (!(str_ends_with($uri, '/') || str_starts_with($path, '/'))) {
                $path = '/' . $path;
            }
            $uri .= $path;
        }
        $query = $this->getQuery();
        if ($query) {
            $uri .= '?' . $query;
        }
        $fragment = $this->getFragment();
        if ($fragment) {
            $uri .= '#' . $fragment;
        }

        return $uri;
    }
}

```

ファクトリーも作成します。
`/app/fw/src/Http/UriFactory.php`に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\UriFactoryInterface;
use Psr\Http\Message\UriInterface;

class UriFactory implements UriFactoryInterface
{
    public function createUri(string $uri = ''): UriInterface
    {
        return new Uri($uri);
    }
}

```

引き続き、`ServerRequest`の作成に入ります。
`/app/fw/src/Http/ServerRequest.php`に、次のように記述してください。
なお、今回はファイル関連の操作に関しては、実装を簡単にするために端折っています。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use O0h\KantanFwLite\Http\Stream;
use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\StreamInterface;
use Psr\Http\Message\UriInterface;

class ServerRequest implements ServerRequestInterface
{
    use MessageTrait;

    private ?string $requestTarget = null;

    public function __construct(
        private string $protocolVersion = '1.1',
        private string $method = 'GET',
        private UriInterface $uri = new Uri(),
        array $headers = [],
        private StreamInterface $body = new Stream(''),
        private array $serverParams = [],
        private array $cookieParams = [],
        private array $queryParams = [],
        private $parsedBody = null,
    ) {
        $this->headers = array_map(fn($value) => (array)$value, $headers);
    }

    public function getServerParams(): array
    {
        return $this->serverParams;
    }

    public function getCookieParams(): array
    {
        return $this->cookieParams;
    }

    public function withCookieParams(array $cookies): self
    {
        $new = clone $this;
        $new->cookieParams = $cookies;

        return $new;
    }

    public function getQueryParams(): array
    {
        return $this->queryParams;
    }

    public function withQueryParams(array $query): self
    {
        $new = clone $this;
        $new->queryParams = $query;

        return $new;
    }

    public function getUploadedFiles(): array
    {
        throw new \LogicException(__METHOD__ . 'は実装されていません');
    }

    public function withUploadedFiles(array $uploadedFiles): self
    {
        throw new \LogicException(__METHOD__ . 'は実装されていません');
    }

    public function getParsedBody()
    {
        return $this->parsedBody;
    }

    public function withParsedBody($data): self
    {
        if (!(
            $data === null ||
            is_array($data) ||
            is_object($data)
        )) {
            throw new InvalidArgumentException('nullかarrayかobjectのみ利用できます');
        }

        $new = clone $this;
        $new->parsedBody = $data;

        return $new;
    }

    public function getAttributes(): array
    {
        return $this->attributes;
    }

    public function getAttribute($name, $default = null)
    {
        return $this->attributes[$name] ?? $default;
    }

    public function withAttribute($name, $value): self
    {
        $new = clone $this;
        $new->attributes[$name] = $value;

        return $new;
    }

    public function withoutAttribute($name): self
    {
        if (!array_key_exists($name, $this->attributes)) {
            return $this;
        }
        $new = clone $this;
        unset($new->attributes[$name]);

        return $new;
    }

    public function getRequestTarget(): string
    {
        if ($this->requestTarget) {
            return $this->requestTarget;
        }

        $target = $this->uri->getPath();
        $query = $this->uri->getQuery();
        if ($query) {
            $target .= '?' . $query;
        }

        return $target ?: '/';
    }

    public function withRequestTarget(string $requestTarget): RequestInterface
    {
        $new = clone $this;
        $new->requestTarget = $requestTarget;

        return $new;
    }

    public function getMethod(): string
    {
        return $this->method;
    }

    public function withMethod(string $method): RequestInterface
    {
        $new = clone $this;
        $new->method = $method;

        return $new;
    }

    public function getUri(): UriInterface
    {
        return $this->uri;
    }

    public function withUri(UriInterface $uri, bool $preserveHost = false): RequestInterface
    {
        $new = clone $this;
        $new->uri = $uri;

        if (!$preserveHost && $uri->getHost()) {
            $host = $uri->getHost();
            $new->headers['Host'] = $host;
        }

        return $new;
    }
}

```

ファクトリーも作成します。
PSR-17で規定されているインターフェイスに加えて、スーパーグローバル変数から自動的に情報を収集して組み立てるヘルパーメソッド、`fromGlobals()`も用意します。

`/app/fw/src/Http/ServerRequestFactory.php`に、次のように記述してください。

```php
<?php

declare(strict_types=1);

namespace O0h\KantanFwLite\Http;

use Psr\Http\Message\ServerRequestFactoryInterface;
use Psr\Http\Message\ServerRequestInterface;

class ServerRequestFactory implements ServerRequestFactoryInterface
{
    public function createServerRequest(string $method, $uri, array $serverParams = []): ServerRequestInterface
    {
        return new ServerRequest(
            method: $method,
            uri: (new UriFactory())->createUri($uri),
            serverParams: $serverParams,
        );
    }

    /**
     * スーパーグローバル変数からServerRequestインスタンスを組み立てて返す
     *
     * @return ServerRequest
     */
    public static function fromGlobals(): ServerRequest
    {
        return new ServerRequest(
            method: $_SERVER['REQUEST_METHOD'],
            uri: (new UriFactory())->createUri($_SERVER['REQUEST_URI']),
            serverParams: $_SERVER,
            cookieParams: $_COOKIE,
            queryParams: $_GET,
            parsedBody: $_POST,
        );
    }
}

```

PSR-7のクラスの実装ラッシュは、これで一段落です。お疲れ様でした！



:::message
ResponseやServerRequestは、実装方法にあまり大きな差が生まれにくい部分です。
また、今回のコードは、実装を完結にするために脆弱な部分が多くあります。
(例えば、URI周りが適切にバリデーションもエスケープもされていません。不正な情報が埋め込まれてしまうリスクがあります。)

そうした手間を避けるため、世の中で流通しているパッケージを活用するのも賢いやり方と言えます。

Packagist上で `psr/http-factory-implementation`という仮想パッケージを検索すると、PSR-17準拠の機能を提供できるパッケージを見つけることが出来ます。

https://packagist.org/providers/psr/http-factory-implementation

その中でも、nyholm/psr7はシンプルで軽量な実装となっており、実用するのも中身を読んでPSR-17/PSR-7への理解を深める教材としてもオススメです。

https://github.com/Nyholm/psr7

:::

## ServerRequestを利用する

今作ったクラスたちを、実際にアプリケーションやFWに組み込んでいきましょう。

今回は、`ServerRequest`の生成と注入の責任を、`index.php`に持たせようと考えています。
まずはここから始めていきましょう。

最初に`Application::run()`が`ServerRequestInterface`を受け付けるようにします。

```diff
--- a/app/fw/src/BaseApplication.php
+++ b/app/fw/src/BaseApplication.php
@@ -7,10 +7,11 @@
 use O0h\KantanFwLite\Http\Emitter;
 use O0h\KantanFwLite\Http\ResponseFactory;
 use O0h\KantanFwLite\Http\StreamFactory;
+use Psr\Http\Message\ServerRequestInterface;
 
 abstract class BaseApplication
 {
-    final public function run(): void
+    final public function run(ServerRequestInterface $serverRequest): void
     {
         $contents = sprintf('Hello, World!<br> %sは、ただいま%sです', date('e'), date('D'));
         $body = (new StreamFactory())->createStream($contents);

```

`index.php`が適切に`run()`を呼べるようにします。
`/app/public/index.php`を、次のように書き換えてください。

```diff
--- a/app/public/index.php
+++ b/app/public/index.php
@@ -5,5 +5,5 @@
 require dirname(__FILE__, 2) . '/vendor/autoload.php';
 
 $app = new \O0h\KiribanBbs\Application();
-
-$app->run();
+$serverRequest = \O0h\KantanFwLite\Http\ServerRequestFactory::fromGlobals();
+$app->run($serverRequest);

```

念のため、ブラウザで今まで通りの画面が表示できるかを確認してみてください。正常に動作しているでしょうか？

これで、KiribanBBSのアプリケーションは「PSR-7の``ServerRequest`」に対応しました！嬉しいですね。

## Actionの作成と利用

「リクエストに応じてコンテンツを作成するロジック」を担う、Actionのレイヤーを作成していきます。
HTTP Messagesと密な関係になるため、クラスの配置としては`O0h\KantanFwLite\Http\Action\`を考えています。

・・・そうすると、HTTPにEmitterやメッセージのファミリーに加えて、更に新しい概念が入ってくることになりますね。段々と、ここまでに作ったHTTP Messageのクラスが増えてきたのが気になります。
そこで、このタイミングで `O0h\KantanFwLite\Http\Message\`を増やして、メッセージとそのファクトリを移動してしまいましょう。

新しいクラス名(FQCN)は、次のようになります。

* `O0h\KantanFwLite\Http\Message\MessageTrait`
* `O0h\KantanFwLite\Http\Message\Response`
* `O0h\KantanFwLite\Http\Message\ResponseFactory`
* `O0h\KantanFwLite\Http\Message\ServerRequest`
* `O0h\KantanFwLite\Http\Message\ServerRequestFactory`
* `O0h\KantanFwLite\Http\Message\Stream`
* `O0h\KantanFwLite\Http\Message\StreamFactory`
* `O0h\KantanFwLite\Http\Message\Uri`
* `O0h\KantanFwLite\Http\Message\UriFactory`





- Request
- Action
  - requestを受け付けて、それを解釈してコンテンツを作成するもの
- まだviewは考えない
