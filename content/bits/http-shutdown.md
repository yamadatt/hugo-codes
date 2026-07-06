---
type: blurb
date: 2026-07-04T14:30:00+09:00
tags:
  - go
  - http
  - ops
---

`http.Server.Shutdown`へ渡す`context`は、停止シグナルでキャンセル済みのものではなく、新しく作った猶予時間付きのものにします。
