---
title: <span>go:embed</span> を使った静的アセットのシングルバイナリ化
title_safe: go:embedを使った静的アセットのシングルバイナリ化
highlight: orange
pack: duotone
icon: box-open
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - static-assets
---
Go 1.16 から導入された `embed` パッケージを使うと、HTMLテンプレート、CSS、画像、SQLファイルなどの静的アセットをコンパイル時にバイナリの中に直接埋め込むことができます。これにより、サーバーへのデプロイが「単一のバイナリファイルを配置するだけ」で完結するようになります。

<!--more-->

## 1. ファイルを文字列またはスライスとして埋め込む

最もシンプルな例として、単一のテキストファイルを文字列やバイトスライスとして埋め込みます。

```go
package main

import (
	_ "embed"
	"fmt"
)

//go:embed version.txt
var version string

func main() {
	fmt.Println("App Version:", version)
}
```

`//go:embed` ディレクティブの直後の変数に内容が自動的に代入されます。

## 2. ディレクトリ全体を FS として埋め込む

Webサーバーのテンプレートなど、複数のファイルをまとめて埋め込むには `embed.FS` (ファイルシステムインターフェース) を利用します。

```go
package main

import (
	"embed"
	"html/template"
	"net/http"
)

// templates ディレクトリ内のすべての .html ファイルを埋め込む
//go:embed templates/*.html
var templateFS embed.FS

func main() {
	// embed.FS は html/template パッケージと親和性があります
	tmpl := template.Must(template.ParseFS(templateFS, "templates/*.html"))

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		tmpl.ExecuteTemplate(w, "index.html", nil)
	})

	http.ListenAndServe(":8080", nil)
}
```

## 3. 注意点と制限事項

`go:embed` を使用する際には、いくつかルールがあります。

1. **パッケージレベルの変数のみ**: ローカル変数には適用できません。必ずパッケージレベル（グローバル）の変数である必要があります。
2. **相対パスの制限**: 埋め込むファイルパスは、そのGoソースファイルと同じディレクトリ、またはそのサブディレクトリ内である必要があります。親ディレクトリ (`../`) や絶対パスの指定はできません。
3. **インポートが必要**: `embed` パッケージ自体のインポートが必要です。使わないように見えても `import "embed"`（または副作用のための `_ "embed"`）が必要です。

## まとめ

`go:embed` の登場により、Dockerコンテナを作成する際も静的ファイルを別でコピーする手間がなくなり、軽量でポータブルな配布が可能になりました。デプロイの複雑さを下げるために非常に重宝する機能です。
