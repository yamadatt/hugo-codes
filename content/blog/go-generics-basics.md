---
title: Goの <span>Generics</span> の基本と実用パターン
title_safe: GoのGenericsの基本と実用パターン
highlight: blue
pack: duotone
icon: code
draft: false
toc: true
comments: false
date: 2026-06-24
category: Go
tags:
  - go
  - generics
---
Go 1.18から導入されたGenerics（型パラメータ）は、インターフェースによる抽象化とは異なり、コンパイル時の型安全性を維持したまま冗長なコードを共通化できます。今回は、Genericsを効果的に活用する基本的な方法と実用パターンを整理します。

<!--more-->

## 基本的な関数の抽象化

Genericsを使用すると、任意の型を受け取って処理する関数を型安全に定義できます。最もシンプルな例として、スライスの特定の要素が含まれているかを判定する関数を考えます。

```go
package main

import "fmt"

// Contains はスライスの中に特定の要素が存在するかを判定します。
// comparable 制約は、比較演算子（==, !=）が利用可能な型に限定します。
func Contains[T comparable](slice []T, target T) bool {
	for _, v := range slice {
		if v == target {
			return true
		}
	}
	return false
}

func main() {
	ints := []int{1, 2, 3}
	fmt.Println(Contains(ints, 2)) // true

	strs := []string{"go", "rust", "zig"}
	fmt.Println(Contains(strs, "python")) // false
}
```

`comparable` はGoの定義済み制約であり、数値、文字列、ポインタなど、比較可能なすべての型にマッチします。

## カスタム制約の定義

特定のインターフェースやベースの型に制限したい場合、`interface` キーワードを使って独自の制約を定義します。

```go
package main

// Number は整数または浮動小数点数を表す制約です。
type Number interface {
	~int | ~int64 | ~float64
}

// ~（チルダ）記号は、int や float64 をベースとしたエイリアス型（type MyInt int）も許可することを意味します。
func Sum[T Number](slice []T) T {
	var total T
	for _, v := range slice {
		total += v
	}
	return total
}
```

## 構造体での型パラメータの使用

関数だけでなく、構造体自体にも型パラメータを持たせることができます。これにより、任意の型を保持できるコンテナ（コレクション）を構築できます。

```go
type Set[T comparable] struct {
	elements map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
	return &Set[T]{
		elements: make(map[T]struct{}),
	}
}

func (s *Set[T]) Add(val T) {
	s.elements[val] = struct{}{}
}

func (s *Set[T]) Has(val T) bool {
	_, exists := s.elements[val]
	return exists
}
```

## いつGenericsを使うべきか

Genericsは強力ですが、多用するとコードの可読性が低下します。以下のガイドラインを参考に設計すると良いでしょう。

- **使うべき場面**:
  - スライス、マップ、チャネルに対するヘルパー関数（例: Map/Filter関数やシャッフル処理）
  - 型に依存しない汎用的なデータ構造（例: セット、キュー、連結リスト、キャッシュコンテナ）
- **避けるべき場面**:
  - メソッド呼び出しだけで抽象化できる場合（標準の `io.Reader` インターフェースで十分に扱える場合など）
  - 実装が型ごとに大きく異なる場合

## まとめ

Genericsは、コンパイル時の型検証による安全性を確保しながら、繰り返しコードを削除できる強力な仕組みです。基本的には `comparable` やカスタム制約を組み合わせ、必要な時にだけ導入することが成功の鍵です。
