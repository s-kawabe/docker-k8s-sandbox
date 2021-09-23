# chaoter6 応用的なコンテナの使い方

これ以降で登場するもの
1. コンテナとホスト間でファイルをコピーする
2. ボリュームのマウント
3. コンテナのイメージ化
4. コンテナの改造
5. Docker Hubへの登録
6. Docker Compose
7. Kubernetes

##　コンテナとホスト間でファイルをコピーする
コンテナ上にたてたサーバーとのやりとりを、ソフトウェアを介さずに行いたい場合
コンテナとホストOSの間でファイルをコピーする方法がある。
コピーはコンテナとホストで双方向で可能となっている。

コピーコマンド「dcoker cp」

ホストからコンテナ
```
docker cp ホスト側パス コンテナ名:コンテナ側パス
```

コンテナからホスト
```
docker cp コンテナ名:コンテナ側パス ホスト側パス
```

### Apacheのコンテナにindex.htmlをコピーして初期表示をこれにする(ホストからコンテナへのコピー)

```
docker run --name apa000ex19 -d -p 8089:80 httpd
```
Apacheコンテナの作成
→この時点でlocalhostにアクセスしても初期状態のまま。

```
docker cp /Users/kawabeshintarou/Desktop/playgrounds/docker/docker-k8s-basic/chap6/index.html apa000ex19:/usr/local/apache2/htdocs/
```
chap6フォルダに用意したhtmlをコピーする
→localhostを表示させるとコピーしたhtmlが表示される

### コンテナからホストにファイルのコピーをする

```
mv /Users/kawabeshintarou/Desktop/playgrounds/docker/docker-k8s-basic/chap6/index.html /Users/kawabeshintarou/Desktop/playgrounds/docker/docker-k8s-basic/chap6/index2.html
```

```
docker cp apa000ex19:/usr/local/apache2/htdocs/index.html /Users/kawabeshintarou/Desktop/playgrounds/docker/docker-k8s-basic/chap6/
```
→ 先ほどapacheコンテナにコピーしたindex.htmlがホスト側にコピーされる。

## ボリュームのマウント
ボリュームをマウントすると、コンテナの一部分を土台の一部分のように扱える。
**データを永続化できる。** 外部HDDのようなイメージ。
「ボリュームをコンテナ本体にマウントする。」と言ったりする
マウントには2つの方法がある。

- ボリュームとは
ストレージの１領域を区切ったもので、ハードディスクやSSDの区切られた１領域のようなイメージ。

- マウントとは
「取り付ける」の意味。
対象を接続して、OSやソフトウェアの支配下におくこと。
USBメモリをPCにさして、USBの中身を操作可能にする→「USBをPCにマウントする」
と言った感じ。

コンテナへのマウントは「docker run」時の-vオプションとして命令する

### ボリュームマウント
Docker Engineが管理している領域内にボリュームを作成しディスクとしてコンテナにマウントする
→ 滅多に触らないが消してはいけないファイルなど

ボリュームマウントはホストOSの差によるファイルの場所を意識する必要がない
例えばOSによって指定するべきファイルパスが異なる。など。

**作成したPC上のボリュームにファイルをおくと、コンテナにも反映される**

### バインドマウント
Dockerをインストールしたパソコンのドキュメントやデスクトップなど
Docker Engineの管理していない場所の既に存在するディレクトリをコンテナにマウントする
→ 頻繁に触りたいファイルなど

### 記憶領域のマウント
アプリのショートカットをつくるとき、アプリの実際の本体は別の場所にある。
しかしショートカットをつくることでそこに本体があるかのように扱うことができる。
**記憶領域のマウントもこのようなイメージ**

記憶領域をマウントする場合、先にその記憶領域を作成しておく。

1. バインドマウント
```
docker run ~~~ -v 実際の記憶領域パス:コンテナの記憶領域パス
```
バインドマウントの記述例

2. ボリュームマウント
```
docker volume create [ボリューム名]
```
ボリューム作成(ボリュームマウント)


``` 
docker run ~~~ -v ボリューム名:コンテナの記憶領域パス
```
ボリュームマウントの記述例

### バインドマウントをしてみる
```
docker run --name apa000ex20 -d -p 8090:80 -v Users/kawabeshintarou/Documents/apa_folder:/usr/local/apache2/htdogs httpd
```
ローカルの方のディレクトリにhtmlを格納するとapache側の表示も変わる

### ボリュームマウントをしてみる
```
docker volume create apa000vol1
```

```
docker volume inspect apa000vol1
```

```
docker run --name apa000ex21 -d -p 8091:80 -v apa000vol1:/usr/local/apache2/htdogs httpd
```

```
docker container inspect apa000ex21
```
→コンテナにボリュームがマウントされていることが確認できる。


## コンテナのイメージ化
イメージの作成には２パターン
- すでにあるコンテナを「commit」でイメージの書き出しをする方法
- Dockerfileを使ってイメージを生成する方法 

### commitでコンテナからイメージを作る

```
docker commit コンテナ名 作成するイメージ名
```

### Dockerfileでイメージを作る
1. ベースにする Docker イメージを決める
2. `docker run -it <docker-image> sh` でコンテナ内部で作業
3. 1行ずつ、うまくいったらどこかにメモ
4. 失敗したらいったん `exit` して再度 `docker run`
5. ファイルの取り込みやポートの外部公開が必要ならオプション付きで `docker run`
6. 全部うまくいったら Dockerfile にする
https://qiita.com/pottava/items/452bf80e334bc1fee69a#dockerfile-%E3%83%99%E3%82%B9%E3%83%88%E3%83%97%E3%83%A9%E3%82%AF%E3%83%86%E3%82%A3%E3%82%B9

## コンテナの改造
Dockerを使う場面では、自社で開発したシステムを載せることも多く
公式が配布しているソフトウェアを改造したいこおtもある。

**コンテナを改造する方法**
(2つを併用することが多い)
1. `docker cp`によるファイルやりとり
2. Linuxのコマンドで命令する

- コンテナの命令にはshell(bash)が必要
→ `/bin/bash`コマンドにてbashを起動させる必要があるが、この引数は`docker run`コマンドや`docker exec`コマンド
につけて実行する。 

- bashを使ってコンテナ内部をいじっている時は、Dockerコマンドは効かない
- コンテナを作ったり削除したりする場合 → Docker(Engine)に向けた命令
  - Docker Engineの起動と停止
  - コンテナの起動と停止
  - コンテナ内部のファイルやりとり 

- コンテナ内部をbashで操作する場合 → コンテナ内部に向けた命令
  - ソフトウェアのインストール 
  - ソフトウェアの実行と停止
  - ソフトウェアの設定変更
  - ファイル操作