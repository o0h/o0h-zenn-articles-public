---
title: "4章: テストの実行"
free: true
---

まずは、動作確認も兼ねてテストを実行してみましょう。  
もちろんPhpStormを使います！

`vendor/vlucas/phpdotenv/tests/Dotenv/DotenvTest.php` を例に使いましょう。
(チュートリアル用のレポジトリでは、`vlucas/phpdotenv` パッケージに合わせてPHPUnit構成の設定が行われています)


目当てのクラスを開いて
![](/./images/2-1_run-test/2-1_run-test_2024-05-11-08-14-07.png)

テストケースとなるメソッドが記述されている行 **ではない** 適当なところを右クリックして、`Run 'DotenvTest(PHPUnit)'`をクリックすると・・
![](/./images/2-1_run-test/2-1_run-test_2024-05-11-08-15-18.png)

とっても簡単にテストが実行されます！！
![](/./images/2-1_run-test/2-1_run-test_2024-05-11-08-17-34.png)