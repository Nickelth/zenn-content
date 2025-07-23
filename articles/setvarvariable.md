---
title: "なぜ set var = value は危険なのか──Windowsバッチ変数代入の正しい書き方"
emoji: "⌨"
type: "tech"
topics: ["cmd", "cli", "Windows"]
published: true
---

## 1. はじめに
Windowsのバッチファイルにおける変数代入は、一見単純に見えるが、空白や展開のタイミングによって意図しない動作を引き起こすことがある。本記事では、典型的な書き方である `set var = value` がどのような問題を引き起こすかを検証し、代わりに推奨される `set "var=value"` 形式の利点を紹介する。

## 2. `set var = value` が抱える問題
以下のコードを実行すると：
``` bat:batchfile
set var = hello
echo [%var%]
```
出力は `[ hello]` となる。`var` に代入されたのは ` hello`（先頭に空白付き）であり、意図した値とは異なる。

### 2.1 不正な変数名が定義されるケース
実際には `var `（末尾に空白のある変数名）が定義されているため、`%var%` は空であり、`%var %` のように書かなければ参照できない。

## 3. 正しい書き方：`set "var=value"`
``` bat:batchfile
set "var=hello"
echo [%var%]
```

この形式では、クォートによって左右の空白を防ぎつつ、変数名と値を明確に分離することができる。
また、値の中に空白や特殊文字が含まれていた場合も、正しく処理される。

### 3.1 `set var="value"`が危険な理由
一部の現場では、以下のような形式が好まれる場合がある。
``` bat:batchfile
set var="hello world"
```

この書き方は一見するとクォートによって値が保護されているように見えるが、実際にはクォートが変数の値として含まれる点に注意が必要である。
``` bat:batchfile
set var="hello world"
echo [%var%]
```
このコードは `"hello world"` を出力する。すなわち、変数 `var` の値は `hello world` ではなく `"hello world"` である。

この差は、他のコマンドや条件分岐で致命的な不具合を引き起こす可能性がある。

#### 3.1.1 比較が失敗する
``` bat:batchfile
set var="value"
if %var%==value echo match
```
このような条件分岐は、`"value"` と `value` を比較することになるため成立せず、結果として `match` は表示されない。
また、展開の結果構文が壊れ、`==value` のように解釈されて文法エラーになることもある。

#### 3.1.2 パスや引数として扱うときに失敗する
``` bat:batchfile
set filepath="C:\path\to\file.txt"
type %filepath%
```

このように変数にパスを代入して利用すると、`type` コマンドは `"C:\path\to\file.txt"`（クォート付き）という名前のファイルを探すことになり、ファイルが見つからないというエラーが発生する。

### 3.2 適切な使い分け
``` bat:batchfile
set "filepath=C:\path\to\file.txt"
type "%filepath%"
```
正しい方法は、値にクォートを含めずに代入し、必要に応じて展開時にクォートで囲むことである。この構成であれば、他のコマンドに対しても安定して値を渡すことができる。

## 4. StackOverflowでの実例と解説
StackOverflowでも、この書き方が推奨されている。

>“The space definitively makes a difference in cmd ( set var = value sets a variable %var<space>% to <space>value )”
>– https://stackoverflow.com/questions/15004825/

また、以下のような回答も存在する：

>“use recommended set syntax to avoid them: set "var=value" (note the quotes and their location)”
>– https://superuser.com/questions/1577510/

これらの議論からもわかるように、`set "var=value"` は安全であるだけでなく、意図しない展開や誤動作を防ぐために有効な手法である。

## 5. 遅延展開と組み合わせる場合の注意
`setlocal enabledelayedexpansion` を使用する場合でも、`set "var=value"` の形式は有効である。以下の例を参照：
``` bat:batchfile
setlocal enabledelayedexpansion
set "name=world"
echo !name!
```

## 6. 実運用での適用例
実際のバッチスクリプトにおいては、以下のような形式での変数定義が安全である：
``` bat:batchfile
set "taskname=DataImportTask"
set "logfile=%~dp0log\import.log"
```
特に `path` や `file name` など、空白やバックスラッシュを含む値を扱う際は、クォートの有無によって動作が変わるため注意が必要となる。

## 7. おわりに
`set var = value`や`set var="hello world"` という書き方は、単純に見えてバッチファイルにおける数々の問題の温床となる。
より安全かつ正確なスクリプトを書くためには、`set "var=value"` の形式を採用するべきである。
多少記述が煩雑に見えるかもしれないが、長期的な保守性と予測可能な動作を得る上では、十分に妥当な選択肢といえる。

