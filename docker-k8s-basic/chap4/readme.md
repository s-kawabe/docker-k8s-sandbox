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
