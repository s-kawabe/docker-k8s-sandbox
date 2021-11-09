# Kubernetesのデプロイ・クラスタ構築

# GKEのセットアップ

Google Cloud SDK `gcloud` をインストールする。

```
gcloud components update
```

```
gcloud auth login
```

gcloudで操作する対象のプロジェクトIDを設定する
```
gcloud config set project gihyo-kube-xxxxxxx
```

デフォルトリージョンを選択する (東京)
```
gcloud config set conpute/zone asia-northeast1-a
```

# Kubernetesクラスタの作成

```
gcloud container clusters create gihyo --cluster-version=1.10.4-gke.2 \
  --machine-type-standard-1 \
  --num-nodes=3
```
optionでKubernetesクラスタのバージョン、ノードとなるインスタンスの数を指定する。

作成したクラスタ情報は`gcloud container clusters describe gihyo`、あたは
Cloud ConsoleのKubernetes Engineページで確認できる

```
gcloud container clusters get-credentials gihyo
```
kubectlにgekの認証情報を設定する

```
kubectl get nodes
~~~~~
kubectl proxy
```
作成したクラスタのダッシュボードを表示

```
kubens kube-system
kubectl get pod
```

## Master Slave構成のMySQLをGKE上に構築する

Dockerで永続化データを扱うコンテナを実行するためにはデータボリュームが利用される。
標準のデータボリュームはコンテナをデプロイしたホストに配置されている必要があり、この方針だと
ノード間をまたがってコンテナの再配置を行うような場合、取り回しに手間がかかるデータボリュームは扱いづらい。

そこでホストから分離可能な外部ストレージをボリュームとして利用できる。
Podが別のホストに再配置された時、外部ストレージとしてのボリュームはデプロイされたホストに自動で割り当てられる。
これでホストとデータボリュームという関係が外れて、外部ストレージを利用できるため永続化データを
扱うアプリをコンテナ運用しやすくなる。
次のようなリソースを使って実現する。
- PersistentVolume
- PersistentVolumeClaim
- StrageClass
- StatefulSet

### PersistentVolumeとPersistentVolumeClaim
これらはクラスタが構築されているプラットフォームに対応した永続ボリュームを作成するためのリソース。
PersistentVolumeはストレージの実体。(GKEだとGCEPersistentDisk)
PersistentVolumeClaimはストレージを論理的に抽象化したリソース。

### StarageClass
StorageClassはPersistentVolumeが確保するストレージの種類を定義できるリソース。

