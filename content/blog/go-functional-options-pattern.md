---
title: <span>Functional Options</span> パターンによる柔軟な構造体生成
title_safe: Functional Optionsパターンによる柔軟な構造体生成
highlight: lime
pack: duotone
icon: sliders-horizontal
draft: false
toc: true
comments: false
date: 2026-06-26
category: Go
tags:
  - go
  - design-patterns
---
Goには引数のオーバーロード（同じ名前で異なる引数をとる関数）や関数のデフォルト引数機能がありません。そのため、多数のオプション設定を持つオブジェクトを初期化する際、設計が難しくなることがあります。これを美しく解決するのが `Functional Options` パターンです。

<!--more-->

## 1. 従来の設計とその問題点

例えば、ポートやタイムアウト、TLS設定などを持つサーバー構造体を定義します。

```go
type Server struct {
	Addr    string
	Port    int
	Timeout time.Duration
	TLS     bool
}
```

この初期化関数を作る際、すべてのオプションを引数に渡すと、不要な場合でも呼び出し元が多くの引数を指定する必要があります。

```go
// 呼び出しが面倒
srv := NewServer("127.0.0.1", 8080, 30*time.Second, false)
```

あるいは、`Config` 構造体を渡す手法もありますが、デフォルト値の制御が難しくなるというデメリットがあります。

## 2. Functional Options パターンの実装

このパターンでは、設定を変更するための「関数」をオプションとして渡します。

```go
package main

import (
	"time"
)

type Server struct {
	addr    string
	port    int
	timeout time.Duration
}

// Option は Server の設定を変更する関数型です。
type Option func(*Server)

// オプション設定用の関数群を定義します。
func WithPort(port int) Option {
	return func(s *Server) {
		s.port = port
	}
}

func WithTimeout(timeout time.Duration) Option {
	return func(s *Server) {
		s.timeout = timeout
	}
}

// NewServer はデフォルト値を適用後、任意のオプションを処理します。
func NewServer(addr string, opts ...Option) *Server {
	// デフォルト値の設定
	srv := &Server{
		addr:    addr,
		port:    80,
		timeout: 30 * time.Second,
	}

	// 渡されたオプション（関数）を順次実行
	for _, opt := range opts {
		opt(srv)
	}

	return srv
}

func main() {
	// デフォルト値のまま作成
	s1 := NewServer("127.0.0.1")

	// 特定のポートとタイムアウトを指定して作成
	s2 := NewServer("127.0.0.1", WithPort(8080), WithTimeout(10*time.Second))
}
```

## 3. このパターンのメリット

- **拡張性**: 将来新しいオプションを追加しても、`NewServer` のシグネチャを変更する必要がないため、後方互換性が保たれます。
- **可読性**: 呼び出し時に `WithPort(8080)` のように意図が明示されるため、コードが読みやすくなります。
- **デフォルト値の隠蔽**: 呼び出し元が指定しなかった項目に対して、安全なデフォルト値を簡単に適用できます。

## まとめ

Functional Options パターンは、Go のAPI設計（特にgRPCクライアントやHTTPサーバー等のパッケージ）で非常に広く採用されているデファクトスタンダードです。設定項目が多いモジュールを自作する際は、ぜひこのパターンを検討してみてください。
