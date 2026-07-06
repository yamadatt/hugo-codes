---
title: Goで始める <span>WebAssembly (WASM)</span> と WASI の基本
title_safe: Goで始めるWebAssembly (WASM) と WASI の基本
highlight: sky
pack: duotone
icon: globe
draft: false
toc: true
comments: false
date: 2026-06-09
category: Go
tags:
  - go
  - webassembly
  - wasi
---
WebAssembly (WASM) はブラウザ上でネイティブコードに近い速度でコードを動かす技術ですが、近年はサーバーサイドでポータブルなコンテナ技術としても注目を集めています。今回は、GoコードをWASM/WASI向けにビルドして動かす基本手順を解説します。

<!--more-->

## 1. ブラウザ向けの WebAssembly コンパイル

Goは標準で WebAssembly へのコンパイルをサポートしています。ターゲットOSを `js`、アーキテクチャを `wasm` に指定します。

```sh
GOOS=js GOARCH=wasm go build -o main.wasm main.go
```

ブラウザでこのファイルを読み込んで実行するには、Goが提供するJavaScriptヘルパーファイル (`wasm_exec.js`) が必要です。

```sh
# ヘルパーファイルをコピー
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" .
```

これをHTML側から以下のように読み込んで初期化します。

```html
<script src="wasm_exec.js"></script>
<script>
  const go = new Go();
  WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject).then((result) => {
    go.run(result.instance);
  });
</script>
```

## 2. WASI (WebAssembly System Interface) 向けビルド

ブラウザの外（例えば Node.js や Wasmtime などのランタイム）でファイルI/Oやネットワークアクセスを行うために、WASI というシステム呼び出し規格が整備されています。
Go 1.21 から、標準で WASI ターゲットのビルド (`GOOS=wasip1`) が可能になりました。

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello WebAssembly from WASI!")
}
```

以下のコマンドで WASI 形式のバイナリを出力します。

```sh
GOOS=wasip1 GOARCH=wasm go build -o main.wasm main.go
```

これを `wasmtime` などのランタイムを使って実行できます。

```sh
wasmtime main.wasm
# 出力: Hello WebAssembly from WASI!
```

## まとめ

GoのWASM対応は着実に進化しており、特に `wasip1` ターゲットの追加によってブラウザ以外のサーバーサイドやエッジサーバー（Cloudflare Workersなど）でも動作するようになりました。ポータブルでセキュアなアプリケーション配布手段として、今後の展開が非常に期待される技術です。
