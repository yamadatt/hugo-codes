---
title: Goの <span>interface</span> 設計：暗黙的実装と疎結合の実現
title_safe: Goのinterface設計：暗黙的実装と疎結合の実現
highlight: pink
pack: duotone
icon: code-merge
draft: false
toc: true
comments: false
date: 2026-06-22
category: Go
tags:
  - go
  - interface
  - clean-code
---
Goのインターフェース設計は、他の多くの言語（JavaやTypeScriptなど）とは異なる最大の特徴を持っています。それが「ダックタイピング（暗黙的実装）」です。今回は、Goにおける効果的なインターフェースの設計方法と注意点について整理します。

<!--more-->

## 1. 暗黙的実装とは？

Goでは、`implements` キーワードを明示的に宣言する必要はありません。構造体が必要なメソッドをすべて実装していれば、自動的にそのインターフェースを満たしていると判定されます。

```go
type Logger interface {
	Log(message string)
}

type ConsoleLogger struct{}

// Log メソッドを実装するだけで Logger を満たしたことになる
func (c ConsoleLogger) Log(message string) {
	println(message)
}
```

これにより、インターフェースを定義する側と、それを実装する側が完全に分離され、サードパーティのパッケージが提供する構造体であっても自作インターフェースに当てはめることが可能になります。

## 2. インターフェースは「利用側」が定義する

Goの優れた設計アプローチは、**「具象クラスを提供する側ではなく、それを利用する側がインターフェースを定義する」**という原則です。

例えば、あるデータベースモジュールがあるとき、DB側で巨大なインターフェースを提供するのではなく、ビジネスロジック側で「これだけのメソッドが必要」という最小限のインターフェースを定義します。

```go
// ビジネスロジック側で定義する
type UserReader interface {
	FindUser(id string) (*User, error)
}

type UserService struct {
	repo UserReader // 最小限の依存のみを受け取る
}
```

これにより、ユニットテスト時にモック（Mock）を作成するのが非常に容易になります。

## 3. インターフェースの大きさを最小限にする

Goの格言に **「Interfaceが小さくなるほど、その抽象は強力になる (The bigger the interface, the weaker the abstraction)」** というものがあります。
標準ライブラリの `io.Reader` や `io.Writer` はメソッドが1つしかありませんが、それゆえに世界中のあらゆるストリームデータを共通の方法で繋ぐことができます。可能であれば、メソッドは1〜2個程度に留めるのが理想です。

## まとめ

Goのインターフェースは、疎結合な設計を実現するための最重要ツールです。実装側に `implements` を書かせず、利用側が必要な分だけインターフェースを小さく定義するアプローチを心がけましょう。
