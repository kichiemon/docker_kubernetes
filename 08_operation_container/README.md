# コンテナの運用

## 8.1ロギングの運用

- 標準出力に出して、それをFluentdなどのログコレクタで収集することが多い
- Dockerは、標準出力をログとして出力される機構が備えられている
    - /var/lig/docker/containers/コンテナIDのコンテナID-json.log で確認できる
    - json-fileという `logging driver` によるもの
        - json-file
        - journald
        - awslogs
        - gcplogs
        - fluentd
        - など

非コンテナの様にログをファイルに出力することの弊害
- コンテナが停止してディスクから完全に削除されるとログが残らない

ログローテート
- ロギングに挙動を制御するためのオプションがある
    - `--log-opt`
- `docker container run -it --rm -p 8080:8080 --log-opt max-size=1m --log-opt max-file=5 gihyodocker/echo:latest`
    - max-file : 値を超えた分を消す
    - max-size : ローテートのサイズ

FluentdとElasticsearchによるログ収集・検索機構
- dockerホストが増えると、ログファイルの管理が大変
- JSONログファイルを別の場所に転送して集約管理する仕組みが欲しい

// ElasticsearchとKibanaを構築
- [docker-compose.yml](logging/docker-compose.yml)
- http://localhost:5601 を開くと、Kibana
- fluentdのimageをビルドする
    - [fluentd-elasticsearch/Dockerfile](./logging/fluentd-elasticsearch/Dockerfile)
    - `docker image build -t ch08/fluentd-elasticsearch:latest .`
- echoコンテナのlogging部分で driverを指定
```
logging:
 driver: "fluentd"
 options:
  fluentd-address: "localhost:24224"
  tag: "docker.{{.Name}}"
```
- `$ docker compose up -d`
- `$ curl http://localhost:8080`

#### 複数のDockerホストのログをどう集めるか？ fluentd logging driverの運用イメージ
強くお薦めされれているもの
- fluentdを各ホストにエージェント的に配置する方式
    - 各ホストに配置する分sん型では、ホスト障害で影響を受けるのは限定的で、影響範囲を抑えることができる


## 8.2 Dockerホストやデーモンの運用

`--live-restore` をつけると、Dockerデーモンを停止してもコンテナを停止させないことが可能
* コンテナを停止させずに、Dockerのアップデートが可能となる


e.g. Ubuntuの場合    
/lib/systemd/system/docker.serviceにオプションを追加する    
```
[Service]
ExecStart=/usr/bin/dockerd -live-restore
```

Dockerを再起動
```
$ systemctl deamon-reload
$ service docker restart
```

### Dockerdのチューニング

Linux系だと、`/etc/docker/deamon.json` に記述して再起動して設定

**max-concurrent-downloads**    
`docker image pull` の並列度設定。デフォルトは3


**max-concurrent-uploads**    
`docker image push` の並列度設定。デフォルトは3

**registry-mirror**    
Docker Hubのミラーレジストリを設定するオプション    
Docker Hubのミラーレジストリをローカルに立てて、そこからpullできるようにする。DockerHubのレイテンシに左右されなくなる



## 8.3 障害対策

リスクと対策

### Image起因の障害
* 意図しないImageができた
* latestのイメージは使わない
* Imageのテストを行う
    * `container-structure-test` がよく利用される
```
curl -LO https://storage.googleapis.com/container-structure-test/latest/container-structure-test-darwin-amd64 && chmod +x container-structure-test-darwin-amd64 && sudo mv container-structure-test-darwin-amd64 /usr/local/bin/container-structure-test
```
    * sample : [./image_test/test-tododb.yaml](image_test/test-tododb.yaml)

### ディスク容量の枯渇
ホストのディスク容量には注意が必要
* Dockerを運用してもディスクを浪費しない構造にしておくことが大事
    * cronなどで `dockder system prune -a` を実行してImageを削除
* ログをためないこと

### Kubernetesにおける障害ポイント

### Nodeダウン
サーバー障害によるNodeが停止する可能性を念頭におく
* Auto-Healing
    * Nodeが落ちると、ReplicaSetの指定数だけ別のNodeに再配置される
* つまり、ReplicaSetを管理するDeploymentやStatefulSet、DeamonSetを利用してPodを作成することが大切

### Podの配置戦略
Podが複数のNodeに配置されていればサービスダウンせずにPodの再配置が可能
* ReplicaSetの落とし穴として
    * 同じNodeに配置されると、1つのNodeのダウンで死んでしまう
    * Kubenetesはリソースの空き気味なNodeに配置するので、同じNodeに配置されることもありうる
* これを解決するのが、Pod AntiAffinity
    * 「CのPodがあるNodeにDのPodを配置したくない」
    * `spec.affinity.podAntiAffinity` を設定する
    * labelSelectorは「app=echoであるPod」の意
        * 「app=echoであるPodがデプロイされているNodeには、app=echoであるPodをデプロイしない」
```.yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - echo
        topologyKey: "kubernetes.io/hostname"
```

### CPUを多く利用するPodをNode Affinityで隔離する
* 過度にCPUリソースを要するPodは専用Nodeに配置
* ほかのPodに影響を与えないように気をつける
* Node Affinity
    * `spec.affinity.nodeAffinity` を設定する
```.yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelectorTerms:
        - matchExpressions:
          - key: instancegroup
            operator: In
            values:
            - "batch"
```

### Horizontal Pod Autoscalerを利用
* Pod数を自動的に増減させるためのKubernetesリソース
  * HPAは、DeploymentやReplicaSetに対してPodのオートスケールを実行する条件を設定するリソース
* `kubectl get hpa`
* 次のCluster Autoscalerと組み合わせて利用すること
* e.g. CPU 40%超えたらスケール
```.yaml
kind: HorizontalPodAutoscaler
...
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 40
```

### Cluster Autoscalerを利用したNodeのオートスケール
* Nodeの数を自動調整
* GKEの場合
    * create cluster時に、
    * `--enable-autoscaling` `--min-nodes` `--max-nodes`
* GKE以外では、Helmを利用
    * `$ helm install --namespace kube-system --name cluster-autoscaler stable/cluster-autoscaler`

### Helmのリリース履歴を制限
* ConfigMap：Nameとバージョンのセット
    * Helmでのインストール・アップデートを繰り返すと、ConfigMapが溜まっていく
    * `$ helm init --history-max 20` で最大数を設定して回避
```
$ kubectl -n kube-system get configmap
NAME                                 DATA   AGE
coredns                              1      37h
elasticsearch-config                 3      8h
extension-apiserver-authentication   6      37h
kube-proxy                           2      37h
kubeadm-config                       2      37h
kubelet-config-1.14                  1      37h
```






