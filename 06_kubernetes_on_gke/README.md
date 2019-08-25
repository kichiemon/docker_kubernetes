# 6章 Kubernetesのデプロイ・クラスタ構築

## GKE setup

```
$ gcloud components update 
$ gcloud auth login 
```

// set config
```
$ gcloud config set project PROJECT_ID
$ gcloud config set compute/zone asia-northeast1-a
```

// create cluster
```
$ gcloud container clusters create gihyo --cluster-version=1.11.10-gke.6 --machine-type=n1-standard-1 --num-nodes=3
NAME   LOCATION           MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
gihyo  asia-northeast1-a  1.11.10-gke.6   35.194.103.129  n1-standard-1  1.11.10-gke.6  3          RUNNING
```

// クラスタを操作するための認証情報をえる
```
$ gcloud container clusters get-credentials gihyo

Fetching cluster endpoint and auth data.
kubeconfig entry generated for gihyo.

# nodesを取得できたらOk
$ kubectl get nodes
NAME                                   STATUS   ROLES    AGE   VERSION
gke-gihyo-default-pool-b40f17f8-flb5   Ready    <none>   4m    v1.11.10-gke.6
gke-gihyo-default-pool-b40f17f8-s246   Ready    <none>   4m    v1.11.10-gke.6
gke-gihyo-default-pool-b40f17f8-zn3s   Ready    <none>   4m    v1.11.10-gke.6
```

// StorageClassの生成 [storage-class-ssd.yml](try/storage-class-ssd.yml)
```
$ kubectl apply -f storage-class-ssd.yml
storageclass.storage.k8s.io/ssd created
```


// StatefulSetも生成 [msyql-master.yml](try/mysql-master.yml), [msyql-slave.yml](try/mysql-slave.yml), 
```
$ kubectl apply -f mysql-master.yml
service/mysql-master unchanged
statefulset.apps/mysql-master created

$ kubectl apply -f mysql-slave.yml --validate=false
service/mysql-slave unchanged
statefulset.apps/mysql-slave created
```

// 作成されたことを確認。podのIDが連番になっている
```
$ kubectl get pod
NAME             READY   STATUS    RESTARTS   AGE
mysql-master-0   1/1     Running   0          8m
mysql-slave-0    1/1     Running   0          1m
mysql-slave-1    1/1     Running   0          33s
```

// データ作成しReplicaに同期されていることを確認
```
$ kubectl exec -it msyql-master-0 init-data.sh

$ root@mysql-slave-0:/# mysql -u root -pgihyo tododb -e "SHOW TABLES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------------+
| Tables_in_tododb |
+------------------+
| todo             |
+------------------+
```

// todo APIをデプロイする [todo-api.yml](try/todo-api.yml)
```
$ kubectl apply -f todo-api.yml
service/todoapi created
deployment.apps/todoapi created
```

// todo Web Appをデプロイする [todo-api.yml](try/todo-api.yml)
```
$ kubectl apply -f todo-web.yml
service/todoweb created
deployment.apps/todoweb created
```

emptyDirにはどういつPod内のそれぞれから好きなパスでアクセスできる
volumeMountsでは、作成した仮想Volumeをコンテナにマウントしている
マウントするコンテナのパスはmountPathで指定する
```
nginx: /var/www/_nuxt
web: /dist
```

仮想Volumeでコンテナ間のディレクトリ共有が可能になる。
- → 仮想Volumeはからなので、コンテナにマウントされたものも空
- → webコンテナのassetsファイルをNginxコンテナにコピーする必要がある
    - → Lifecycleイベントを利用する
    - → コンテナの開始・終了時に任意のコマンドを実行できる
    - LifecycleイベントではDockerfileに手を加える必要がない


あとで公開するために、NodePortを指定している。
```
$ kubectl get svc todoweb
NAME      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
todoweb   NodePort   10.55.255.3   <none>        80:32124/TCP   8m
```

// IngressでWebAppをインターネットに公開する [ingress.yml](try/ingress.yml)
- GCPのLoadBalancerが利用される
```
$ kubectl apply -f ingress.yml
ingress.extensions/ingress created

$ kubectl get ingress
NAME      HOSTS   ADDRESS   PORTS   AGE
ingress   *                 80      32s
```

## MySQLの配置

Dockerで永続化データを扱うコンテナを実行するためには、データボリュームが利用される
- 標準だと、デプロイしたホストに配置されている必要があるが、それだとNode感をまたがってコンテナを再配置するような場合に取り回しにくい...
    - → Kubernetesだと、ホストから分離可能な外部ストレージをボリュームとして利用できる   
    - → デプロイされたホストに自動で割り当てられる
これを実現するリソース
- PersistentVolume
    - ストレージの実体。GCPでは、GCEPersistentDisk
- PersistentVolumeClaim
    - 抽象化したもの
    - PersistentVolumeに対して必要な容量を動的に確保する
```.yaml
apiversion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd # StorageClassのリソース名のこと
  resources:
    requests:
      storage: 4Gi # 容量を指定
```
- StorageClass
    - PersistentVolumeが確保するストレージの種類を定義
    - GCPだと、「標準」「SSD」がある
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  labels:
    kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```
- StatefulSet
    - データストアのように継続的にデータを永続化するようなアプリに向いているリソース
    - pod-0, pod-1, pod-2のように連番のIDでPodを作成する
        - この識別子は、Podが再生成されても保たれる
        - スケーリングもこの連番が続くように行われる
    - PodのIDが固定化されることで、ストレージとの紐付けが顔脳となる
    - ステートフルなReplicaSetという位置付け


