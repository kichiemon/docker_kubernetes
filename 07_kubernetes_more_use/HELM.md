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

// curlで叩いてみる
```
$ curl http://localhost -H 'Host: ch06-echo.gihyo.local'
Hello Docker!!⏎
```

####  独自のChartリポジトリを作成する
↑で作成したものはLocalリポジトリで利用可能だが、チーム開発だと、リモートに欲しくなる

- Helmのリポジトリは、Chartアーカイブファイル(.tgz)とリポジトリのインデックスふぁいる(index.yaml)があって、http/httpsで取得できれば成立する
- GitHub Pagesにホスティングすると簡単

// Githubのリポジトリを作成→clone→...

```
$ git checkout -b gh-pages
Switched to a new branch 'gh-pages'
```

```
$ mkdir stable
$ cd stable/

$ helm create example
Creating example

$ helm package example
Successfully packaged chart and saved it to: /Users/ikuya/repository/github.com/kichiemon/charts/stable/example-0.1.0.tgz

$ helm repo index .
```

```
$ tree .
.
├── example
│   ├── Chart.yaml
│   ├── charts
│   ├── templates
│   │   ├── NOTES.txt
│   │   ├── _helpers.tpl
│   │   ├── deployment.yaml
│   │   ├── ingress.yaml
│   │   ├── service.yaml
│   │   └── tests
│   │       └── test-connection.yaml
│   └── values.yaml
├── example-0.1.0.tgz
└── index.yaml
4 directories, 10 files
```

```
$ helm repo add gihyo-stable https://kichiemon.github.io/charts/stable
"gihyo-stable" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "gihyo-stable" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

https://github.com/kichiemon/charts.git にPUSHした

