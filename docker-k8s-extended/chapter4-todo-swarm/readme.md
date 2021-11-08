# 実践アプリケーション開発(Swarm編)

```
docker container exec -it manager docker node ls
```
ここにアプリケーションを構築していく。
次のようにStackを構成していく。

- MySQL
  - mysql_master
  - mysql_slave
- Application
  - app_nginx
  - app_api
- Frontend
  - frontend_nginx
  - frontend_web

最初にoverlayネットワークを構築しておく。
TODOアプリの各サービスは作成したこのネットワークに属するようにする。

```
docker container exec -it manager \
docker network create --driver=overlay --attachable todoapp
```

以下の手順で実施する
1. データストアとなるMaster/Slave構成のMySQL Serviceの構築
2. MySQLとデータのやりとりをするためのAPI実装
3. webアプリケーションとAPI間にリバースプロキシとなるNginxを通じてアクセスできるように設定
4. APIを利用してSSRをするWebアプリケーションを実装
5. フロント側にリバースプロキシ(Nginx)を置く

# MySQL Serviceの構築
- Master/Slave構成にする
- コンテナ内部での設定ファイルやスクリプトの扱い
- データベースの初期処理
- Master/Slaveコンテナ間でのレプリケーションの設定の仕方

```
git clone https://github.com/gihyodocker/tododb
```

- 今回Master/SlaveでDockerfileは分けず、一つで2つ作る
- 環境変数の有無によってMaster/Slaveの挙動を制御する

## 認証情報
次の情報を環境変数に登録
- Master
  - rootユーザのパスワード
  - TODOアプリのデータベース
  - データベースユーザ
  - パスワード
- Slave
  - Masterのホスト名
  - rootユーザのパスワード
  - TODOアプリのデータベース
  - データベースユーザー
  - パスワード
  - Masterに登録するレプリケーションユーザー
  - レプリケーションユーザーパスワード

### MySQLのレプリケーションとは
レプリケーションはMySQLが提供する標準機能の一つです．
レプリケーションでは同じデータを持つデータベースを複数用意し，それぞれに同じ操作を行います．
MySQLはデータの複製を他のサーバに持たせる機能を提供しています．

それぞれに命名と役割は以下の通りです．

1. master（マスター）
  クライアントがデータを変更するサーバ
  データが変更されたら変更内容をスレーブに転送する
2. slave（スレーブ）
  マスターの変更内容を受け取り，データベースに反映する

## MySQLのDockerfile
`ch04/tododb:latest`という名前でイメージを作成する
Swarmクラスタのworkerノードで利用できるように`localhost:5000/ch04/tododb:latest`
としてregstryにpushしておく。
→最初にregistryに上げたものが間違っていたので２にしてみた
→それでも動かなかった😢

```
docker image build --platform linux/x86_64 -t ch04/tododb2:latest .
```

```
docker image tag ch04/tododb2:latest localhost:5000/ch04/tododb2:latest
```

```
docker image push localhost:5000/ch04/tododb2:latest
=> latest: digest: sha256:f718f2915af8e21d0b39421c2a58abdb0ac612422375e3e86a74dd738d053dc3 size: 4497
```

## Swarm上でMaster/Slaveサービスを実行する
MasterとSlaveのServiceを構築するyamlを記述し、managerコンテナに向けて
`todo-mysql`スタックとしてデプロイする。
Masterは1、Slaveは2でレプリカがデプロイされていることがわかる。

yamlは以下に置かないといけない
→ コンテナ内のstackと繋がっているホスト側ディレクトリはここになる。
→ docker-compose.ymlはここにあるのでここを主体とする
/Users/kawabeshintarou/Desktop/playgrounds/docker/docker-k8s-extended/chapter3-container-extend/docker-in-docker/stack

ここで、docker-compose downをしていたので以下をやる必要があった
- docker swarm init
- docker swarm join
- docker network create

```
docker container exec -it manager \
docker stack deploy -c /stack/todo-mysql.yml todo_mysql

=> Creating service todo_mysql_master
=> Creating service todo_mysql_slave
```

```
docker container exec -it manager \
docker service ls
```

## MySQLコンテナを確認し初期データを投入する
MassterコンテナがSwarmのどのノードに配置されているかを知る

```
docker container exec -it manager \
docker service ps todo_mysql_master --no-trunc \
--filter "desired-state=running"
```
→NODEが指定されていない模様...


`docker service ps`の--formatオプションを使い特定のコンテナに入るための
コマンドを標準出力に表示する。
(いちいちdocker service psをしてノードのIDとタスクのIDをコピペしなくてすむ)
→ NODEの部分が出てこなかった😢

```
docker container exec -it manager \
docker service ps todo_mysql_master \
--no-trunc \
--filter "desired-state=running" \
--format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"
=> docker container exec -it  docker container exec -it todo_mysql_master.1.oi26yy2d6k3s1umh5v2sa1p2g bash
```

masterコンテナでinit-data.shを実行してテーブルとデータを作成する。

```
docker container exec -it docker container exec -it todo_mysql_master.1.oi26yy2d6k3s1umh5v2sa1p2g \
init-data.sh

docker container exec -it docker container exec -it todo_mysql_master.1.oi26yy2d6k3s1umh5v2sa1p2g \
mysql -u gihyo -pgihyo tododb
```

Master/Slave構成なので、Masterに登録したデータがSlaveにも反映されているか確認
こういう場合は多段docker container execで確認する。

# API Serviceの構築

```
git clone https://github.com/gihyodocker/todoapi
```

エントリポイントはmain.goになる。
環境変数の取得、MySQLへのDB接続処理、HTTPリクエストのハンドラ作成・エンドポイントの登録、サーバーの実行など

