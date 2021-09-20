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


