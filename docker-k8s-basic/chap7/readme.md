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
