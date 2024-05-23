---
draft: true
title: "Go 製フレームワーク Gin を使って REST API を作成してみる"
date: "2024-05-19T18:16:37+09:00"
description: Gin を使った REST API の作成、テストコードを書いてみる。
---

REST API を Go 製フレームワーク Gin を使って作ってみる記事です。  
単に実装するだけでなく、業務で使用することも踏まえて API の動作を検証するテストコードにも触れていきます。  

Go の環境構築やプロジェクト作成手順については触れません。

なお、以下の記事を参考にしています。

- [Tutorial: Developing a RESTful API with Go and Gin - The Go Programming Language](https://go.dev/doc/tutorial/web-service-gin)

---

## 環境

私の環境です。

- Windows 11, Ubuntu on WSL2
- Go 1.22.2 ([asdf](https://asdf-vm.com/) でインストール)

## アルバム管理 API を作成する

今回は、チュートリアルに従って CD 店のアルバム管理のようなものを作ります。

### 事前準備

ディレクトリを作成します。

```shell
mkdir web-service-gin
cd web-service-gin
```

パッケージ管理のためのモジュールを作成します。

```shell
go mod init example/web-service-gin
```

### 最小構成を作成する

#### 実装

`main.go` を作成し、まずは `/ping` にアクセスしたら `pong` と返してくるような、シンプルな API を作ります。

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.String(http.StatusOK, "pong")
	})

	err := r.Run(":8080")
	if err != nil {
		panic("failed to start server")
	}
}
```

import に記述したパッケージをモジュールに追加するため、以下のコマンドを実行します。

```shell
go mod tidy
```

`go run .` でサーバを起動し、<http://localhost:8080/ping> にリクエストを送って結果を検証します。

```shell
diff -su <(echo -n "pong") <(curl http://localhost:8080/ping)
```

API が正常に動作していれば以下のような出力結果となります。

```text
$ diff -su <(echo -n "pong") <(curl http://localhost:8080/ping)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100     4  100     4    0     0    477      0 --:--:-- --:--:-- --:--:--   500
Files /dev/fd/63 and /dev/fd/62 are identical
```

#### リファクタリング

この後のテストコードに備えて、ルーティングを含むサーバ設定処理をメソッドに切り出しておきます。

```go
func SetupRouter() *gin.Engine {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.String(http.StatusOK, "pong")
	})

	return r
}
```

`main()` は以下のようになります。

```go
func main() {
	r := SetupRouter()

	err := r.Run(":8080")
	if err != nil {
		panic("failed to start server")
	}
}
```

#### テストコード

テストコードはファイル名が `*_test.go` となるように作ります。
ここでは `main_test.go` とします。

{{< alert type="info" >}}
Go では同じパッケージ内にテストを書く方法と別パッケージにテストを書いてインポートする方法があります。  
この例では後者を採用しています。
{{< /alert >}}

```go
package main_test

import (
	api "example/web-service-gin"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestPingRoute(t *testing.T) {
	router := api.SetupRouter()

	w := httptest.NewRecorder()
	req, _ := http.NewRequest("GET", "/ping", nil)
	router.ServeHTTP(w, req)

	type res struct {
		code int
		body string
	}
	want := res{
		code: http.StatusOK,
		body: "pong",
	}
	got := res{
		code: w.Code,
		body: w.Body.String(),
	}
	if got.code != want.code {
		t.Errorf("Status code: want %d, but got %d", want.code, got.code)
	}
	if got.body != want.body {
		t.Errorf("Body: want %q, but got %q", want.body, got.body)
	}
}
```

{{< alert type="warning" >}}
Go の testing ライブラリには assert メソッドがありません。その理由については以下の記事が参考になります。  
<https://qiita.com/Jxck/items/8717a5982547cfa54ebc>
{{< /alert >}}

パッケージを `go mod tidy` で更新し、`go test` でテストを実行します。

```text
$ go test
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /ping                     --> example/web-service-gin.SetupRouter.func1 (3 handlers)
[GIN] 2024/05/19 - 20:32:46 | 200 |      11.801µs |                 | GET      "/ping"
PASS
ok      example/web-service-gin 0.005s
```
