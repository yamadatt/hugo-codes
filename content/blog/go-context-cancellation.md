---
title: Goの <span>context</span> を境界で扱う
title_safe: Goのcontextを境界で扱う
highlight: blue
pack: duotone
icon: function
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - context
  - concurrency
---
`context.Context`は、値を持ち回るための便利袋ではなく、処理の寿命を伝播するための境界です。HTTPリクエスト、ジョブ、外部API呼び出しのどこで締め切りを決めるかを揃えると、Goの並行処理はかなり読みやすくなります。

<!--more-->

## 呼び出し元が予算を決める

下位の関数が勝手に長いタイムアウトを作ると、呼び出し元の意図よりも処理が長生きします。基本は、エントリポイントで予算を決め、下位にはその`ctx`を渡します。

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	user, err := service.LoadUser(ctx, r.PathValue("id"))
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}

	_ = json.NewEncoder(w).Encode(user)
}
```

`defer cancel()`はタイマーを解放するためにも必要です。タイムアウトしたかどうかだけでなく、成功した場合にも呼びます。

## goroutineには必ず停止条件を渡す

goroutineを起動する関数は、終了条件も同じ場所で読めるようにしておくと保守しやすくなります。

```go
func stream(ctx context.Context, out chan<- Event, src Source) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		ev, err := src.Next(ctx)
		if err != nil {
			return err
		}

		select {
		case <-ctx.Done():
			return ctx.Err()
		case out <- ev:
		}
	}
}
```

送信側の`select`にも`ctx.Done()`を入れるのが重要です。受信側が止まったあとも送信で詰まる、という種類のリークを避けられます。

## 値は境界情報だけにする

`context.WithValue`は認証済みユーザーID、リクエストID、トレースIDのような横断的な値に限定します。関数の主要入力を`ctx`に隠すと、テストもレビューも難しくなります。

```go
type requestIDKey struct{}

func WithRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey{}, id)
}

func RequestID(ctx context.Context) string {
	id, _ := ctx.Value(requestIDKey{}).(string)
	return id
}
```

キーは独自型にして、パッケージ外から衝突できないようにします。

## まとめ

`context`は「どの処理をいつ諦めるか」を表す設計要素です。エントリポイントで予算を決め、goroutineの停止条件に使い、値の用途を絞るだけで、キャンセルとタイムアウトの挙動はかなり予測しやすくなります。
