---
title: <span>go/ast</span> によるソースコード解析と静的解析ツール自作
title_safe: go/astによるソースコード解析と静的解析ツール自作
highlight: teal
pack: duotone
icon: sitemap
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - static-analysis
  - ast
---
Goのコンパイラフロントエンドや静的解析エコシステムは非常にオープンであり、標準ライブラリ `go/parser` や `go/ast` を使うことで、簡単にソースコードをパースして抽象構文木（AST: Abstract Syntax Tree）を取得できます。今回はその基礎と、静的解析ツールの作り方を解説します。

<!--more-->

## 1. AST (抽象構文木) とは？

ASTは、ソースコードの構造を木構造のデータとして表現したものです。変数定義、関数宣言、条件分岐などの文法要素がノードとして表現されます。これを利用して、特定の命名規則の違反やバグの原因となるパターンの検出が行えます。

## 2. コードをパースしてASTを取得する

以下のコードは、Goのソースファイルを読み込み、そこに含まれる関数宣言の名前をすべて出力する例です。

```go
package main

import (
	"fmt"
	"go/ast"
	"go/parser"
	"go/token"
)

func main() {
	src := `
package main
func Hello() { println("Hello") }
func world() { println("World") }
`

	fset := token.NewFileSet()
	// 文字列からパースを実行
	f, err := parser.ParseFile(fset, "main.go", src, 0)
	if err != nil {
		panic(err)
	}

	// ASTのノードをトラバース（巡回）する
	ast.Inspect(f, func(n ast.Node) bool {
		// 関数宣言のノードを探す
		funcDecl, ok := n.(*ast.FuncDecl)
		if ok {
			fmt.Println("Found function:", funcDecl.Name.Name)
		}
		return true
	})
}
```

## 3. 実践：非推奨パッケージ関数の検知

例えば、非推奨となった `ioutil` パッケージ（Go 1.16から非推奨）の呼び出しを検出する簡易静的解析ルールを作ることができます。

```go
ast.Inspect(f, func(n ast.Node) bool {
	call, ok := n.(*ast.CallExpr)
	if !ok {
		return true
	}
	
	// 関数の選択式 (例: ioutil.ReadFile) かを確認
	sel, ok := call.Fun.(*ast.SelectorExpr)
	if ok {
		ident, ok := sel.X.(*ast.Ident)
		if ok && ident.Name == "ioutil" {
			fmt.Printf("Warning: avoid using deprecated 'ioutil.%s'\n", sel.Sel.Name)
		}
	}
	return true
})
```

## まとめ

Goは言語仕様がシンプルであるため、ASTのパースやトラバースが非常に分かりやすくなっています。独自のコーディング規約チェッカーや、コードジェネレータ（`go generate` 用のツールなど）を自作する際、`go/ast` は非常に強力な武器になります。
