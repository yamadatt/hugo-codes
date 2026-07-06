---
title: Goの構造体タグと <span>reflection</span> の仕組み
title_safe: Goの構造体タグとreflectionの仕組み
highlight: amber
pack: duotone
icon: tags
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - reflection
---
Goでは、JSONのシリアライズやORM（O/Rマッピング）の際に構造体の後ろに `` `json:"id"` `` のようなメタデータを記述します。これは「構造体タグ（Struct Tag）」と呼ばれ、実行時に `reflect` パッケージを使って読み取られます。今回はその仕組みと自作バリデータなどの応用方法を解説します。

<!--more-->

## 構造体タグとは？

構造体タグは、構造体のフィールドに付与するメタデータ文字列です。Goの言語機能としては単純な文字列として扱われますが、規約として `key:"value"` の形式で記述されます。

```go
type User struct {
	ID   int    `json:"id" db:"user_id"`
	Name string `json:"name" db:"user_name"`
}
```

## reflectionによるタグの取得方法

`reflect` パッケージを使用することで、コンパイル時には決まっていない変数の型情報を実行時に動的に検査できます。

```go
package main

import (
	"fmt"
	"reflect"
)

type Config struct {
	Host string `env:"APP_HOST"`
	Port int    `env:"APP_PORT"`
}

func main() {
	cfg := Config{Host: "localhost", Port: 8080}
	
	// reflect.TypeOf で型情報を取得
	t := reflect.TypeOf(cfg)

	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		
		// 構造体タグをパースして取得
		envKey := field.Tag.Get("env")
		fmt.Printf("Field: %s, Tag (env): %s\n", field.Name, envKey)
	}
}
```

`reflect.TypeOf` で構造体のメタ情報を表す `reflect.Type` を取得し、`Field(i).Tag.Get("key")` で対応するタグの値を抽出できます。

## 実践：簡易的なバリデータの作成

Reflection を利用して、必須フィールドが空でないかを検証するカスタムバリデータ関数を作成してみます。

```go
package main

import (
	"errors"
	"fmt"
	"reflect"
)

type RegisterRequest struct {
	Username string `validate:"required"`
	Email    string `validate:"required"`
	Age      int
}

func Validate(s interface{}) error {
	val := reflect.ValueOf(s)

	// ポインタが渡された場合は実体を取得
	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}

	if val.Kind() != reflect.Struct {
		return errors.New("must be a struct")
	}

	typ := val.Type()

	for i := 0; i < val.NumField(); i++ {
		fieldVal := val.Field(i)
		fieldType := typ.Field(i)

		// validate タグがあるか確認
		tag := fieldType.Tag.Get("validate")
		if tag == "required" {
			// 今回は簡易的に文字列型のみ検証
			if fieldVal.Kind() == reflect.String && fieldVal.String() == "" {
				return fmt.Errorf("field %s is required", fieldType.Name)
			}
		}
	}

	return nil
}

func main() {
	req := RegisterRequest{
		Username: "",
		Email:    "test@example.com",
	}

	if err := Validate(req); err != nil {
		fmt.Println("Validation error:", err) // field Username is required
	} else {
		fmt.Println("Validation passed")
	}
}
```

## reflectionの注意点

Reflection は強力ですが、以下のデメリットがあります。

1. **パフォーマンスの低下**: 反射を用いた型情報の解決は、静的なコード実行に比べて処理コストが高くなります。高頻度で呼び出されるホットパスでの過度な使用は避けるべきです。
2. **コンパイル時安全性の欠如**: 静的型付けの恩恵であるコンパイルチェックを受けられないため、存在しないフィールドへのアクセスなどで実行時パニックを引き起こす危険性があります。

## まとめ

構造体タグと `reflect` パッケージを組み合わせることで、柔軟性の高いメタプログラミングが可能になります。ライブラリなどの構築時には非常に便利ですが、通常のビジネスロジックでは過度な使用を避け、パフォーマンスと安全性のバランスを考慮しましょう。
