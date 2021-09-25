# chapter4 - apacheをdocker run

```
docker run --name apa000ex1 httpd
```
→imageが入っていない場合はリモートからpullしようとする

```
docker ps
```
apa000ex1が立ち上がっていることを確認

```
docker stop apa000ex1
docker ps
```
apa000ex1が停止することを確認

```
docker rm apa000ex1
docker ps -a
```
apa000ex1が削除されていることを確認

- docker rmはコンテナIDやコンテナIDの省略形でもできる
CONTAINER IDが「2d4ngk94bec」などなら、
フルネーム、もしくは「docker rm 2d」などでも実行可能

# コンテナと通信
- ApacheはWebサーバーを提供するソフトウェア、ApacheにファイルをおくことでWebサイトとしてみられる
- インターネットにコンテナを載せて、ブラウザでアクセスできるようにする
- コンテナ作成時にdocker runのオプションとして設定する

実際このようなサーバ側の構成としては上から
- Apache container
- Docker Engine
- Linux
- 物理マシン
のような構成になっていて、実際やりとりを行うのは物理マシンである。
httpdでは使用上ポート80で受け付けるようになっている

## -pオプション
docker run時などに「ホストのポート番号」と「コンテナのポート番号」をコロンでつないで結びつける
```
-p 8080:80
```

## docker rmの前にstopしなきゃいけない
先にrmした時のエラー
```
Error response from daemon: You cannot remove a running container d298e9f73f28911df69733bd2c02af7da19483516fa5728972174f8c9df92e5b. Stop the container before attempting removal or force remove
```

## apacheコンテナを作成してブラウザでアクセスする
1. apacheコンテナ作成・起動
```
docker run --name apa000x2 -d -p 8080:80 httpd
```
httpdバージョンを指定しないと自動でlatestになる。

2. ブラウザで確認
localhost:8080でアクセスできることを確認する。

3. 停止、削除
```
docker stop apa000x2
```

```
docker rm apa000x2
```

4. 確認
docker ps -aでも確認できず、localhost:8080でもアクセスできなくなっている

## さまざまなコンテナ作成
Linux OSのコンテナは中に入って操作することが可能なので、引数にてシェルコマンドと呼ばれる操作のために指定をする
- Linux OSのみ入ったコンテナ
  - ubuntu
  - CentOS
  - DebianOS
  - Fedora
  - BizyBox
  - Alpine linux

WebサーバーやDBサーバー用のコンテナはサーバで稼働するため、ポート番号をオプションにつける必要がある
また、データベース管理ソフトの場合は**基本的にルートパスワードの指定が必要。**
- WebサーバーやDBサーバー用のコンテナ
  - Apache
  - Nginx
  - MySQL
  - PostgreSQL
  - MariaDB

プログラムを実行するにはその言語の実行環境が必要。
実行環境もコンテナとして提供されている。
- openjdk
- python
- php
- ruby
- perl
- gcc
- node
- registory
- wordpress
- nextcloud
- redmine

## Nginxのコンテナを立てる
```
docker run --name nginx000ex6 -d -p 8084:80 nginx
```
→ localhost:8084にアクセスできる
→ docker stopとdocker rmをする

## MySQLのコンテナを立てる
```
docker run --name mysql1000ex7 -dit -e MYSQL_ROOT_PASSWORD=myrootpass mysql
```

- -ditはキーボード操作を可能にし、バックグラウンドで実行させるコマンド
- DBコンテナを立てる場合は引数にルートPASSWORDを指定する
- -eコマンドで引数を指定している

m1macのばあい
https://ichi-station.com/docker-mysql-for-m1-mac/
→ 「--platform linux/amd64」オプションをつける！

→ docker psで立ち上がっていることを確認し停止して削除する

🌟docker run時にimageのバージョンを指定できる
```
docker run --name apa000ex2 -d -p 8080:80 httpd:2.2
```
→指定しない場合はlatestになる。
→rmする際もバージョンをうしろにつける。

## imageの削除
```
docker rmi xxxx
```
または
```
docker image rm
```

→image名は複数指定すると複数一度に削除ができる
→imageのバージョンは「TAG」というところの列に表示されている

# chapter5 - dockerに複数のコンテナを入れて動かしてみる

## コンテナにWordPresstとMySQLを入れて動かす
- WordPressを動かすには、ApacheやDB、PHPの実行環境が必要
- 公式が用意しているWordPressイメージにはいろいろ入っている
  - そこからWordPressコンテナと
  - MySQLコンテナを作成する。
- 上記2つのコンテナは連携し合っていなければアプリは動かない
- そこで、仮想的なネットワークを作成するための「docker network create」コマンドを仕様する
- docker network rmやdocker network lsで操作することができる

### MySQL起動時
```
docker network create wordpress000net1
```
ネットワークの作成

```
docker run --platform linux/amd64 --name mysql000ex11 -dit --net=wordpress000net1 -e MYSQL_ROOT_PASSWORD=myrootpass -e MYSQL_DATABASE=wordpress000db -e MYSQL_USER=wordpress000kun -e MYSQL_PASSWORD=wkunpass mysql --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --default-authentication-plugin=mysql_native_password
```
→MYSQLにはROOTユーザと一般ユーザ両方にパスワードの設定が必要。
→**-eオプション**にて環境変数を設定している
→最後のほうで引数を定義している

### WordPress起動
```
docker run --name wordpress000ex12 -dit --net=wordpress000net1 -p 8085:80 -e WORDPRESS_DB_HOST=mysql000ex11 -e WORDPRESS_DB_NAME=wordpress000db -e WORDPRESS_DB_USER=wordpress000kun -e WORDPRESS_DB_PASSWORD=wkunpass wordpress
```
MySQLに接続するために設定値はDBにて設定したものを設定する。

## RedmineのコンテナとMariaDBのコンテナを作成する

### ネットワークの作成
```
docker network create redmine000net3
```

### MariaDBコンテナの作成
```
docker run --name mariadb000ex15 -dit --net=redmine000net3 -e MYSQL_ROOT_PASSWORD=mariarootpass -e MYSQL_DATABASE=redmine000db -e MYSQL_USER=redmine000kun -e MYSQL_PASSWORD=rkunpass mariadb --character=set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --default-authentication-plugin=mysql_native_password
```

### Redmineコンテナの作成
```
docker run -dit --name redmine000ex16 --network redmine000net3 -p 8087:3000 -e REDMINE_DB_MYSQL=mariadb000ex15 -e REDMINE_DB_DATABASE=redmine000db -e REDMINE_DB_USERNAME=redmine000kun -e READMINE_DB_PASSWORD=rkunpass redmine
```

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

- FROMにはベースとするDocker Image を記述する
→ CentOSとかAlpine Linuxとか、 弊社ではnodeが書いてあったりした

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

# chapter7 - Docker Compose

## DockerComposeとは
WHY - 複数コンテナで構成するシステムを作るのが面倒になってくる。
引数やオプション、ボリュームやネットワークの管理が増える。

こういった構築に関わるコマンドの内容を一つのテキストに書き込んで一気に実行、停止、破棄できるのがDocker Compose。

yamlファイルを書いて定義する。
内容は起動するコンテナ名とその内容、Networkの設定、Volumeの設定など。

## コマンド
- docker-compose up
  - yamlの内容に従ってイメージをダウンロードしたりコンテナを作成、起動したりする 
- docker-compose down
  - コンテナとネットワークを停止・削除する。(この際ボリュームとイメージは残る。)
- docker-compose stop
  - コンテナとネットワークを停止する。

## docker-compose.yaml
- version
  - バージョンを記述する
- services
  - コンテナの情報
- networks
  - ネットワークの情報
- volumes
  - ボリュームの情報

- image
  - 利用するイメージの使用
- networks
  - 接続するネットワークの指定(--net)
- volumes
  - 記憶領域のマウントの設定(-v, --mount)
- ports
  - ポートのマッピング設定(-p)
- environment
  - 環境変数の設定(-e)
- depends_on
  - 別のサービスに依存していることを表す
- restart
  - コンテナが停止したときの再試行ポリシーの設定

**他の定義項目**
- command
- container_name
- dns
- env_file
- entrypoint
- external_links
- extra_hosts
- logging
- network_mode

## Docker Composeの実行
```
docker-compose -d 定義ファイルのパス [up,down,stop] オプション
```
- -d はバックグラウンドで実行するオプション
- docker-compose.yamlがカレントにあるならファイルパス指定は省略できる。

docker-compose.yamlのあるディレクトリにいって以下を実行
```
docker-compose up -d
```

**後始末**
```
docker-compose down
```
→imageとvolumeは削除されない

```
docker volume rm wordpress_mysql000vol11 wordpress_wordpress000vol12
```

```
docker rmi wordpress mysql httpd
```

# Kubernetesとは

コンテナのオーケストレーションツール。
複数のコンテナを管理して運用できるのがKubernetesの役割。
**複数のコンテナ**とは**全く同じ構成のコンテナが複数**という意味。

また、docker-composeと違う点は、docker-composeが複数のコンテナやボリュームの作成
のみを行えるのに対し、k8sは作成して管理しあるべき状態を維持するように監視することが可能

また、定義ファイル(docker-compose.yamlとマニフェストファイル)は、k8sの方は作成された定義ファイルをetcdというデータベースで管理するという点でも異なる。

この情報によってpodが管理される。
また、k8sの定義ファイルはコマンドによっても書き換えられる。

## Kubernetesは複数の物理的マシンに複数のコンテナがあることが前提
20個のコンテナが必要ならば20回`docker run`しなければならない、という事を改善する。
Docker Composeを使用したとしても物理マシンの数が多ければ手間は相当なものになる。
そこでKubernetesを使用することで**コンテナの作成や管理をよしなにやってくれる**

**マニフェストファイル**というものを定義することで、全ての物理マシンにコンテナを作成して管理する

# マスターノードとワーカーノード

## マスターノード

ワーカーノードを管理し、コントロールするノード
マスターノード上ではコンテナは動いていない

管理する側のPCには**kubectl**を入れて、これを使ってマスターノードにログインし
初期設定や各種調整を行う

## ワーカーノード

実際にタスクを動かすノード

## クラスター

マスターとワーカーで構成されたk8sシステムの一群を**クラスター**という
クラスターは自立していて、人間が命令をせずとも、マスターに設定された内容に従って
マスターノードが自立的にワーカーノードを管理する。

## Kubernetesを使うには

- k8sのインストール
- CNI(仮想ネットワークドライバ)のインストール
    - flannel
    - Calio
    - AWS VPC CNI

マスターノードはコンテナなどの状態管理のために**etcd**というデータベースを入れる。
ワーカーノードにはDockerEngineなどのコンテナエンジンが必要

**etcd**
定義ファイルをデータベース化したもの。
etcdに定義されたオブジェクトをインスタンス化することによって
実際のPodやサービスなどが稼働する。

## コントロールプレーンとkube-let

マスターノードでは、**コントロールプレーン**を使ってワーカーノードの管理をする

コントロールプレーンは5つのコンポーネントで構成される。

マスターノードのコントロールプレーン構成

- **kube-apiserver**
外部とやりとりをするプロセス
- **kube-controller-maneger**
コントローラーを統括管理、実行する
- **kube-scheduler**
Podをワーカーノードへ割り当てる
- **cloud-controller-manager**
クラウドサービスと連携してサービスを作る
- **etcd**
クラスター情報を全管理するデータベース
    - etcd以外はk8sの標準搭載となっている

ワーカーノードの構成

- **kube-let**
マスターノード側のkube-schedulerと連携してワーカーノード上にPodを配置して実行。
また、実行中のPodの状態を定期的に監視してkube-schedulerへ通知する
- **kube-proxy**
ネットワーク通信をルーティングする仕組み

## Kubernetesの構成

### Pod

kubernetesでは**コンテナはPodという単位で管理される。**
Podはコンテナとボリュームがセットになったもの。
しかしPod内にボリュームを作らないこともある

### サービス

**Podをまとめて管理する**ものをサービスと呼ぶ。
異なるワーカーノードに属するPodもその違いを気にせずにサービスがPodをまとめることもある
→複数のワーカーノードでも同じPodは一つのサービスが管理する

また、この役割はいわば**ロードバランサー**と同じ。
サービスごとに自動でIPアドレスが割り当てられ、そこにアクセスしてくる通信を捌く。
より上の各ワーカーノードへの振り分け自体は、本物のロードバランサやイングレスなどで行う。

### レプリカセット

**Podの数**を管理する。
障害などでPodが停止した場合など、足りない分を増やしたり、定義ファイルを元に実際のPodに
差分があった場合は削除したりする。
Podはサービスとレプリカセットの2つの管理のもとで稼働する。

レプリカセットにより管理されている同一構成のPodは**レプリカ**という

### デプロイメント

**Podのデプロイを管理する。**Podがどのイメージを使うかなどの情報を持つ。
レプリカセットは単体では使い勝手が悪いので、基本的にデプロイメントと一緒に使う。

## kubernetesのその他のリソース

- **pod**
- podtemplates
- replication controllers
- resource quotas
- secrets
- service accounts
- **service**
- demonsets
- **deployments**
- **replicasets**
- statefulsets
- cronjobs
- jobs