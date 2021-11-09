# Kubernetes入門

# kubectlのインストール
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
```

kubectilバイナリを実行可能にする
```
chmod +x ./kubectl
```

PATHを通す
```
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
```

```
kubectl version --client
=> Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.3", GitCommit:"c92036820499fedefec0f847e2054d824aea6cd1", GitTreeState:"clean", BuildDate:"2021-10-27T18:41:28Z", GoVersion:"go1.16.9", Compiler:"gc", Platform:"darwin/arm64"}
```

kubernetesにデプロイされているコンテナを可視化できる管理画面のインストール
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

```
kubectl get pod --namespace=kube-system -l k8s-app=kubernetes-dashboard
```

```
kubectl proxy
```

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
にアクセスするとtokenを求められる

```
kubectl -n kube-system get secret | grep deploy
```

```
kubectl -n kube-system describe secret [secret-name]
```
tokenが取得できるのでこれでサインインする。
管理画面が開ける。

# Kubernetesの概念
- Node
  - Kubernetesクラスタで実行するコンテナを配置するためのサーバ
- NameSpace
  - Kubernetesクラスタ内で作る仮想的なクラスタ
- Pod
  - コンテナ集合体の単位で、コンテナを実行する方法を定義する
- ReplicaSet
  - 同じ仕様のPodを複数生成・管理する
- Deployment
  - ReplicaSetの世代管理をする
- Service
  - Podの集合にアクセスするための経路を作る
- Ingress
  - ServiceをKubernetesクラスタの外に公開
- ConfigMap
  - 設定情報を定義し、Podに供給する
- PersistentVolume
  - Podが利用するストレージのサイズや種別を定義する
- PersistentVolumeClaim
  - PersistentVolumeを動的に確保する
- StorageClass
  - PersistentVolumeが確保するストレージの種類を定義する
- StatefulSet
  - 同じ仕様で一貫性のあるPodを複数生成・管理する
- Job
  - 常駐目的ではない複数のPodを作成し、正常終了することを確保する
- CronJob
  - cron記法でスケジューリングして実行されるJob
- Secret
  - 認証情報などの機密データを定義する
- Role
  - Namespace内で操作可能なKubernetesリソースのルールを定義する
- RoleBinding
  - RoleとKubernetesリソースを利用するユーザを紐づける
- ClusterRole
  - Cluster全体で操作可能なKubernetesリソースのルールを定義する
- ClusterRoleBinging
  - ClusterRoleとKubernetesリソースを利用するユーザを紐づける
- ServiceAccount
  - PodにKubernetesリソースを操作させる際に利用するユーザー

# KubernetesクラスタとNode
Nodeはクラスタが持つリソースで最も大きな概念である。
Nodeはkubernetesクラスタの管理下に登録されているDockerホストのこと。
Kubernetesクラスタには全体を管理するサーバである**Masterが少なくとも１つは配置されている**

```
kubectl get nodes
->
NAME             STATUS   ROLES                  AGE    VERSION
docker-desktop   Ready    control-plane,master   2d2h   v1.21.3
```
ローカル環境のkubernetesであれば、クラスタ作成時に作られたVMがNodeの１つとして登録されている。
**クラウドで動作させているKubernetesであればGCPにおけるGCE、AWSにおけるEC2のインスタンスがNodeになる。**

## Masterを構成する管理コンポーネント
KubernetesのMasterサーバーにデプロイされる管理コンポーネントは
次のようなもので構成される。

### kube-apiserver
kubernetesのAPIを公開するコンポーネント。
kubectlからのリソース操作を受け付ける。

### etcd
高い可用性を備えた分散キーバリューストアで、
Kubernetesクラスタのパッキングストアとして利用される。

### kube-scheduler
Nodeを監視してコンテナを配置する最適なNodeを選択する。

### kube-controller-manager
リソースを制御するコントローラーを実行する。

→**GKEではKubernetesのフルマネージドサービスなので開発者がMasterの存在を意識する必要はほぼない。**

# Namespace
```
kubectl get namespace
```
Namespaceによってクラスタの中に入れ子となる仮想的なクラスタを作成できる。
これは一定の規模を有するチーム開発で有用で、開発者ごとにNamespaceを用意すると綺麗に整理できる

# Pod
Podはコンテナの集合体の単位で少なくとも１つのコンテナを持つ。
k8sとDockerの場合Podが持つのはDockerコンテナ1つ以上となる。
コンテナ設計の関係で、ひとまとめにしたい2つ以上のコンテナ(NginxとGoなど)がある場合
Podでまとめてデプロイする。
Pod必ずどこかのNodeに配置される、同じPodを複数のNodeに配置する事も可能。

なお、Masterノードは管理用のコンポーネントのPodだけがデプロイされているNodeとなる。
ここにはアプリケーション用のPodはデプロイされない。

## Podの作成とデプロイ
Podの作成はkubectlで行う事もできるが、バージョン管理の観点としてもyamlファイルで定義することがほとんど。
こういった定義ファイルはマニフェストファイルと呼ばれる。

```yml
apiVersion: v1
# k8sのリソースの種類を指定する
kind: Pod
# このリソースの名称として設定されるメタデータ
metadata:
  name: simple-echo
spec:
  containers:
  - name: nginx
    image: gihyodocker/nginx-proxy:latest
    env:
    - name: BACKEND_HOST
      value: localhost:8080
    # EXPOSEするポートを指定 
    ports:
    - containerPosrt: 80
  - name: echo
    image: gihyodocker/echo:latest
    ports:
    - containerPort: 8080
```

これを`apply`することでローカルk８sにPodを作成する

```
kubectl apply -f simple-pod.yaml
-> pod/simple-echo created
```

```
kubectl get pod
```
→Podの状態を取得

```
kubectl exec -it simple-echo sh -c nginx
```
→docker container execのようにコンテナの中に入る(-c で中のコンテナを指定する)

```
kubectl logs -f simple-echo -c echo
```
→kubectl logsでPos内のコンテナの標準出力を表示できる

```
kubectl delete pod simple-echo
```
→Podの削除、Pod以外のリソースもこのコマンドで実行できる

```
kubetl delete -f simple-pod-yaml
```
→マニフェストファイルベースでもPodを削除できる。
このファイルに定義されたリソース全てが削除される。

# ReplicaSet
Podを定義したマニフェストファイルからは1つのPodしか作成できない。
ReplicaSetを定義することで同じ仕様のPodを複数生成・管理できる。

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo
  labels:
    app: echo
spec:
  # 作成するPod数
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template: # template以下はPodリソースにおける定義と同じ
    metadata:
      Labels:
        app: echo
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        # EXPOSEするポートを指定 
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```

```
kubectl apply -f simple-replicaset.yaml
```

```
kubectl get pod
```
ReplicaSetを操作してPodの数を減らすと、減らした分のPodは削除される。
削除されたPodは復元できない。

```
kubectl delete -f simple-replicaset.yaml
```

# Deployment
ReplicaSetよりも上位のリソース。
アプリケーションデプロイの基本単位となるリソースで、
ReplicaSetは同一仕様Podのレプリカの数を管理。制御するためのリソースだったが、
DeploymentはReplicaSetを管理・操作するために用意されている。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template: # tepmlate以下はPodリソースにおけるspec定義と同じ
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        # EXPOSEするポートを指定 
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```

```
kubectl apply -f simple-deployment.yaml --record
```
--recordオプションで実行コマンドの記録を印字する

```
kubectl get pod,replicaset,deployment --selector app=echo    
```

```
kubectl rollout history deployment echo
->
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yaml --record=true
```
このようにDeploymentリビジョンが確認できる。

実運用ではReplicaSetを直接定義することは少なく、Deploymentのマニフェストファイルを扱う
運用がほとんどである。

### ReplicaSetライフサイクル
- Pod数のみを更新しても新規ReplicaSetは生まれない
  - これだけだと新しいリビジョンには切り替わらない
- コンテナ定義を更新
  - コンテナ定義を更新するとPodが新しくなる
- kubectl rollout undo deploymend echo
  - のようにやると直前のリビジョンにロールバックできる

# Service
ServiceはPodの集合（主にReplicaSet）に対する経路やサービスディスカバリを提供する。
Serviceのターゲットとなる一連のPodはServiceで定義するラベルセレクタによって決定される。

例としてreplicasetの定義ファイルにreleaseとラベルをつけてReplicaSetを２つ定義する

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-spring
  labels:
    app: echo
    release: spring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      release: spring
  template:
    metadata:
      labels:
        app: echo
        # 追加した
        release: spring
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        # EXPOSEするポートを指定 
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-summer
  labels:
    app: echo
    # 追加した
    release: summer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
      # 追加した
      release: summer
  template:
    metadata:
      labels:
        app: echo
        # 追加した
        release: summer
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        # EXPOSEするポートを指定 
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```

applyするとreleaseラベルにspringとsummerを持つPodがそれぞれ実行されていることがわかる。
ここでrelease=summerを持つPodだけにアクセスできるようなServiceを作ってみる。
spec.selector属性にはServiceのターゲットとしたいPodが持つラベルを設定する。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
    release: summer
  posrts:
    - name: http
      port: 80
```

PodのラベルがServiceにセレクタで定義しているラベルと合致した場合、対象のPodは
そのServiceのターゲットとなり、Service経由してトラフィックが流れるようにする。

```
kubectl apply -f simple-service.yaml
```

実際にrelease=summerを持つPodだけのトラフィックが流れるか確認する。
役割ごとに適切にService名を割り振って名前解決を行うことが大事。

## ClusterIP Service
作成されるServiceにはさまざまな種類があり、yaml内で定義できる。
デフォルトは**ClusterIP Service**。
ClusterIPではKubernetesのクラスタ上の内部IPアドレスにServiceを公開できる。
あるPodから別のPod群へのアクセスはServiceを介して行うことができ、かつService名で名前解決ができるようになる。

## NodePort Service
クラスタ外からアクセスできるService。
ClusterIPを作るという点ではClusterIP Serviceと同じだが
各ノード上からServiceポートへ接続するためのグローバルなポートを開けるという違いがある。

## LoadBalancer Service
ローカルKubernetes環境では利用できないService。
主に各クラウドプラットフォームで提供されているロードバランサーと連携するためのもの。
GGP: Cloud Load Balancing
AWS: Elastic Load Balancing

## ExternalName Service
selectorもport定義ももたない特殊なService。
Kubernetesクラスタ内から外部のホストを解決するためのエイリアスを提供する。

# Ingress
Kubernetesクラスタの外にServiceに公開するためにはServiceをNodePortで公開することで可能だが
あくまでもL4層レベルまでしか扱えないため、HTTP/HTTPSのようにパスベースに
転送先のServiceを切り替えるといったL7層レベルの制御までは行えない。
IngressはServiceのKubernetesクラスタの外への公開と、VirtualHostや
パスベースでの高度なHTTPルーティングを両立する。
**HTTP/HTTPSのサービスを公開するユースケースではほぼIngressを使用することになる。**

ingressはL7層のルーティングが可能なので、VirtualHostの仕組みのように指定した
ホストやパスに合致したサービスにリクエストを委譲できる。

クラスタの外からのHTTPリクエストをServiceにルーディングするためには`nginx_ingress_controller`
をデプロイする必要がある。

# Job
Kubernetesは常駐型サーバーアプリケーション以外にも、ジョブサーバーなど多様な使い道がある。
Jobは1つのPodを作成し、**指定された数のPodが正常に完了するまでを管理するリソース。**
Jobによる全てのPodが正常に終了しても、Podは削除されずに保持されるため終了後にPodのログや実行結果を分析できる。
常駐型アプリケーションよりも大規模な計算やバッチ志向のアプリケーションに向いている。
JobはPodを複数並列で実行することで容易にスケールアウトできる。
Podとして実行することでKubernetesのServiceと連携した処理を行いやすいという面もある。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pingpong
  labels:
    app: pingpong
spec:
  parallelism: 3
  template:
    metadata:
      labels:
        app: pingpong
    spec:
      containers:
      - name: pingpong
        image: gihyodocker/alpine:bash
        command: ["bin/sh"]
        args:
          - "-c"
          - |
            echo [`date`] ping!
            sleep 10
            echo [`date`] pong!
        restartPolicy: Never
```

# Cronjob
Jobは一度きりのPodの実行だが、CronJobリソースを利用するとスケジューリングして定期的にPodを実行できる。

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pingpong
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: pingpong
        spec:
          containers:
          - name: pingpong
            image: gihyodocker/alpine:bash
            command: ["/bin/sh"]
            args:
            - "-c"
            - |
            echo [`date`] ping!
            sleep 10
            echo [`date`] pong!
        restartPolicy: OnFairure
```

spec.scheduleにCron記法でPosの起動スケジュールを定義できる。
CronJobのマニフェストファイルを適用するとジョブが作成され、指定したCronの条件基づいた
スケジュールでPosを作成する。
