# Kubernetes入門

Kubernetesとは、Google社主導で開発された、コンテナの運用を自動化するためのコンテナオーケストレーションシステム。

- コンテナオーケストレーションを実現・管理するための統合的なシステム
- 操作のためのAPIやCLIツールも提供する
- デプロイなど、運用管理の自動化を実現できる
    - Serverのリソースを考慮したコンテナ配置、スケーリングなどなど
- Dockerコンテナを管理するという意味ではKubernetesはSwarmとほぼ同じ立ち位置
- Kubernetes入門はCompose/Stack/Swarmの機能を統合しつつ、より高度に管理できるもの


## 環境


Kubernetesのインストール

```
$ curl -LO https://storage.googleapis.com/kubernetesrelease/release/v1.10.4/bin/darwin/amd64/kubectl \
  && chmod +x kubectl\
  && mv kubectl /usr/local/bin/
```

Kubernetesのデプロイ
e.g. ダッシュボード https://github.com/kubernetes/dashboard
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
```
$ kubectl proxy
```

```
$ open http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

## Kubernetes概念
Kubernetesクラスタ内のリソースには↓がある

- Node
    - クラスタが持つ最も大きな概念
    - クラスタ管理下に登録されているDockerホストのこと。コンテナのデプロイのために利用される
    - クラスタには全体を管理するサーバであるMasterが少なくとも一つが配置される
    - GCPではGCE、AWSではEC2がNode
    - クラスタに配置されているNodeの数、Nodeのマシンスペックによって配置できるコンテナの数は変わってくる
    - クラスタに参加しているNodeを取得する
        - `$ kubectl get nodes`
```
$ kubectl get pod --namespace=kube-system -l k8s-app=kubernetes-dashboard
NAME                                    READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-5f7b999d65-v5wps   1/1     Running   2          9d
```

- Namespace
    - クラスタの中に入れことして配置する仮想的なクラスタ
    - Namespaceの一覧を取得する `$ kubectl get namespace`
    - 使いどころとして
        - 開発者それぞれのNamespaceを用意することでメインのNamespaceが散らかるのを防ぐ
        - Namespaceごとに権限を設定できるので堅牢で細やかな権限制御
- Pod
    - コンテナの集合体の単位
    - 少なくとも一つのコンテナを持つ
    - Podないのコンテナは全て同一のNodeに配置される
    - Podの粒度
        - 1つのNodeに配置されるという特性に着目
        - →　同時にデプロイしないと整合性を保つことができない時は同一Podにまとめる
    - kind: Pod
    - `kubectl get pod`
    - docker execと同じうように `kubectl exec -it simple-echo sh -c nginx` でコンテナに入れる
    - `kubectl logs -f simple-echo -c echo` でログをtailできる
    - `kubectl delete pod simple-echo` でPodを削除
    - `kubectl delete -f simple-pod.yml` でマニフェストファイルベースで削除できる
- ReplicaSet
    - 同一の複数のPodを実行する単位
    - 一定規模以上になってくるとスケールしていかないと。可用性を高める。
    - kind: ReplicaSet
    - Podの数は replicas: 3
    - `$ kubectl apply -f simple-replicaset.yml` 
- Deployment
    - アプリケーションのデプロイの基本単位
    - ReplicaSetを管理・操作するために提供されている
    - 定義自体はReplicaSetとさほど変わらないが、ReplicaSetの世代管理をできることが特徴
    - `--record` でkubectlのコマンドを記録できる
    - `kubectl rollout history deployment echo` でリビジョンを確認できる
    - 運用面で言えば、ReplicaSetをそのまま使うことはなく、Deploymentを使うことがほとんど
    - ReplicaSetの更新はどんな風にされるのか?
        - Pod数のみを変更しても新規のReplicaSetは生まれない
        - コンテナ定義を更新すると、revisionが上がり新しく作られる
    - `rollout history` でrevisionを観れる
    - `rollout history deployment echo --revision=1` とすることで、指定したrevisionの状態を見れる
    - `kubectl rollout undo deployment echo` でロールバックできる
- Service
    - クラスタ内において、Podの集合(ReplicaSet)に対する経路やサービスディスカバリを提供するもの
    - Serviceでは、関連づけたいPodのラベルを指定するとターゲットとして対象となる → Service経由でトラフィックが流れるようになる
    - 基本的にServiceはクラスタの中からしかアクセスできない
        - Kubernetesクラスタ内に一時的なデバッグコンテナをデプロイして、curlでhttpリクエストを送ると動作確認できる
            - 接続用のコンテナをデプロイ `kubectl run -i --rm --tty debug --image=gihyodocker/fundamental:0.1.0 --restart=Never -- bash -il`
    - Kubernetes内のアプリケーション同士を協調させるような時に有用。
        - Service内で名前解決できる
        - ターゲットのPodにリクエストを分散することもできる
    - Serviceの種類
        - ClusterIP Service
            - デフォルトはこれ
            - Kubernetesクラスタ上の内部IPアドレスにServiceを公開できる
            - あるPodから別のPod軍へのアクセスをServiceを介して実行でき、Service名で名前解決できる
            - ただし、外からはリーチできない
        - NodePort Service
            - クラスタ外からアクセスできるService
            - 各ノードからServiceポートに接続するためのぐろーばるなポートを開ける
                - `$ kubectl get svc echo` でえられるport にcurl できる
        - LoadBalancer Service
            - ローカルのKubernetes環境では利用できない
            - 主にクラウドプラットフォームで提供されているロードバランサーと連携するためのもの
                - e.g. GCPではCLB, AWSではALB
        - ExternalName Service
            - selectorもport定義も持たない特殊なService
           - Kubernetesクラスタ内部から、外のホストを解決するためのエイリアスを提供する
```
/// 参考
apiVersion: v1
kind: Service
metadata:
  name: gihyo
spec:
  type: ExternalName
  externalName: gihyo.jp
```
- Ingress
    - ServiceのKubernetesクラスタ外への公開
    - VircualHost, パスベースのHTTPルーティングを提供
    - HTTP/HTTPSのサービス公開するにはIngressを利用
    - パブリッククラウドでは提供されているLBを利用できる
        - e.g. GCPではCLB, AWSではALB
- ConfigMap
    - 設定情報を定義し、Podに提供する
- PersistentVolume
    - Podが利用するストレージのサイズや種別を定義する
- PersistentVolumeClaim
    - PersistentVolumeを動的に確保する
- StorageClass
    - PersistentVolumeが確保するストレージの種別を定義
- StatefulSet
    - 同じ仕様で一意性のあるPodを複数生成・管理する
- Job
    - バッチ系
    - 常駐でない複数のPodを生成できる
- CronJob
    - cron記法でスケジューリングして実行されるJob
- Secret
    - 機密データの定義
- Role
    - Namespace内で操作可能なKubernetesリソースのルールを定義
- RoleBinding
    - RoleとKubernetesリソースを利用するユーザーを紐付け
- ClusterRole
    - Cluster全体で操作可能なKubernetesリソースのルールを定義する
- ClusterRoleBinding
    - ClusterRoleとKubernetesリソースを利用するユーザーを紐付ける
- ServiceAccount
    - PodにKubernetesリソースを操作させる際に利用するユーザ

### 補足

利用できるapiVersionは　`kubectl api-versions` で確認できる

```
$ kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
compose.docker.com/v1alpha3
compose.docker.com/v1beta1
compose.docker.com/v1beta2
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```
