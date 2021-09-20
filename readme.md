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


