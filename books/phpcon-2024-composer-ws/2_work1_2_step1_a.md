---
title: "├ W-①: STEP-1 パッケージ定義ファイルの読み込み"
---

## STEP-1: `composer.lock`ファイルの読み込み

### 課題の背景

まずは最初に、プロジェクトの定義ファイルを読み込む必要があります。

ちなみに、実際のComposerでは `.lock`ファイル単独では処理できません。Tinyposerでは、実装の簡略化のために`.lock`だけで動かしてしまいます。

![](/images/2_work1_2_step1_a/2/json-file-required.png)
_composer.jsonがなくてエラーになっている様子_

では、`.lock`ファイルはどのように読み込めばいいでしょうか？

実は、全く難しいことはありません！拡張子こそちがえど、その内実はJSONファイルです。テキストファイルとして読み込んだうえで、JSONとしてデコードしてあげれば全て完了です！

### 課題

作業用ファイルは `/opt/work/1_install/step1.php`です。

1. `/opt/work/composer.lock`にある`.lock`ファイルを読み込んでください。
2. 読み込んだファイルの内容を連想配列にして、変数 `$lockData `に代入してください。

### 参考: このステップで役に立ちそうな関数etc

- https://php.net/file_get_contents
- https://php.net/fread
- https://php.net/json_decode
