---
title: "3章 PSR-4: AutoloaderとCakePHPのディレクトリ構成"
free: true
---

# CakePHPアプリケーションのディレクトリ構成

CakePHPのアプリケーションスケルトン(https://github.com/cakephp/app)は、以下のようなディレクトリ構成になっています。[^current-ver]

```sh
$ tree -L 3 -d
.
├── bin
├── config
│   └── schema
├── logs
├── plugins
├── resources
├── src
│   ├── Command
│   ├── Console
│   ├── Controller
│   │   └── Component
│   ├── Model
│   │   ├── Behavior
│   │   ├── Entity
│   │   └── Table
│   └── View
│       ├── Cell
│       └── Helper
├── templates
│   ├── Error
│   ├── Pages
│   ├── cell
│   ├── element
│   │   └── flash
│   ├── email
│   │   ├── html
│   │   └── text
│   └── layout
│       └── email
├── tests
│   ├── Fixture
│   └── TestCase
│       ├── Controller
│       ├── Model
│       └── View
├── tmp
│   ├── cache
│   │   ├── models
│   │   ├── persistent
│   │   └── views
│   ├── sessions
│   └── tests
└── webroot
    ├── css
    ├── font
    ├── img
    └── js

47 directories
```

[^current-ver]: 4.2.2時点

簡単に、各ディレクトリの役割を見ていきましょう[^cake-dirs] 

* bin: 実行ファイルが配置されます。デフォルトでは 「cakeコマンド」のみが入っています
* config: アプリケーションの設定ファイルが配置されます [doc](https://book.cakephp.org/4/ja/development/configuration.html)
  * schema: CakePHPの機能を利用する際に必要とされるSQLファイルが配置されます(滅多に利用することはありません)。例えば、 `sessions.sql`については[データベースでセッションを管理する際に利用できる](https://book.cakephp.org/4/ja/development/sessions.html#id5)との説明があります
* logs: ファイルでログを管理する場合にログファイルが配置されます
* plugins: (vendorディレクトリに置かない)プラグインが配置されます [doc](https://book.cakephp.org/4/ja/plugins.html#plugin-create-your-own)
* resources: アプリケーション内から参照されるリソースが配置されます。例えば、国際化のための翻訳ファイルが `resources/locales`以下に配置されます
* src: アプリケーションロジックが配置されます
  * Command: CLI実行用の、cakeコマンド(`bin/cake`)から利用するクラスファイルが配置されます
  * Console: CLI実行用の、cakeコマンドを経由せずに単独で利用するクラスファイルが配置されます(Composerイベント(`post-install-cmd`) にフックして利用される`Installer`クラスなど)[^command-and-console]
  * Controller: コントローラークラスが配置されます [doc](https://book.cakephp.org/4/ja/controllers.html)
    * Component: コントローラーレイヤーでの振る舞いを共有するためのコンポーネントクラスが配置されます [doc](https://book.cakephp.org/4/ja/controllers/components.html)
  * Model: モデルクラスが配置されます
    * Behavior: モデル(テーブル)レイヤーでの振る舞いを共有するためのビヘイビアクラスが配置されます [doc](https://book.cakephp.org/4/ja/orm/behaviors.html)
    * Entity: エンティティクラスが配置されます[doc](https://book.cakephp.org/4/ja/orm/entities.html)
    * Table: テーブルクラスが配置されます[doc](https://book.cakephp.org/4/ja/orm/table-objects.html)
  * View: ビュークラスが配置されます[^view-and-templates] [doc](https://book.cakephp.org/4/ja/views.html) 
    * Cell: ビューレイヤーを拡張するセルクラスが配置されます
    * Helper: ビューレイヤーでの振る舞いを共有するためのヘルパークラスが配置されます [doc](https://book.cakephp.org/4/ja/views/helpers.html)
* templates: テンプレートファイルが配置されます
  * Error: Errorコントローラーのアクションに対応するテンプレートが配置されます
  * Pages: Pagesコントローラーのアクションに対応するテンプレートが配置されます
  * cell: セルクラスに対応するテンプレートが配置されます(セルクラスごとに対応する名前のサブディレクトリが設置されます) [doc](https://book.cakephp.org/4/ja/views/cells.html)
  * element: テンプレートの内容を共有するためのエレメントテンプレートファイルが配置されます [doc](https://book.cakephp.org/4/ja/views.html#view-elements)
    * flash: フラッシュメッセージ(セッションを利用して1度だけユーザーに表示するメッセージ)のテンプレートが配置されます [doc](https://book.cakephp.org/4/ja/controllers/components/flash.html)
  * email: Eメール用のテンプレートが配置されます
    * html, text: HTMLメール、プレーンテキストメールのそれぞれに対応するテンプレートが配置されます
  * layout: テンプレートの共通部分を管理するテンプレートレイアウトファイルが配置されます [doc](https://book.cakephp.org/4/ja/views.html#view-layouts)
    * email
* tests: テストに関するファイルが配置されます
  * Fixture: フィクスチャクラスやが配置されます。クラスファイルではないフィクスチャも慣例的にこのディレクトリに配置されます [doc](https://book.cakephp.org/4/ja/development/testing.html#test-fixtures)
  * TestCase: テストケースクラスが配置されます
    * Controller, Model, View: srcディレクトリと同様のディレクトリ構成をとり、テストケースクラスが配置されます
* tmp: 一時ファイルが出力されます
  * cache: キャッシュファイルが出力されます
    * models: テーブルの構造情報などを扱う、モデルキャッシュが出力されます
    * persistent: 国際化対応など、動的に生成された後に保持されるべきキャッシュが出力されます
    * views:  ビューキャッシュが出力されます
  * sessions: セッション管理をファイルで行った場合に、セッション情報が出力されます
  * tests: 特に用途はありませんが、テスト時に一時ファイルを作成する場合の出力先に利用できます
* webroot: ウェブアクセス時に利用される静的ファイルが配置されます
  * css, font, img,  js: 名前の通りで、ウェブページ用のリソースが配置されます。`index.php`以外は、CakePHPの動作ロジックには関係がありません



[^console-and-command]: 基本的には、CommandクラスによってCLIタスクを実装していくのがおすすめです。元々は「Shell」と呼ばれる空間も存在していたところから、3.6時代にデザインの見直しが行われて(責務やライフサイクルを整理することで)設置されたのがCommandです。当時の議論については、[RFC \- Command Classes \(replacing Shell / Tasks\) · Issue \#11137 · cakephp/cakephp](https://github.com/cakephp/cakephp/issues/11137) などで追うことが出来ます。
[^view-and-templates]:CakePHP2の時代には、プレゼンテーションのためのテンプレートが `View`ディレクトリに配置されていましたが、3になったさいにView/templateの分離が行われています
[^cake-dirs]:公式ドキュメントにも記載があるので、参考にしてください https://book.cakephp.org/4/ja/intro/cakephp-folder-structure.html

# PSR-4とディレクトリ・ファイル名

ディレクトリの一覧を見ると、大文字で始まる名前(StudlyCaps)のディレクトリとそうでないディレクトリが混在している事に気づきます。これには明確な規定があります。

ベースとなっているのは、クラスのオートローディングのための規約である **PSR-4** です。

https://www.php-fig.org/psr/psr-4/

PSR-4は、PHPの名前空間とディレクトリ構成・クラスファイルの配置場所について規定します。これによって、色々なFWやライブラリが提供するオートローダーが互換性を持ち、汎用的に利用できるようになります。幅広く適用できるオートローダーが実現できることは、世の中に存在するライブラリ等にとって、様々なプロジェクトやライブラリにシームレスに組み込めるようになることを意味します。
PSR-4の規定において「クラス」と称されるものには、クラス・トレイト・インターフェイスを含みます。具体的には _The term "class" refers to classes, interfaces, traits, and other similar structures._と記述されていますが、ここでいう _other similar structures_とは、最近のPHPでいうとEnumも含まれると解釈できます。

PSR-4では、名前空間について次の内容を規定します。

* FQCN[^fqcn]は以下のような構造になります
  * 必ず「トップレベル」の名前空間を持つ(「ベンダー名」など)
  * 1つ以上のサブ名前空間を持つことができる
  * 必ず終端はクラス名になること
  * 必ず大文字と小文字を区別して扱うこと(case-sensitive)
* ディレクトリとファイル名は以下のような構造になります
  * ベースとなるディレクトリを定め、そこに対応する命名として「名前空間プレフィックス」を備える
  * サブ名前空間とベースディレクトリからの下層パスが一致する
  * 終端のクラス名とファイル名が一致する
    * ※ これにより、1ファイルの中で宣言できるクラスは1つまでとなる

イメージをつかむために、CakePHPのアプリケーションスケルトンのディレクトリ構造と名前空間の対応を見てみましょう

| パス                                              | 名前空間                                                  |
| ------------------------------------------------- | --------------------------------------------------------- |
| src                                               | 名前空間プレフィックスは `\App`                           |
| src/Controller                                    | `\App\Controller`                                         |
| src/Controller/PagesController                    | FQCNは`\App\Controller\PagesController`                   |
| src/Model                                         | `\App\Model`                                              |
| src/Model/Table                                   | `\App\Model\Table`                                        |
| src/Model/Table/UsersTable.php                    | FQCNは`\App\Model\Table\UsersTable`                       |
| tests                                             | 名前空間プレフィックスは `\App\Test`                      |
| tests/TestCase                                    | `\App\Test\TestCase`                                      |
| tests/TestCase/Controller                         | `\App\Test\TestCase\Controller`                           |
| tests/TestCase/Controller/PagesControllerTest.php | FQCNは`\App\Test\TestCase\Controller\PagesControllerTest` |

今回の場合、「トップレベル」は `\App`であり、`src`ディレクトリには`\App`を、`tests`ディレクトリには`\App\Test`プレフィックスを適用している構造になります。その下は、サブ名前空間やクラス名をそのまま利用したとおりです。
ディレクトリ構造やディレクトリ名、ファイル名が名前空間やFQCNと対応していることが分かるのではないでしょうか。ベースディレクトリや名前空間プレフィックスの指定に関しては、次章で詳しく扱います。

こうした「クラスを配置するディレクトリ」が主に「StudlyCapsになっているディレクトリ」になっています。名前空間の要素について、PSR-4では「アルファベットは大文字と小文字を利用することが出来る」とは明示しているものの、規則についての規定はありません。慣習的なものとして、(クラス名と同様に)StudlyCapsが用いられることが多いです。
例えば、`tmp/cache`ディレクトリや`templates/element`などは、ディレクトリ名が小文字になっています。これらのディレクトリには、オートローディングの対象となるようなクラスファイルを格納することが目的にないため、敢えて異なる命名のスタイルになっているのです。それと同時に、`templates/Pages`などは、クラスとは関係ないディレクトリなのにStudlyCapsになっています。これは、「コントローラー名とテンプレートの対応を分かりやすくする」のが狙いかと思われます。CakePHPは「設定より規約」のFWであるため、こうしたルールも尊重していくことが大事です。

このように、ディレクトリの命名1つをとっても「何をしたいか」の意図が反映されており、プロジェクト全体の一貫性を保つための工夫となっているのです。

もう1つの例として、CakePHPのFW本体の内容を見てみましょう。

| パス                                      | 名前空間                                         |
| ----------------------------------------- | ------------------------------------------------ |
| src                                       | 名前空間プレフィックスは `\Cake`                 |
| src/ORM                                   | `\Cake\ORM`                                      |
| src/ORM/Behavior                          | `\Cake\ORM\Behavior`                             |
| src/ORM/Behavior/CounterCacheBehavior.php | FQCNは `\Cake\ORM\Behavior\CounterCacheBehavior` |
| src/ORM/Behavior/TimestampBehavior.php    | FQCNは `\Cake\ORM\Behavior\TimestampBehavior`    |
| src/ORM/Behavior.php                      | FQCNは `\Cake\ORM\Behavior`                      |
| src/ORM/Entity.php                        | FQCNは `\Cake\ORM\Entity`                        |
| src/ORM/Table.php                         | FQCNは `\Cake\ORM\Table`                         |

今度は、「トップレベル」は`\Cake`となりました。また、「大文字と小文字を任意に組み合わせられる」とされている通り、`ORM`のように連続で大文字を利用した名前空間を設定しても何も問題はありません。
同時に、サブ名前空間としての`Cake\ORM\Behavior`とFQCNとしての `Cake\ORM\Behavior`が同時に存在していますが、支障はありません。あくまで「存在しないクラスを自動的に読み込む」というオートローディングの仕組みを効率よく利用するための規定がPSR-4であり、「(サブ)ディレクトリを読み込みに行く」という状況は想定する必要がないからです。ここで、「FQCNの終端はクラス名になること」という仕様の意味がでてきます。

CakePHPを利用したアプリケーション開発では、開発者は任意のディレクトリを設置することが可能です。元々のスケルトンにて用意されている構成要素だけでは、やりたい設計に不十分なケースもあるでしょう(例えば、「サービス層」のディレクトリはデフォルトでは含まれていません)。その場合には、「PSR-4を意識してディレクトリ構成をデザインする」「ディレクトリ名をStudlyCapsにするか、snake_caseにするか」といった点を意識しましょう。そうすれば、プロジェクトに参加する開発者に見通しの良さを提供できるようになります。

こうした命名規則に従った世界観では、どのように成し遂げたかったオートローディングが実現されるでしょうか？興味を持った方は、PHP-FIGのホームページに掲載されている「PSR-4 Example Implementations」を読んでみてください。実に単純な仕組みで、便利なオートローディングが実現されることが明らかになるはずです。

https://www.php-fig.org/psr/psr-4/examples/

[^fqcn]: 「完全修飾クラス名」のことで、省略せずに名前空間を含むクラス名を意味します。 例えばCakePHPのContorllerクラスで言うと、`\Cake\Controller\Controller`のことです。

# PSR-1とファイルの構成

PSR-4はオートローディングのための規約ですが、PSRシリーズではクラス名やファイル内容に関して規定をした **PSR-1** があります。

https://www.php-fig.org/psr/psr-1/

日本語でのリソースは、インフィニットループ社のブログにて公開されている翻訳記事があります。この投稿の後にPSR-1自体が少々更新されているため[^psr0-amendment]に現状のものと相違もありますが、大筋を掴む上では問題ありません。正確な情報はPHP-FIGのサイトで確認をとってください

https://www.infiniteloop.co.jp/docs/psr/psr-1-basic-coding-standard.php

この規約は「基本的なコーディング規約」を定めたものですが、コーディングの「スタイル」を定めたものではありません(スタイルに関する規約は別に存在します)。

その中でも、PSR-4と特に密接に関わるのは「副作用」「名前空間とクラス名」のセクションです。

まず、PHPのファイルは「副作用を持つものと、そうでないものを明確に区別するべき」とされています。
副作用とは、関数の実行や何らかの出力を行ったり、グローバル変数の変更や設定を変更したり、明示的な他ファイルの読み込み(`require`や`include`)などです。その逆に、「副作用を持たない」とは、クラスや関数・定数などの定義を行う( _declare new symbols_)ことです。
副作用のあるファイルをそうでないファイルと区別しておくことで、「あるクラスを利用したかったのに意図せぬ処理が実行された」という自体を避けることが出来ます。これは、PSR-4のようなオートローディングを前提としていく上での安心感に繋がります。

名前空間とクラス名のセクションでは、「名前空間は、PSRのオートローディング規約(PSR-0/PSR-4)に従わなければならない」とされています(ただし、PSR-0については既に[非推奨とされている](https://github.com/php-fig/fig-standards/pull/341)ので、これから何かを作るのであれば実質的にPSR-4の一択となります)。そして、クラス名については「StudlyCapsにしなければならない」とされています。

これらを踏まえて、「クラスを宣言するファイルは、それ以外の仕事はしない」「FQCNの末端はとそのファイル名はStudlyCapsになる」という事になります。PSR-4だけを見ると明確にされていない内容ですが、クラス名・ファイル名・ファイル内に含めていいものといった観点に一般的な基準が設けられることになります。

「クラス[^class-is]を宣言するファイル」については決め事がある一方で、副作用を持つファイルの命名に関する取り決めはありません。慣習的に、これらは `snake_case`で命名されることが多いように思います。副作用を持つファイルだけではなく、関数を定義するファイル・定数の定義をまとめたファイルについても同様です。CakePHPを例に取れば、関数を定義する `functions.php`やデバッグ系の関数や「時間定義定数」を定義する `basics.php`といったファイルが存在します[^functions]。

[^psr0-amendment]: 主だったもので言うと、PSR-0の位置付けの変更による修正があります https://github.com/php-fig/fig-standards/pull/247
[^class-is]: PSR-4での用語と同様に、intarface/trait/enumを含みます
[^functions]:グローバル関数および定数については、 https://book.cakephp.org/4/ja/core-libraries/global-constants-and-functions.html で確認できます
