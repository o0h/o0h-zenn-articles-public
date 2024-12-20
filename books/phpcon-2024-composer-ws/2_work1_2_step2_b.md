---
title: "├ W-①: STEP-2 <解説>依存パッケージ取得の基礎枠作り"
---

## STEP-2: 実装例とコードの解説

作業前のファイルの全景

```php
assert(isset($lockData));

/* === STEP-2 ココから === */
// foreach ( ・・・

/* === STEP-2 ココまで === */
```

### 1.`$lockData` から、パッケージの情報を取り出してください

例によって簡易な実装で充分なので、フィールド2つ対してそれぞれforeachループを回せば充分でしょう。

```php
foreach ($lockData['packages'] as $package) {
  // TODO: $packageの処理
}

foreach ($lockData['packages-dev'] as $package) {
  // TODO: $packageの処理
}

```



### 2.  取り出した情報に対して、`processPackage()` 関数を呼び出してください

ループ文の中で、指定された関数を呼びます。

```php
foreach ($lockData['packages'] as $package) {
  processPackage($package);
}

foreach ($lockData['packages-dev'] as $package) {
  processPackage($package);
}

```

