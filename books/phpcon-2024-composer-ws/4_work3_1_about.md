---
title: "├ W-③: 実装の流れ"
---

- ## dump-autoloadフェーズの概念

  まずは「元ネタ」となるComposerの仕組みについて整理しましょう。大まかに、次の仕事が含まれてます。

  1. インストール済みのパッケージ情報を収集
  2. パッケージ情報の `autoload`フィールドを取得[^composer-schema-autoload]
  3. `PSR-4` `PSR-0` `Classmap` `Files` のそれぞれのオートロード方式について、該当するファイルパスやキー(FQCNやnamespace)の対応表の作成
  4. Composer自身が雛形を持っているClassLoaderをロジックを `vendor/composer` 以下に配置
     * この内部で、作成済みの対応表を読み込む
  5. オートロードファイル (`vendor/autoload.php`)も作成する

  これを簡易にしたものを、WORK−③で扱っていきます。

  

  [^composer-schema-autoload]: https://getcomposer.org/doc/04-schema.md#autoload

  https://speakerdeck.com/o0h/phpcon-okinawa-2024

  ## WORK-②での実装の流れ

  このワークは、次の4ステップに分かれています。

  1. 各パッケージの.jsonを読み込んでpsr-4用の名前空間マップの作成
  2. 各パッケージの.jsonを読み込んでイーガーロード対象とするファイルのリストの作成
  3. psr-4用のクラスローダーの作成
  4. オートロード設定用のファイルの作成

  それでは、次のページからTinyposerの「dump-autoload」処理を実装していきましょう！