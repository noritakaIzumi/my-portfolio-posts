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
	// given
	r := api.SetupRouter()
	w := httptest.NewRecorder()
	req, _ := http.NewRequest("GET", "/ping", nil)

	// when
	r.ServeHTTP(w, req)

	// then
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

{{< alert type="success" >}}
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

### アルバム一覧を取得する

アルバム一覧を取得するため、`/albums` に GET リクエストを送るとアルバム一覧が返ってくるような API を作成します。

#### 実装

`SetupRouter()` に以下のルーティングを追加します。

```go
	r.GET("/albums", getAlbums)
```

今回はアルバムを変数に定義し、変数の内容をそのままレスポンスする内容にします。

```go
// album represents data about a record album.
type album struct {
	ID     string  `json:"id"`
	Title  string  `json:"title"`
	Artist string  `json:"artist"`
	Price  float64 `json:"price"`
}

// albums slice to seed record album data.
var albums = []album{
	{ID: "1", Title: "Blue Train", Artist: "John Coltrane", Price: 56.99},
	{ID: "2", Title: "Jeru", Artist: "Gerry Mulligan", Price: 17.99},
	{ID: "3", Title: "Sarah Vaughan and Clifford Brown", Artist: "Sarah Vaughan", Price: 39.99},
}

// getAlbums responds with the list of all albums as JSON.
func getAlbums(c *gin.Context) {
	c.JSON(http.StatusOK, albums)
}
```

#### テストコード

`encoding/json` と `github.com/go-test/deep` をインポートしたうえで以下のコードを追加します。

```go
type album map[string]interface{}

func TestGetAlbums(t *testing.T) {
	// given
	r := api.SetupRouter()
	w := httptest.NewRecorder()
	req, _ := http.NewRequest("GET", "/albums", nil)

	// when
	r.ServeHTTP(w, req)

	// then
	var gotBody []album
	if err := json.Unmarshal(w.Body.Bytes(), &gotBody); err != nil {
		t.Error("Failed to unmarshal json string")
		t.Logf("got: %s\n", w.Body.String())
	}
	type res struct {
		code int
		body []album
	}
	want := res{
		code: http.StatusOK,
		body: []album{
			{"id": "1", "title": "Blue Train", "artist": "John Coltrane", "price": 56.99},
			{"id": "2", "title": "Jeru", "artist": "Gerry Mulligan", "price": 17.99},
			{"id": "3", "title": "Sarah Vaughan and Clifford Brown", "artist": "Sarah Vaughan", "price": 39.99},
		},
	}
	got := res{
		code: w.Code,
		body: gotBody,
	}

	if got.code != want.code {
		t.Errorf("Status code: want %d, but got %d", want.code, got.code)
	}
	if diff := deep.Equal(got.body, want.body); diff != nil {
		t.Error("Failed to assert body")
		for _, s := range diff {
			t.Log(s)
		}
	}
}
```

### 個々のアルバムを取得する

次はアルバムの ID を指定して個別にアルバム情報を取得する API を作成します。  
例えば、ID が 1 のアルバムを取得したいときは `/album/1` に GET を送れるようにします。

#### 実装

`SetupRouter()` に以下のルーティングを追加します。

```go
	r.GET("/albums/:id", getAlbumByID)
```

DB に見立てた album 変数の中から目的の ID に合致するものを返却します。  
見つからなければ NotFound とします。

```go
// getAlbumByID locates the album whose ID value matches the id
// parameter sent by the client, then returns that album as a response.
func getAlbumByID(c *gin.Context) {
	id := c.Param("id")

	for _, a := range albums {
		if a.ID == id {
			c.JSON(http.StatusCreated, a)
			return
		}
	}
	c.JSON(http.StatusNotFound, gin.H{"message": "album not found"})
}
```

#### テストコード

まずは存在する ID に対するテストです。

```go
func TestGetAlbum_forExistingAlbum(t *testing.T) {
	type args struct {
		id   string
		want album
	}
	type tc struct {
		name string
		args args
	}

	tcs := []tc{
		{name: "id:1", args: args{
			id:   "1",
			want: album{"id": "1", "title": "Blue Train", "artist": "John Coltrane", "price": 56.99},
		}},
		{name: "id:2", args: args{
			id:   "2",
			want: album{"id": "2", "title": "Jeru", "artist": "Gerry Mulligan", "price": 17.99},
		}},
		{name: "id:3", args: args{
			id:   "3",
			want: album{"id": "3", "title": "Sarah Vaughan and Clifford Brown", "artist": "Sarah Vaughan", "price": 39.99},
		}},
	}

	type res struct {
		code int
		body album
	}

	router := api.SetupRouter()
	for _, tt := range tcs {
		t.Run(tt.name, func(t *testing.T) {
			w := httptest.NewRecorder()
			url := fmt.Sprintf("/albums/%s", tt.args.id)
			req, _ := http.NewRequest("GET", url, nil)

			router.ServeHTTP(w, req)

			var gotBody album
			if err := json.Unmarshal(w.Body.Bytes(), &gotBody); err != nil {
				t.Error("Failed to unmarshal json string")
				t.Logf("got: %s\n", w.Body.String())
			}

			want := res{
				code: http.StatusCreated,
				body: tt.args.want,
			}
			got := res{
				code: w.Code,
				body: gotBody,
			}

			if got.code != want.code {
				t.Errorf("Status code: want %d, but got %d", want.code, got.code)
			}
			if diff := deep.Equal(got.body, want.body); diff != nil {
				t.Error("Failed to assert body")
				for _, s := range diff {
					t.Log(s)
				}
			}
		})
	}
}
```

{{< alert type="success" >}}
パラメタライズの関数も Golang の標準ライブラリにはなく、「それって各テストケースに対して for 文回しているのと一緒だよね、なら
for 文使いましょ？」ということです。  
参考記事：<https://qiita.com/a-suenami/items/2b6ba734ef6f69068253>
{{< /alert >}}

次に、存在しない ID に対するテストです。

```go
func TestGetAlbum_forNonExistentAlbum(t *testing.T) {
	router := api.SetupRouter()
	w := httptest.NewRecorder()
	url := fmt.Sprintf("/albums/%s", "4")
	req, _ := http.NewRequest("GET", url, nil)

	router.ServeHTTP(w, req)

	if w.Code != http.StatusNotFound {
		t.Errorf("Status code: want %d, but got %d", http.StatusNotFound, w.Code)
	}
}
```
