# 実用的なコンテナの構築とデプロイ
**永続化データをどう扱うか**
Dockerコンテナの実行中に書き込まれたファイルは、ホスト側にファイルやディレクトリをマウントしない限り
コンテナを破棄したタイミングで削除されてしまう。
ステートフルな性質を持つアプリをコンテナ運用するためには、新しいバージョンの
コンテナがデプロイされても以前のバージョンを保って利用できることが求められる。
- Data Volumeで各コンテナとホストで永続化データを共有
- Data Volumeコンテナという永続化データ用のコンテナを起動する

## Data Volume(ホストコンテナ間)
dicker container run コマンドのオプションで指定する

```
docker container run [options] -v ホスト側ディレクトリ:コンテナ側ディレクトリ　リポジトリ名[:タグ] [コマンド] [コマンド引数]
```

## Data Volumeコンテナ
**Data Volumeコンテナ**: コンテナのデータ永続化手法として推奨されている。データを持つだけのコンテナ。

これを利用する手法では、コンテナ間でディレクトリを共有する。
DockerVolumeコンテナは、その内容がディスクに保存されるという特性を生かす。
ディスクに保存されたコンテナが持つ永続化データの一部をVolumeとして別のコンテナに共有できるようにしたものがData Volumeコンテナである。

Docker VolumeコンテナのVolumeはDockerの記憶領域であるホスト側の`/var/lib/docker/volumes/`以下に配置されており、
Docker Volumesコンテナ方式はDockerの管理下にあるディレクトリのみに影響する。

コンテナ内のアプリからデータへの操作がカプセル化され、ホストを意識せずにData Volumeを利用できる。

## MySQLのデータをData Volumeコンテナに保持

```Dockerfile
FROM busybox

VOLUME /var/lib/mysql

CMD ["bin/true"]
```

```
docker image build -t example/mysql-data:latest .
```

```
docker container run --platform linux/amd64 -d --name mysql-data example/mysql-data:latest
``` 

// --rm: コンテナを停止すると同時に削除が行われる
```
docker container run -d --rm --name mysql \
環境変数いろいろ
--volumes-from mysql-data \
mysql:5.7
```

`--volumes-from`により、`mysql-data`のData VolumeコンテナをMySQLコンテナにマウントする。

# コンテナ配置戦略
多くのリクエストがくるアプリケーションでは、コンテナを1つのホストに配置するだけでなく
複数のコンテナを複数のホストに配置する必要がある
→複数のコンテナをどのように配置、制御するかという課題がある。
→Composeはあくまでもシングルホストで複数のコンテナを起動するということのみ。

## Docker Swarm
コンテナオーケストレーションツールの一つで、複数のDockerホストを束ねてクライアント化する
Dockerでのコンテナオーケストレーションに関わる名称
- Compose (docker-compose)
  - 複数のコンテナを使うDockerアプリケーションの管理
- Swarm (docker swarm)
  - クラスタの構築や管理を担う(主にマルチホスト)
- Service (docker service)
  - Swarm前提、クラスタ内のService(1つ以上のコンテナの集まり)を管理する
- Stack (docker stack)
  - Swarm前提、複数のServiceをまとめたアプリケーション全体の管理

実践では複数のDockerホストをクラウドサービスなどによって用意するが通常の環境では用意できないので
**「Dockerホストとして機能するDockerコンテナを複数個立てる」**
Docker in Docker (dindコンテナ)

- Docker
  - Docker registory(/var/lib/registry)
    - registry-data(volume mount)
    - post:5000
  - manager
  - worker01
  - worker02
  - worker03

上の構成でやる

### registry
registryはDockerレジストリとなるコンテナで、managerやworkerコンテナから参照される。
実際はDocker Hubなどを使ってイメージを取得する場合がほとんど。
registryコンテナのデータは永続化のためにホストにマウントしておく。

### manager
Swarmクラスタ全体を制御する。
workerと呼ばれるDockerホストへServiceが保持するコンテナを適切に配置する。
これらをComposeで構築する

```yml
version: "3"
services:
  registry:
  manager:
  worker01:
  worker02:
  worker03:
```

```
docker-compose up -d
docker container ls
```

→ 5つのコンテナが立ち上がっていることが確認できる
これだけだとdindコンテナを複数実行しただけなので、クラスタとして動作するわけではない。
クラスタを管理する役割を担うmanagerが必要。
ホストからmanagerコンテナに対して`docker swarm init`を実行してSwarmのmanagerに設定する。
`JOINトークン`が発行される。DockerホストをSwarmクラスタのworkerとして
登録するにはこれが必要になる。

```
docker container exec -it manager docker swarm init
```

```
Swarm initialized: current node (tnmsjmfedixtoucv888xf1s45) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-49zppl3pyj2qkv819rl4azjey3vaq5h57bwto47n5miuzlt77x-bjheo9ayl24y81y7fqbz41bpf 172.19.0.3:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

JOINトークンを利用して3つのノードをSwarmクラスタにworkerとして登録する。

```
docker container exec -it worker01 docker swarm join \
--token SWMTKN-1-49zppl3pyj2qkv819rl4azjey3vaq5h57bwto47n5miuzlt77x-bjheo9ayl24y81y7fqbz41bpf manager:2377
```

```
This node joined a swarm as a worker.
```

これを3つのコンテナで行い、
Swarmクラスタの状態をmanager上で`docker node ls`で確認する。

```
docker container exec -it manager docker node ls
```

```
rz7w3n9e1ooj7z7cv5s6vfqpv     5c59b4d07b38        Ready               Active                                  18.05.0-ce
tnmsjmfedixtoucv888xf1s45 *   5fb4185e628d        Ready               Active              Leader              18.05.0-ce
se8pz0ut9k8naz8kle0knoonk     848b297ef350        Ready               Active                                  18.05.0-ce
t3wegxj4197fn858gnu8lsdr8     b35810b975ba        Ready               Active                                  18.05.0-ce
```

### DockerレジストリにイメージをPushする
registryコンテナが実行状態なのでホスト側からregistryコンテナに対してイメージをpushする。
**外側のDockerでビルドしたDockerイメージはレジストリ経由で取得する必要がある**

```
docker image tag example/echo:latest localhost:5000/example/echo:latest
```
レジストリのホストはイメージのpush先およびpull先のレジストリを表す。
今回作成したregistryコンテナはホスト側からlocalhost:5000でアクセスできるため、
リポジトリ名の前にこれを付与することで、このタグをそのまま`docker image push`コマンドの
引数に指定することで指定のレジストリにイメージをpushできる。

```
docker image push localhost:5000/example/echo:latest
```

### workerコンテナでregistryコンテナからpull
registryにpushしたものをdocker image pullできるか試す。
worker01からレジストリはregistryで名前解決ができる。

```
docker container exec -it worker01 docker image pull registry:5000/example/echo:latest
```

→worker01コンテナ内でexample/echo:latestイメージが取得できた。

## Service
Service: アプリケーションを構成する一部のコンテナを制御するための単位

serviceを作詞するために、Dockerイメージは先ほどregistryコンテナにpushしたものを利用。
Serviceはmanagerコンテナ内から`docker service create`で作成。

```
docker container exec -it manager \
docker service create --replicas 1 --publish 8000:8080 --name echo registry:5000/example/echo:latest 
```

`docker service ls`でServiceの一覧がわかる

```
docker container exec -it manager docker service ls
```

制御するレプリカを増やせた。
`docker service scale`で該当Serviceのコンテナの数を増減させられる。
コンテナをスケールさせる際に使う?

```
docker container exec -it manager docker service scale echo=6
```

=> 

```
echo scaled to 6
overall progress: 6 out of 6 tasks 
1/6: running   [==================================================>] 
2/6: running   [==================================================>] 
3/6: running   [==================================================>] 
4/6: running   [==================================================>] 
5/6: running   [==================================================>] 
6/6: running   [==================================================>] 
verify: Service converged 
```

Swarmクラスタ上で実行されているコンテナを確認すると、6個のechoコンテナが実行されている。
これらは**ServiceによってSwarmクラスタのノードに分散して配置されている。**


Service 削除しておく
```
docker container exec -it manager docker service rm echo

docker container exec -it manager docker service ls
```

上からわかるようにServiceはコンテナを利用したアプリケーションを簡単にスケールアウトさせられる。
Serviceなしでこれを行うには`docker container run`を繰り返し手動で行わなければいけない。

## Stack
Stack: 複数のServiceをグルーピングした単位、アプリケーションの全体の構成を定義する。

上のServiceは1つのアプリケーションイメージしか扱うことができないが、
複数のserviceが協調して動作することで成立するアプリケーションも多く存在する。
これを解決する上位概念がStack。
**Swarm上でのスケールイン・スケールアウトが可能になったCompose** 的な位置付け。
　
StackによってデプロイされるService群はoverlayネットワークに所属する。
overlayネットワークは複数のDockerホストにデプロイされているコンテナ群を同じネットワークに
配置させることができる。

Serviceから別のoverlayネットワークに存在するServiceをディスカバリすることはできない。

クライアントと宛先のServiceが同一のoverlayネットワークに所属させる必要がある。
ch03というoverlayネットワークを作成しておき、Stackから作られる各Serviceをこの中に所属させる。

```
docker container exec -it manager docker network create --driver=overlay --attachable ch03
```

以下をstackディレクトリに作成する。

```yml
version: "3"
services:
  nginx:
    image: gihyodocker/nginx-proxy:latest
    deploy:
      replicas: 3
      placement: 
        constraints: [node.role != manager]
    environment:
      BACKEND_HOST: echo_api:8080
    depends_on:
      - api
    networks:
      - ch03
  api:
    image: registry:5000/example/echo:latest
    deploy:
      replicas: 3
      placement:
        constraints: [node.role != manager]
    networks:
      - ch03

networks:
  ch03:
    external: true
```

フロントエンドにNginxがあり、バックエンドのAPIに対してリバースプロキシする構成

### docker stack サブコマンド
- deploy
  - 新規にStackをデプロイ、または更新
- ls
  - デプロイされているStackの一覧を表示する
- ps
  - Stackによってデプロイされているコンテナの一覧表示
- rm
  - デプロイされているStackを削除
- services
  - Stack内のService一覧を表示

### Stackをデプロイ
stackディレクトリはmanagerコンテナの/stackにマウントされている。

```
docker container exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo
```

```
Creating service echo_nginx
Creating service echo_api
```

echoスタックのServiceの一覧表示

```
docker container exec -it manager docker stack services echo
```

### Stackでデプロイされたコンテナを確認

```
docker container exec -it manager docker stack ps echo
```

### visualizerで確認

visualizer.ymlを stackディレクトリに作成

```yml
version: "3"

services:
  app:
    image: dockersamples/visualizer
    ports:
      - "9000:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      mode: global
      placement:
        constraints: [node.role == manager]
```

deploy.mode.globalは特定のコンテナをクラスタ上の全ノードに配置できる設定。
ポートはホストからmanagerへ9000:9000なのでmanagerからvisualizerコンテナへは
9000:8080で繋げられるようにする。

```
docker container exec -it manager docker stack deploy -c /stack/visualizer.yml visualizer
```

```
docker container exec -it manager docker stack rm echo
```

## ServiceをSwarmクラスタ外から利用する

# まとめ
- Serviceはレプリカ数（コンテナの数）を制御することで容易にコンテナを複製でき、複数のノードに配置できるためスケールアウトへの親和性が高い
- Serviceによって管理される複数のレプリカはService名で名前解決でき、かつServiceへのトラフィックはレプリカへ分散される
- Swarmクラスタの外からSwarmのServiceを利用するには、Serviceにトラフィックを分散するためのプロキシを用意する
- Stackは複数のServiceをグルーピングでき、複数のServiceで形成されるアプリケーションのデプロイに役立つ