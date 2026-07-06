---
title: <span>net/http</span> の終了処理を実装する
title_safe: net/httpの終了処理を実装する
highlight: purple
pack: duotone
icon: cell-tower
draft: false
toc: true
comments: false
date: 2026-06-23
category: Go
tags:
  - go
  - http
  - ops
---
HTTPサーバーは起動より終了処理のほうが雑になりがちです。`http.Server.Shutdown`を中心に組み立てると、リクエスト処理中の接続を待ちながら、一定時間でプロセスを止められます。

<!--more-->

## Serverを明示的に持つ

`http.ListenAndServe`を直接呼ぶと、後から`Shutdown`する対象を持てません。`http.Server`を値として作っておきます。

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", healthz)

	srv := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
	}

	errCh := make(chan error, 1)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	if err := waitAndShutdown(context.Background(), srv, errCh); err != nil {
		log.Fatal(err)
	}
}
```

`ReadHeaderTimeout`も忘れずに設定します。ヘッダー読み取りで長時間ぶら下がる接続を抑えるためです。

## signal.NotifyContextで停止を受ける

OSシグナルを`context`に変換すると、終了処理の流れが読みやすくなります。

```go
func waitAndShutdown(parent context.Context, srv *http.Server, errCh <-chan error) error {
	ctx, stop := signal.NotifyContext(parent, os.Interrupt, syscall.SIGTERM)
	defer stop()

	select {
	case <-ctx.Done():
	case err := <-errCh:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			return err
		}
		return nil
	}

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown http server: %w", err)
	}

	if err := <-errCh; err != nil && !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}
```

`Shutdown`には、親のキャンセル済み`ctx`ではなく新しいタイムアウト付き`context`を渡します。停止シグナルを受けた時点で親はキャンセル済みなので、そのまま使うと猶予時間を待てません。

## Kubernetes前提なら猶予時間を揃える

コンテナ環境では、アプリ側の猶予時間、ロードバランサーの切り離し、`terminationGracePeriodSeconds`を合わせます。アプリが10秒待つのに、実行基盤が5秒で強制終了する設定では意味がありません。

終了処理の実装と同じくらい、外側の設定もレビュー対象にします。

## まとめ

`net/http`の終了処理は、`http.Server`を明示的に持ち、シグナルを受け、別のタイムアウト付き`context`で`Shutdown`するだけで安定します。起動コードと同じファイルに終了条件まで書いておくと、運用時の挙動を追いやすくなります。
