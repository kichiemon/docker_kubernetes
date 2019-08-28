# Kubernetesの発展的な利用

常駐型アプリだけでなく、そのほかにも色々利用可能

## Job
1つ以上のPodを生成し、指定された数のPodが正常終了するまでを管理するリソース
- Jobによる全てのPodが正常終了しても、Podは削除されない
    - → 実行結果の分析ができる
    - → バッチ志向のアプリケーションに向いている
- 一度実行する処理
- JobはPodを複数並列実行することで容易にスケールアウト可能
- Jobリソースでは、restartPolicyをAlwaysに設定できない。NeverかOnFailureのどちらかのみ。
- sample [simple-job.yml](try/simple-job.yml)


## CronJob

- Jobが一度きりのPod実行なのに対して、CronJobは定期的に実行できる
- Cronやsystemd-timerなど、定期実行するものに最適
- コンテナと親和性を持ったままスケジューリング可能なのは大きなメリット。
- *マニフェストファイル* で管理する
- sample [simple-cronjob.yml](try/simple-cronjob.yml)


## Secret

機密情報をBase64エンコードした形で扱える
- sample [nginx-secret.yml](try/nginx-secret.yml)
- sample [basic_auth.yml](try/basic_auth.yml)
- ただし、KubernetesのSecretの仕組みを理解している人には簡単に覗かれてしまうので、多層的に対策を施す必要がある
    - リポジトリ、Kubernetes Dashboard、Podへのアクセスをさせないことが何より大事

[参考]
Deploymentで認証情報を環境変数に設定する場合、valueではなくvalueFrom.secretKeyRefを利用する

```
env:
  - name: TODO_MASTER_URL
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: username
```

`kubectl describe pod todoapi-hiasdjfoi` しても認証情報は隠される

## User管理とRBAC

kubernetesにおけるユーザは、認証ユーザーとサービスアカウントの2種類ある。
*Role-Based Access Control (RBAC)* という仕組みで権限制御する

Role-Based Access Control (RBAC) は、Kubernetesのリソースへのアクセスをロールによって制限する機能

- 認証ユーザー
    - kubectlなどでKubernetesを操作するためのユーザー
- サービスアカウント
    - Kubernetesのリソースの一つ
    - Kubernetesクラスタの内部の権限を管理する
        - Service Accountと紐づけられたPodは権限の範囲内でKubernetesリソースの制御が可能となる

e.g.
- デプロイに関わる操作を一部のユーザに制限する
- ログは誰でもみれる
- クラスタ内でBotを動作させる
- replicasの数を草原させる
- admin, deployer, viewwerといったアクセス権限の違いを持つグループを作成し、認証ユーザーのグループを適切に設定することでデプロイ捜査や、ServiceやIngressの変更を認証ユーザーによって制限する
- 大きな構成変更は、強権を持つ認証ユーザーでしか行えないようにする



#### RBAC の利用

二つの要素で成立する
- Kubernetes API のどの操作が可能であるか、を定義づけたロール
- 認証ユーザー・グループ・ServiceAccountとロールの紐付け

Samples
- [cluster-role.yml](try/rbac/cluster-role.yml)
- [cluster-role-binding.yml](try/rbac/cluster-role-binding.yml)
    - subjects：roleを紐づける対象のServiceAccount or Goupを指定
    - roleRef ：紐づけるClusterRole名

#### 認証ユーザー・グループの作成

// 作る
```
$ kubectl create serviceaccount gihyo-user
serviceaccount/gihyo-user created
```

// 確認する
```
$ kubectl get serviceaccount gihyo-user -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-08-25T23:08:45Z"
  name: gihyo-user
  namespace: default
  resourceVersion: "181213"
  selfLink: /api/v1/namespaces/default/serviceaccounts/gihyo-user
  uid: 49a12dc0-c78d-11e9-a915-42010a9200b8
secrets:
- name: gihyo-user-token-drwxr
```

ServiceAccountを作ると、認証情報となるSecreteリソースも同時に作成される
(ここでいう `gihyo-user-token-drwxr`)

// Tokenを取り出してみる。data.tokenが認証に必要なトークン
```
$ kubectl get secret gihyo-user-token-drwxr -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1C......
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2.....
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: gihyo-user
    kubernetes.io/service-account.uid: 49a12dc0-c78d-11e9-a915-42010a9200b8
  creationTimestamp: "2019-08-25T23:08:45Z"
  name: gihyo-user-token-drwxr
  namespace: default
  resourceVersion: "181212"
  selfLink: /api/v1/namespaces/default/secrets/gihyo-user-token-drwxr
  uid: 49a4896f-c78d-11e9-a915-42010a9200b8
type: kubernetes.io/service-account-token
```

デコードするには
```
$ echo 'token'|base64 -D
```

#### ユーザーを利用する

`~/.kube/config`  に利用する認証除法が記載されている

`$ kubectl config view`で確認できる

e.g.
```
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
```
↑contextは、どの認証ユーザーでどのKubernetesクラスタを操作するか、を決める情報

// contextを作成、セットして切り替える
```
$ kubectl config set-credentials gihyo-user --token=y...........Yf4e_8
User "gihyo-user" set.
```

```
$ kubectl config set-context gihyo-k8s-gihyo-user --cluster=gihyo-k8s --user=gihyo-user
Context "gihyo-k8s-gihyo-user" created.
```

```
$ kubectl config use-context gihyo-k8s-gihyo-user
Switched to context "gihyo-k8s-gihyo-user".
```


#### ServiceAccount
クラスタ内で実行されているPodからクラスタ内の他のリソースへのアクセス制御するためのリソース
- これ自体がリソース
- Bot入りのPod作成などに使える

// create service account
[service-account.yml](try/serviceaccount/service-account.yml)

```
$ kubectl apply -f service-account.yml
serviceaccount/gihyo-pod-reader created
```

// ClusterRoleBindingを作成し、ServiceAccountに紐付ける
[cluster-role-binding.yml](try/serviceaccount/cluster-role-binding.yml)

```
$ kubectl apply -f cluster-role-binding.yml
clusterrolebinding.rbac.authorization.k8s.io/pod-reader-binding created
```

// podの一覧を取得するPodを作成。
[get-pod-list.yml](try/serviceaccount/get-pod-list.yml)

```
$ kubectl apply -f get-pod-list.yml
pod/gihyo-pod-reader created

# → Podの一覧を取得できていることを確認
$ kubectl -n kube-system logs -f gihyo-pod-reader
check pod...
NAMESPACE       NAME                                      READY     STATUS              RESTARTS   AGE
docker          compose-6c67d745f6-tkwwh                  1/1       Running             3          14d
docker          compose-api-57ff65b8c7-8swlh              1/1       Running             3          14d
ingress-nginx   nginx-ingress-controller-8544567f-bw877   1/1       Running             1          3d
kube-system     coredns-fb8b8dccf-59974                   1/1       Running             4          14d
kube-system     coredns-fb8b8dccf-kktjv                   1/1       Running             4          14d
kube-system     etcd-docker-desktop                       1/1       Running             3          14d
kube-system     gihyo-pod-reader                          0/1       ContainerCreating   0          2s
kube-system     kube-apiserver-docker-desktop             1/1       Running             3          14d
kube-system     kube-controller-manager-docker-desktop    1/1       Running             4          14d
kube-system     kube-proxy-dxvd6                          1/1       Running             3          14d
kube-system     kube-scheduler-docker-desktop             1/1       Running             3          14d
kube-system     kubernetes-dashboard-5f7b999d65-v5wps     1/1       Running             3          13d
```

// Podの取得以外をできないことを確認（権限の制御）

e.g. Deploymentを取得するPodを作り、できないことを確認する

[get-deployment-list.yml](try/serviceaccount/get-deployment-list.yml)
```
$ kubectl apply -f get-deployment-list.yml
Error from server (Forbidden): error when creating "get-deployment-list.yml": pods "gihyo-deployment-reader" is forbidden: error looking up service account kube-system/gihyo-deployment-reader: serviceaccount "gihyo-deployment-reader" not found
```

## 7.3 Helm

同じアプリケーションを複数の環境にデプロイするとき、環境変数の値が異なるDeploymentマニフェストを用意する必要が出てきたりする。
このマニフェストファイル一式をクラスタ分用意するのは大変...

→ Helmが助けになる

- Helmは、Kubernetes chartsを管理するためのツール。
  - Helmがパッケージ管理ツール
  - Chartがリソースをまとめたパッケージ
- Helmで Chartを管理することで、煩雑なマニフェスト管理をしやすくすることが主な目的
- Chartを中心としたKubernetes開発の統合的な管理ツールとしても使える

    
*kubectlではなく、Helmでデプロイやアップデートを完結させる*


// set up

helm init を行うと、Tillerというアプリがkube-systemにデプロイされている
- Tillerは、helmコマンドの指示を受け取る
- インストールなどの処理を行う

```
kubectl config use-context docker-for-desktop
Switched to context "docker-for-desktop".
```

```
$ helm init
$HELM_HOME has been configured at /Users/ikuya/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

```
$ kubectl -n kube-system get service,deployment,pod --selector app=helm
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/tiller-deploy   ClusterIP   10.108.57.229   <none>        44134/TCP   28s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/tiller-deploy   1/1     1            1           28s

NAME                                READY   STATUS    RESTARTS   AGE
pod/tiller-deploy-c48485567-7v8zk   1/1     Running   0          28s
```

// versionは揃えるべし
```
$ helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```

// upgrade
```
$ helm init --upgrade

$HELM_HOME has been configured at /Users/ikuya/.helm.

Tiller (the Helm server-side component) has been upgraded to the current version.
Happy Helming!
```

#### Helm概念
- クライアントとサーバで構成される
    - クライアント == cli。命令を行う
    - サーバー == KubernetesにインストールされるTiller。Kubernetesに対して処理を行う
- Chart
    - マニフェストを構築するためのテンプレートをまとめたもの
    - Hlemのリポジトリにtgzとして格納される
- リポジトリ
    - lcoal: ローカで作成されたパッケージ
    - stable: 安定した品質を持ったChatが配置される。デフォルトで利用可能
    - incubator: stableの要件を満たしていないもの
    - `$ helm search` でリポジトリを検索
    - helm/chartsというGitHubリポジトリを参照すると、リポジトリを確認できる

// incubatorを追加したい時
```
$ helm repo add incubator http:s//repo.url
```

#### Chartの構成
- `values.yaml` デフォルトvalueファイル
    - 変更したい場合、カスタムvalueファイルを作成する
-

e.g. Redmineのstable/redmine

// install
```
$ helm install -f redmine.yml --name redmine stable/redmine --version 4.0.0
NAME:   redmine
LAST DEPLOYED: Wed Aug 28 23:44:22 2019
NAMESPACE: kube-system
STATUS: DEPLOYED
.....

```

// installを確認。redmineがある
```
$ helm ls
NAME   	REVISION	UPDATED                 	STATUS  	CHART        	APP VERSION	NAMESPACE
redmine	1       	Wed Aug 28 23:44:22 2019	DEPLOYED	redmine-4.0.0	3.4.6      	kube-system
```

// [redmine.yml](helm/redmine.yml) の上書き設定により、 NodePortにデプロイされている
// http://localhost:32213/を開くと、画面を確認できる
```
$ kubectl get service,deployment --selector release=redmine
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/redmine-mariadb   ClusterIP   10.103.252.52   <none>        3306/TCP       99s
service/redmine-redmine   NodePort    10.99.122.142   <none>        80:32213/TCP   99s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/redmine-redmine   0/1     1            0           99s
```


// releaseを更新する
```
$ helm upgrade -f redmine.yml redmine stable/redmine --version 4.0.0
Release "redmine" has been upgraded. Happy Helming!
....
```

// Chartでアプリをアンインストールする

```
$ helm delete redmine
release "redmine" deleted
```

// revisionを確認してロールバックも可能
```
$ helm ls --all
NAME   	REVISION	UPDATED                 	STATUS 	CHART        	APP VERSION	NAMESPACE
redmine	3       	Wed Aug 28 23:48:18 2019	DELETED	redmine-4.0.0	3.4.6      	kube-system

$ helm rollback redmine 3
Rollback was a success! Happy Helming!
```

// 完全に削除
```
$ helm del --purge redmine
release "redmine" deleted
```


// RBACを有効にする

```
# create service account
$ kubectl create serviceaccount tiller --namespace kube-system
serviceaccount/tiller created

$ kubectl create clusterrolebinding tiller-cluster-rule \
  --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
clusterrolebinding.rbac.authorization.k8s.io/tiller-cluster-rule created

$ kubectl patch deploy --namespace kube-system tiller-deploy \
  -p '{"spec":{"template":{"spec":{"serviceaccount":"tiller"}}}}'
deployment.extensions/tiller-deploy patched (no change)
```

// RBACに対応しているJenkinsを試す

- [jenkins.yml](helm/jenkins.yml)
```
$ helm install -f jenkins.yml --name jenkins stable/jenkins
NAME:   jenkins
LAST DEPLOYED: Wed Aug 28 23:53:39 2019
NAMESPACE: kube-system
STATUS: DEPLOYED
```

```
$ kubectl describe sa,clusterrolebinding -l app=jenkins
Name:                jenkins
Namespace:           kube-system
......
```

####  独自のChartを作成する

// Helmのローカルリポジトリ

```
$ helm repo list
NAME  	URL
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts

$ helm serve &
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```


// ひな形作成

```
$ helm create echo
Creating echo
```

// ディレクトリ構成
```
$ tree .
.
└── echo
    ├── Chart.yaml
    ├── charts
    ├── templates
    │   ├── NOTES.txt
    │   ├── _helpers.tpl
    │   ├── deployment.yaml
    │   ├── ingress.yaml
    │   ├── service.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
```

// 流れ：作りたいDeploymentマニフェストを考える　→　必要なChart設定を行う

開発者が修正するのは、deployment.yamlの `spec.template.spec` の部分。

- Deployment : [deployment.yaml](helm/original/echo/templates/deployment.yaml)
- Service : [service.yaml](helm/original/echo/templates/service.yaml)
- Ingress: [ingress.yaml](helm/original/echo/templates/ingress.yaml)

// packageする
```
$ helm package echo
Successfully packaged chart and saved it to: /Users/ikuya/repository/github.com/kichiemon/docker_kubernetes/07_kubernetes_more_use/helm/original/echo-0.1.0.tgz
```

```
$ helm search echo
NAME      	CHART VERSION	APP VERSION	DESCRIPTION
local/echo	0.1.0        	1.0        	A Helm chart for Kubernetes
```

```
$ helm install -f echo.yml --name echo local/echo
NAME:   echo
LAST DEPLOYED: Thu Aug 29 00:25:23 2019
NAMESPACE: kube-system
STATUS: DEPLOYED
```

```
$ kubectl get deployment,service,ingress --selector app=echo
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/echo   1/1     1            1           31s

NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/echo   ClusterIP   10.101.162.87   <none>        80/TCP    31s

NAME                      HOSTS                   ADDRESS   PORTS   AGE
ingress.extensions/echo   ch06-echo.gihyo.local             80      31s
```
