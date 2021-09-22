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

### ボリュームマウント
Docker Engineが管理している領域内にボリュームを作成しディスクとしてコンテナにマウントする
→ 滅多に触らないが消してはいけないファイルなど

ボリュームマウントはホストOSの差によるファイルの場所を意識する必要がない
例えばOSによって指定するべきファイルパスが異なる。など。

### バインドマウント
Dockerをインストールしたパソコンのドキュメントやデスクトップなど
Docker Engineの管理していない場所の既に存在するディレクトリをコンテナにマウントする
→ 頻繁に触りたいファイルなど

### 記憶領域のマウント
アプリのショートカットをつくるとき、アプリの実際の本体は別の場所にある。
しかしショートカットをつくることでそこに本体があるかのように扱うことができる。
**記憶領域のマウントもこのようなイメージ**

記憶領域をマウントする場合、先にその記憶領域を作成しておく。

```
docker volume create [ボリューム名]
```
ボリューム作成(ボリュームマウント)

```
docker run ~~~ -v 実際の記憶領域パス:コンテナの記憶領域パス
```
バインドマウントの記述例

``` 
docker run ~~~ -v ボリューム名:コンテナの記憶領域パス
```
ボリュームマウントの記述例