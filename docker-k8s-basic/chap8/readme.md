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

## kubernetesを動かす
- kubernetesのインストール(Docker desktopからEnable kubernetesの設定にチェックを入れる)

### **定義ファイルの書き方**
- k8sはマニフェストファイルで登録されたないように従ってPodを作成する。
- 定義ファイルの内容をapplyするとそれがetcdに読み込まれてサーバーがその状態を保つようになる
- 定義ファイルはリソース単位で記述する。
    - 主に最初に作成するのはデプロイメントかサービスになる。
    →Podを作るのだが、Podとして定義ファイルを作成する場合は、
     Podしか作成されないので、それを抱擁する大きなまとまりである
     サービスやデプロイメントとして定義することもある。
    - デプロイメントとして定義すると、それはレプリカセットとPodを定義したことにもなる