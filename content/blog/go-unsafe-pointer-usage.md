---
title: <span>unsafe.Pointer</span> による極限のメモリ操作と注意点
title_safe: unsafe.Pointerによる極限のメモリ操作と注意点
highlight: red
pack: duotone
icon: triangle-exclamation
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - unsafe
  - low-level
---
Goはメモリセーフな言語であり、通常のコードではポインタ演算や不正な領域へのアクセスは禁止されています。しかし、標準パッケージ `unsafe` を利用すると、コンパイラの型検証をバイパスして直接メモリを操作できます。今回はその使い方と多大なリスクについて解説します。

<!--more-->

## 1. `unsafe.Pointer` とは？

`unsafe.Pointer` は、任意の型へのポインタを格納できる特別な型です。Goの通常の型付きポインタ（例: `*int`）から `unsafe.Pointer` を経由して、全く異なる型（例: `*float64`）へキャストすることができます。

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var i int64 = 42
	
	// int64 のメモリ表現を直接 float64 として解釈する（危険なキャスト）
	f := *(*float64)(unsafe.Pointer(&i))
	
	fmt.Println(f) // メモリの中身がそのまま浮動小数点として評価される
}
```

## 2. 応用：コピーフリーな文字列・スライス変換

以前のGoでは、文字列（`string`）とバイトスライス（`[]byte`）のキャスト時にはメモリのコピーが発生していました。これを `unsafe` を使ってゼロコピー化するテクニックがパフォーマンス重視のライブラリ等でよく使われていました。

```go
// Go 1.20 より前でよく使われていたゼロコピー変換の例
func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

※現在（Go 1.20以降）は、標準で `unsafe.String` や `unsafe.Slice` などのより安全なヘルパー関数が追加されています。

## 3. なぜ「unsafe（危険）」なのか？

`unsafe` パッケージの使用には以下の大きなリスクがあります。

1. **GCとの干渉**: GoのGCはオブジェクトのメモリ位置を動かすことがあります。`unsafe.Pointer` を整数型（`uintptr`）にキャストしてアドレス計算を行っている最中にGCが走ると、無効なアドレスを指してしまいパニックやメモリ破壊が発生する可能性があります。
2. **互換性の崩壊**: Goの標準ライブラリやコンパイラ仕様が変わった際、`unsafe` を使っているコードは即座に動作しなくなる危険性があります（Goの互換性保証の対象外です）。

## まとめ

`unsafe` パッケージは、一部の超高速I/Oライブラリやシリアライザなど、どうしてもアロケーションを削減してパフォーマンスを極限まで高める必要がある場合のみ使用されます。原則として通常のアプリケーションコードで使うべきではなく、バグの温床となるため十分に注意しましょう。
