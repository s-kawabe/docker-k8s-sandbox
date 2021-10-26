# 基礎の基礎

image の pull

```
docker image pull gihyodocker/echo:latest
```

download したイメージの実行

```
docker container run --platform linux/amd64 -t -p 9000:8080 gihyodocker/echo:latest
```

ここで作成されたコンテナはオプションでポートフォワーディングを設定しているので、
Docker 実行環境の 9000 ポート経由で HTTP リクエストを受けられるようになっている

```
curl http://localhost:9000/
=> Hello Docker!
```

# ハンズオン

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		log.Println("received request")
		fmt.Fprintf(w, "Hello Docker!!")
	})

	log.Println("start server!")
	server := &http.Server{Addr: ":8080"}
	if err := server.ListenAndServe(); err != nil {
		log.Println(err)
	}
}

```

```Dockerfile
FROM golang:1.9

RUN mkdir /echo
COPY main.go /echo

CMD ["go", "run", "/echo/main.go"]
```

- Go で Web サーバーを作る
- どんな HTTP リクエストに対しても Hello Docker する
- 8080 ポートでサーバーアプリケーションとして動作

```
docker image build -t example/echo:latest .
```