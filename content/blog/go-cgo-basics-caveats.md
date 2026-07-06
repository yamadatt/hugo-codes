---
title: <span>Cgo</span> の基本と呼び出しオーバーヘッドの理解
title_safe: Cgoの基本と呼び出しオーバーヘッドの理解
highlight: slate
pack: duotone
icon: low-vision
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - cgo
  - low-level
---
Goは既存のC言語ライブラリを再利用するための仕組みとして `Cgo` を提供しています。非常に強力で便利な機能ですが、GoランタイムとC言語環境の境界を超える際には少なくないオーバーヘッドが存在します。今回はCgoの基本と注意点について解説します。

<!--more-->

## 1. Cgo の最もシンプルな例

Goコード内に `import "C"` を記述し、その直前のコメント（プリアンブル）にC言語コードを書くことで、GoからCの関数を直接呼び出すことができます。

```go
package main

/*
#include <stdio.h>
#include <stdlib.h>

void helloFromC() {
    printf("Hello from C world!\n");
}
*/
import "C"

func main() {
	// Cの関数を呼び出す
	C.helloFromC()
}
```

## 2. メモリ管理の落とし穴

C言語の世界（ヒープ）で確保したメモリは、Goのガベージコレクタ（GC）の管理対象外となります。そのため、Go側から文字列などをCに渡して変換した場合、明示的に `C.free` を呼んでメモリを解放しないとメモリリークを引き起こします。

```go
package main

/*
#include <stdlib.h>
*/
import "C"
import "unsafe"

func PrintText(text string) {
	// Goの文字列をCのchar*（メモリ確保される）に変換
	cStr := C.CString(text)
	
	// 関数の終了時に必ず解放するようdeferを呼ぶ
	defer C.free(unsafe.Pointer(cStr))

	// Cのライブラリに渡す処理...
}
```

## 3. Cgoの呼び出しオーバーヘッド

Cgoの呼び出しは、通常のGoの関数呼び出しに比べて非常に低速です。その理由は以下の通りです。

- **スタックの切り替え**: Goのゴルーチンは極小の可変スタック（数KB）で動いていますが、Cの関数を実行するにはCの標準スレッドスタック（通常数MB）に切り替える必要があります。
- **スケジューラーの調整**: Cのコード実行中はGoのGCやスケジューラーがスレッドをブロック状態として管理する必要があるため、Goランタイムとの同期コストが発生します。

実際のベンチマークでは、単なる数数値の加算処理であっても、Cgo経由の呼び出しはGoネイティブの呼び出しに比べて **数倍から数十倍** の時間がかかります。

## まとめ

CgoはC言語の資産を活用できる重要な機能ですが、「パフォーマンスの低下」「クロスコンパイルの困難さ（Cコンパイラが必要になる）」「メモリリークの危険性」という代償が伴います。パフォーマンスを極限まで求める場合は、できる限りGoでのピュアな再実装を優先し、どうしても必要な場合のみCgoの境界をまたぐ回数を最小限にする設計にしましょう。
