---
title: "6章: PHPUnitとの対話"
free: true
---

ここから、本格的にお題にアプローチしていきます。
実際にPHPUnitのコードリーディングに挑戦しましょう！！

手始めに、どこでも良いのでテストケース内にブレイクポイントを貼ってください。
なお、設定してあるブレイクポイントについては、`Bookmarks`から確認できます。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-15-56.png)
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-15-22.png)

まずは、「具体となるテストケースが、どこから呼ばれているのか」を遡っていきます。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-22-33.png)
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-22-47.png)

どうやら、`Framework\TestCase::runTest()`が実行元らしいです。
しかし・・・

```php
 $testResult = $this->{$this->methodName}(...$testArguments);
```

動的なメソッド呼び出しですね！ちょっと複雑です。

この`methodName`の内容を知る必要がありますが、フレームにおける一時変数の確認ができると容易です。

`$this`のプロパティを展開して
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-25-37.png)

`->methodName`を探してみてください
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-26-15.png)

今回の設定だと、`testLargeDotenvLoadsEnvironmentVars` ということで、テストケース名(メソッド名)が入っていることが確認できました！
「テストケースの実行を行う」のが、「テストクラスのオブジェクトに、定義されているメソッドの呼び出し」で実現されていることが分かります。なんてことない、オブジェクト指向プログラミングですね！

やや余談ですが、プロパティや変数を右クリック -> `Copy Value`で値をクリップボードにコピーできます。
しばしば便利ですので、覚えておいて損はないでしょう
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-09-28-27.png)

さて、「テストケース(メソッド名)が確定している」ということは、「テストケース(の一覧)を見つけ出した人は更に外側にいる」という判断ができます。
そんな理由で、更に遡っていきましょう！

手前のフレームを見ても、どうやら同じクラス名っぽい感じです。すなわち、`methodName`の指定が終わった `$this`として、同じ状態が続いているのではないか？と推測できます。
そんな仮説を思い浮かべながら、確認をしてみましょう。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-29-48.png)

予想通りです！！

もう1つ遡ると、レシーバが `$test` になっています。  
この `$test` が、先程(この次のフレーム)の `$this` になっていたことが分かります。  
いちおう `methodName` も確認して、その手前のフレームに遡りましょう。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-33-43.png)

この `$test` が、どこで・どうやって作られているのかが気になってきましたね。

いくつか遡っていって、 `\PHPUnit\Framework\TestSuite::collect()` のフレームにたどり着きます。  
ここで初めて `$test` が出てきました！

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-45-38.png)

コードを読み解く際に、「関係ない」と思った部分はブロック単位で折りたたむことができます。
行番号の右隣にあるアイコンをクリックしてください
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-46-56.png)
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-47-57.png)

どうやら、 `$test` は `foreach ($this as $test) {` で作られているようです。  
・・・なぜ、このオブジェクトはぐるぐると回すと TestCaseインスタンスが取り出せるのでしょうか？
PHPの魔法が隠されていそうだな！と思った際には、Copilotさんに聞いてみます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-52-09.png)

> TestSuiteクラスはIteratorAggregateインターフェースを実装しており、getIteratorメソッドを通じてイテレータを提供します。

どうやら、実装されているInterfaceとそのAPIに秘密があるみたいです。
ここで、一旦「このクラスが何を実装・継承しているのか」を簡単に確認しておければ、この後のコードの解釈にも有利になるかも知れません。

継承ツリーを見てみます。
PhpStormには、そうした情報をダイアグラムで表示する機能が備わっているので、使ってみましょう。

クラス名を右クリック > Diagrams -> Show Diagram Popup を開きます。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-55-25.png)

このようなダイアグラムがでてきて、奥深くには `Traversable` があるのが分かります。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-56-10.png)

(余談: 右クリックでPlantUML等の記法にexportできるぞい！

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-12-57-06.png)

ダイアグラム上の矩形をダブルクリックすると、その定義元を参照できます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-16-37.png)

エディタの左下(※設定で変更可能)には、namespace等の「パンくずリスト」が表示されるのですが、`stubs > Core` となっています。
これは、「PHPの標準である」ことを意味します。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-17-27.png)

PHPの標準インターフェイスなのであれば、まずは公式マニュアルを確認しましょう。

https://php.net/Traversable

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-18-51.png)

> そのクラスの中身が foreach を使用してたどれるかどうかを検出するインターフェイスです。

先程の問、「なぜ$thisをグルグルやるとTestCaseが取り出せるのか？」という答えの半分が、ここにありました。
「foreachで使えるようにする」というInterfadeを実装しているからです。

では、残りの半分、「出てくるのが何故TestCaseなのか」を覗いていきます。

マニュアルには

> IteratorAggregate あるいは Iterator を実装しなければなりません。

と記載されており、先程のダイアグラムやCopilotの回答から、 `IteratorAggregate::getIterator()` が肝だということが分かっています。

そしたら、 `TestSuite::getIterator()` の内容を確認していくのが次のステップとなります。

クラスに実装されているメソッドやプロパティについては、 「Stgructure」の機能を用いて容易に確認できます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-24-37.png)

・・・多い！
ので、「ページ内検索」的なカーソルの移動をしましょう
Structureのエリアにフォーカスを当てて、キーボードを叩けばOKです

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-25-54.png)

どうやら、イテレータを要求されると `new TestSuiteIterator($this);` を返しているようです。
自分自身を引数として生成した、 `TestSuiteIterator` です。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-26-16.png)

このクラスの内容を見ていきましょう。
「cmdを押しながらクリック」で、そのクラスの定義元に移動ができます。

開いてみると、 引数の `tests()` を返しているみたいです。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-27-56.png)

`tests()` は何をしている・・？
簡単な確認方法は、 「Quick Definitation」です。
確認したい内容にカーソルを当てて、 `opt + space` を押すと、定義元のソースコードがポップアップで表示されます。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-29-05.png)

なるほど、 `$this->tests` を返しているだけのシンプルなアクセサだ！！ということが分かりました。

そうすると、次の問題は、このtestsがどのように作られているか・・・

cmd + クリックで、まず `$this->tests` の宣言箇所にジャンプします。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-32-04.png)

更に、ジャンプしたプロパティをcmd + クリックをすることで、今度は「そのプロパティ(やメソッド)を利用している箇所」を引っ張り出すことができます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-33-20.png)
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-33-40.png)

複数の列からなる情報が表示されますが、左から

1. ファイル名
2. クラスやトレイト名
3. メソッド名
4. 当該行

という構成となります。

メソッド名から推測するに、 `setTests()` `addTest()` 辺りが怪しいぞ！！とヤマを張ります。
そんなわけで、この2箇所にブレイクポイントを設定しましょう。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-36-30.png)
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-36-47.png)

メソッド名の行番号にブレイクポイントを設定したときに、通常の「赤丸」表示ではなく、「赤ひし形」に形状が変わります。
これは、「行に対するブレイクポイント」と「メソッドに対するブレイクポイント」の違いです。  
必要に応じて使い分けてみてください。

さて、新しい観点でブレイクポイントを設置したので、いったん頭の中もクリアしてはいかがでしょう？
デバッグ実行のResumeないしStopを実行して、再度デバッグ実行でテストを叩くことにします。

`addTest()` で最初に止まりました！

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-39-52.png)

また 引数の内容を見てみると、すでに「テストケースメソッド」がセットされているのが分かりました。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-40-43.png)

また同じやり方で、手前のフレームに遡っていきます。

`addTestMethod()` では、すでにメソッド名が特定されていました
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-41-42.png)

我々の今回の関心事に対して、 `$method` はとても目を引きますね。
一時変数であっても、その宣言箇所に cmd + クリックでジャンプできます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-53-43.png)

`foreach` のイテレーションで、一時変数 `$method` があるみたいです。
このループの大本であるメソッドも `publicMethodsInTestClass` という名前になっており、かなり答えに近づいてきたのではないか？？？とワクワクします。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-55-13.png)

`filterAndSortMethods()` 最終章を予感させる響き！

ココの引数である `$class` 及び `$filter` の中身が気になって仕方ないので、実際にそれを使っている箇所まで時を進めます。
「Run to cursor」機能が便利です。目的の行の上で右クリック、もしくは行番号の右隣にマウスを当てるとﾌﾜｯとメニューが浮かび上がってくるのを捕まえてください。
(もちろん、`filterAndSortMethods`にブレイクポイントを設置して、デバッグ実行全体をやり直してもOKです)

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-13-56-36.png)

・・・しかしながら、 `$filter = 1` となっています。
`$class->getMethods(1)` も、PHPの標準Reflectionですが、「1でフィルタをする」とは・・・？
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-14-06-54.png)

ステップ実行の良いところは「手軽に何度も実行できる」という所なので、ここでは安直に「最後のmethodsって何が入ってるの」を確認してしまいます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-14-25-02.png)

これでは「どんなメソッドが入っているか」がわかりにくいので、少し工夫します。

「Console」をクリックして、フレーム内で「対話実行」ができます。
![](/images/2-3_reading-phpunit/debug-console.png)
![](/images/2-3_reading-phpunit/debug-console-input.png)

ReflectionMethodクラスのnameを取り出す処理を書いて、実行してみましょう

```php
return array_map(fn ($method) => $method->name, $methods);
```

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-14-48-26.png)

結果が出力されます。

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-14-49-55.png)

これをみると、「対象クラスの中に定義されているメソッド」が抽出されており、まだ「テストケースとなるメソッドに限定する」という動きはしていなさそうです。

すなわち、それぞれのメソッドに対して「テストケースかどうか」を判定数r処理が入ってくると推測できます。

先程のフレームに戻ると、 `isTestMethod()` というソレッポイ名前のメソッドが呼ばれていることに気づきました。
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-14-52-38.png)

`isTestMethod()`の内容は、比較的読みやすいコードになっているのではないでしょうか？
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-14-53-41.png)

念の為、実行しながら確認をしてみましょう。
「何がテストケースで、何がテストケースでないか」です。非テストケースなメソッドは含まれているでしょうか？

今回の場合、ループの中で何度も何度も呼ばれる事がありそうなので、毎度停止されることに煩わしさを覚えるかも知れません。
そんなときは、「実行を一時停止しないで、ログを出力する」という機能を使います。

ブレイクポイントを設定して
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-15-01-58.png)

右クリック -> 「Suspend execution」のチェックを外す
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-15-02-42.png)

すると、拡張メニューが出てくるので、「Evaluate and log」に「ロギングの内容」を入力します
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-15-02-20.png)

例えば、次のような式です。

```php
'====[not test]' . $method->getName()
```

![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-15-04-03.png)

Doneを押して、再度デバッグ実行を行ってください。

実行結果に、ログが残っています！！！
![](/images/2-3_reading-phpunit/2-3_reading-phpunit_2024-05-11-15-05-35.png)

---

ここまでで、今回のお題に適切に回答するための材料が揃ったのではないでしょうか？

とても簡単でしたね！

## 練習問題

メソッド名ではなく、Attributeでテストケースを定義している場合は、どのように判別されているでしょう？
