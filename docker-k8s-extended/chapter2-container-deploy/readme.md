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

# イメージの操作

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

```
docker container run --platform linux/arm64 -d example/echo:latest
```

→ バックグラウンドでコンテナを実行する

### ポートフォワーディング

```
curl http://localhost:8080/
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

これだとうまくいかない
Docker コンテナは仮想環境だが、外から 1 つの独立したマシンのように扱える特徴がある
指定した 8080 は「コンテナポート」 と呼ばれ、コンテナの中でしか有効でない
コンテナの外からアクセス可能にするためにポートフォワーディングをつかう。

```
docker container run --platform linux/arm64 -d -p 9000:8080 example/echo:latest
```

ホスト側の 9000 をコンテナ側の 8080 にいポートフォワーディングする

```
curl http://localhost:9000/
=> Hello Docker!!
```

### タグ

Docker イメージのバーションをそのように呼ぶケースがある
「Docker イメージのバージョン」は正確には単なる「イメージ ID」のことである。

# コンテナの操作

**docker コンテナライフサイクル**
docker コンテナは常に 実行中・停止・破棄 の 3 つの状態のどれかになる

# Docker Compose の操作

アプリケーションが一定の規模になれば、複数のアプリケーションを構築して、それらを連携させる必要がある。
つまり Docker コンテナでシステムを構築するということは複数存在するコンテナ同士が通信し、かつコンテナがコンテナの依存を持つ。
そこで単一のコンテナでは問題にならなかった、設定ファイルや環境変数、コンテナ同士の依存関係などを
考えて運用する必要がでてくる。
これを今までのコマンドのみでやろうとすると大変つらいので Docker Compose をつかう。

## 複数コンテナの稼働

### Jenkins でためす

```yaml
version: "3"
services:
  master:
    container_name: master
    image: jenkins:latest
    ports:
      - 8080:8080
    volumes:
      - ./jenkins_home:/var/jenkins_home
```

volume はホストとコンテナ間でファイルがコピーされるのではなく、共有される仕組み。
Dockerfile の COPY や`docker container cp`はホストとコンテナの間でコピーするものだが
volumes は共有となる。
ホストのカレントディレクトリ直下の jenkins_home ディレクトリと Jenkins コンテナ側の/var/jenkinx_home にマウントしている。

```
docker-compose up
=> Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
=> 98113d3eea7648738050856af8d24cc9
```

### Master Jenkins の SSH 鍵を作る

Jenkins の Slave コンテナをつくる。
本来 Jenkins は複数のサーバー(Master と Slave)に分けることが多い
事前準備として Master が Slave に SSH 接続できるようにするため Master コンテナの SSH Key を作成する。
→ Master から Slave に疎通確認をするために必要

```
docker container exec -it master ssh-keygen -t rsa -C ""
=>
Generating public/private rsa key pair.
Enter file in which to save the key (/var/jenkins_home/.ssh/id_rsa): /var/jenkins_home/.ssh/id_rsa
Created directory '/var/jenkins_home/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /var/jenkins_home/.ssh/id_rsa
Your public key has been saved in /var/jenkins_home/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:m+rrgY0cug/Q/o6LowVrzvf7Thl0dZp2J6f+i6GK27U
The key's randomart image is:
```

1. 事前に Master コンテナを作成し SSH 公開鍵の生成をする
2. docker-compose に Slave 用コンテナを追加し、Master の SSH 公開鍵を環境変数に指定
3. links を使用して Master コンテナから Slave コンテナへ通信できるようにする

### SSH に関する予備知識

上では秘密鍵(id_rsa)と公開鍵(id_rsa.pub)が作成された。

秘密鍵：　他の人に教えてはいけない
公開鍵：　他の人に教えてもよい

### 続き

`docker-compose up`すると master と slave01 が立ち上がる
	