---
title: "├ W-③: STEP-3 PSR-4 クラスローダーの作成"
---

# STEP-3 PSR-4 クラスローダーの作成

## A. 課題について

### 背景

STEP-2で「設定(マップ)」は作ったので、いよいよこれを利用したオートローディングのロジックを作成するときです！

ところで、「なぜ設定ファイルとロジックを持つクラスを分離しているのか？」「いずれにせよ`dump-autoload` フェーズは、何かしらのファイルを自動生成する作業をしている。ファイルをまとめる事もできるはず」と疑問が湧きませんか？

実は、Composerの利用する`ClassLoader` クラスは、オリジナルのファイルをそっくりそのままコピーすることで機能を提供しています。コードの改変は行われないのです。

元となるクラスは https://github.com/composer/composer/blob/2.8.3/src/Composer/Autoload/ClassLoader.php で、ファイルコピーの処理は https://github.com/composer/composer/blob/2.8.3/src/Composer/Autoload/AutoloadGenerator.php#L469 で見ることができます。

## Not To Do！！

おそらくココはハンズオン中に説明すると時間が足りなくなる場所・・・ということで、今回は割愛します。クラスのオートロードに関しては、Composerに留まらない一般的な話題とも言えます。もし興味があったら、ぜひコードを追ってみてください！

また、手前味噌ですが「オートローディングについて」に絞って過去に発表した資料があります。興味があったらご覧ください。

https://speakerdeck.com/o0h/phpcon-okinawa-2024

`work/3_dump-autoload/step3.php`  の作業前の全景はこの通りです。

```php
<?php
if (class_exists('Psr4ClassLoader', false)):
    return;
else:

class Psr4ClassLoader
{

    private array $psr4ClassMap;

    public function __construct(
        string $psr4ClassMapPath,
    )
    {
        $this->psr4ClassMap = require $psr4ClassMapPath;
    }

    public function loadClass(string $class): void
    {
        /* === STEP-3 ココから === */



        /* === STEP-3 ココまで === */
    }
}
endif;
function procedure3_3(string $vendorDirPath): string
{
    $reflector = new ReflectionClass('Psr4ClassLoader');
    $fileName = $reflector->getFileName();
    $startLine = $reflector->getStartLine() - 1;
    $endLine = $reflector->getEndLine();
    $length = $endLine - $startLine;

    $source = file($fileName);
    $classDefinition = implode("", array_slice($source, $startLine, $length));

    $psr4ClassLoaderPath = $vendorDirPath . '/autoload.php';
    file_put_contents(
        $psr4ClassLoaderPath,
        "<?php\n{$classDefinition}\n"
    );

    return $psr4ClassLoaderPath;
}

processDumpAutoload(__DIR__ . '/vendor'); // この内部で `procedure3_3` が呼ばれます
```



穴埋めをしたコードが、次のとおりです。

```php
<?php
if (class_exists('Psr4ClassLoader', false)):
    return;
else:

class Psr4ClassLoader
{

    private array $psr4ClassMap;

    public function __construct(
        string $psr4ClassMapPath,
    )
    {
        $this->psr4ClassMap = require $psr4ClassMapPath;
    }

    public function loadClass(string $class): void
    {
        /* === STEP-3 ココから === */
        $elements = explode('\\', $class);
        while ($elements) {
            $search = implode('\\', $elements).'\\';
            $packageRootPaths = $this->psr4ClassMap[$search] ?? null;;
            if (!$packageRootPaths) {
                array_pop($elements);
                continue;
            }
            $sub = str_replace($search, '', $class);
            $subPath = str_replace('\\', '/', $sub);
            foreach ($packageRootPaths as $packageRootPath) {
                $filePath = "{$packageRootPath}/{$subPath}.php";
                if (file_exists($filePath)) {
                    include $filePath;
                    return;
                }
                array_pop($elements);
            }
        }
        /* === STEP-3 ココまで === */
    }
}
endif;
function procedure3_3(string $vendorDirPath): string
{
    $reflector = new ReflectionClass('Psr4ClassLoader');
    $fileName = $reflector->getFileName();
    $startLine = $reflector->getStartLine() - 1;
    $endLine = $reflector->getEndLine();
    $length = $endLine - $startLine;

    $source = file($fileName);
    $classDefinition = implode("", array_slice($source, $startLine, $length));

    $psr4ClassLoaderPath = $vendorDirPath . '/Psr4lassMap.php';
    file_put_contents(
        $psr4ClassLoaderPath,
        "<?php\n{$classDefinition}\n"
    );

    return $psr4ClassLoaderPath;
}

processDumpAutoload(__DIR__ . '/vendor'); // この内部で `procedure3_3` が呼ばれます
```

