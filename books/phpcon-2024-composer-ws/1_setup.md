---
title: "ハンズオンを実施するための準備"
---

## 用意するもの

本ハンズオンに参加するにあたって、参加者は次の要件を満たしている必要があります。

1. Dockerコンテナを利用可能な環境が用意できる
2. GitHubアカウントを持っている

Dockerに関する内容は、本書のスコープ外となります。予め、ご自身にあったものを用意してください。
GitHubアカウントは、パッケージのコンテンツを取得する際にGitHub APIを利用する関係で必要となります。これは実際のComopserでも同様の挙動を見せているのを、模倣するものです。

### GitHub OAuth access tokenの取得

APIを叩くためのトークンを取得する必要があります。具体的に言えば、Public RepositoryのコンテンツにGETアクセスが可能にするものです。

GitHubにログインした状態で、次のURLから「New personal access token (classic)」を生成してください。

URL: https://github.com/settings/tokens/new?scopes=public_repo&description=Phpcon-2024-Tinyposer

※ Note, Expirationは任意のものに変更して構いません。



**(1) scopeのチェックを確認してください**

![](/images/1_setup/1/setup-token.png)
 *public_repo にチェック*

**(2) ページ下部の「Generate Token」ボタンを押してください**

![](/images/1_setup/1/setup-token-generate.png)
*ボタンを押してacess tokenの生成*

**(3) 生成された access tokenをコピーしてください**

この内容を後ほど利用します。なお、表示されているaccess tokenを後から再確認することはできなません。コピーする前にページを離れたり、リロードしないように気をつけてください。

:::message
再生成は可能です。万が一コピーする前に見失ってしまっても、慌てないでください
:::

![](/images/1_setup/1/setup-token-complete.png)
*忘れないように、表示されているaccess tokenを保存*



## 環境構築

本ハンズオンでは、題材となるコードや開発環境が課題用レポジトリとして公開されています。まずは、GitHubから課題用レポジトリをクローンしてください。

https://github.com/o0h/phpcon-2024-workshop-composer-quick

完了後、レポジトリのルートディレクトリにある `runtime.env.example` をコピーして、 `runtime.env` という名前で保存してください。  
`runtime.env` に、先ほど取得したGitHub OAuth トークンを書き込みます。  

```
GITHUB_OAUTH_TOKEN=[取得したトークン]
```

という形式で、セットしてください。

### 動作確認

ルートディレクトリ上で、次のコマンドを実行してください。Dockerコンテナ内ではなく、ホスト上での操作になります。

```sh
make init
```

正常に完了したら、作業に必要な要件を満たしているかをチェックします。  
ルートディレクトリ上で、次のコマンドを実行してください。

```sh
make test
```

`💯 おめでとうございます！全ての要件を満たしています！` と表示されたら、環境構築は完了です。

![](/images/1_setup/1/setup-finish.png)
*無事にチェックが通った際の画面*



## 課題用レポジトリのディレクトリ構成

レポジトリの全体構成は以下のとおりです。

```
.
├── Dockerfile
├── LICENSE
├── Makefile
├── README.md
├── compose.yaml
├── example
│   ├── 1_install
│   ├── 2_require
│   ├── 3_dump-autoload
│   ├── bootstrap.php
│   ├── composer.json
│   ├── composer.lock
├── requirement-check.php
├── runtime.env.example
└── work
    ├── 1_install
    ├── 2_require
    ├── 3_dump-autoload
    ├── bootstrap.php
    ├── composer.json
    ├── composer.lock
    ├── helper.php
    └── staffroom
```

簡単に、各ファイル・ディレクトリの役割を説明していきます。

### 環境構築用ファイル群

Dockerコンテナを利用するために、次のファイルがルートディレクトリに含まれています

* Dockerfile
* compose.yaml

### requirement-check.php

環境構築後に、ハンズオンを進めるための要件を満たしているかを診断するスクリプトです。

### runtime.env.example | runtime.env

コンテナ内で利用する環境変数を管理する  

### work

フェーズごとに、ワークのための課題やリソースが配置されています。  

#### work/**/step*.php

ステップごとに分離された、実習用のファイルです。

本ハンズオンで利用するファイルになります。コレ以外のファイルは、(envファイルを除いて)基本的には触らずに済むようになっています。

#### example/**/exercise/step*.php

ワークの実装例となるファイルです。