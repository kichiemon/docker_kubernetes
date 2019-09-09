# Kubernetesにおけるログの管理

こちらもDockerとやるべきことは大して変わらない。    
コンテナは標準出力に出すこと以外は意識せず、コンテナ外でろぐをかんりすべき

## DeamonSet
- Kubernetesクラスタで管理されているすべてのNodeにたいして、必ず１つずつ配置されるPodを管理する
- ログコレクタなど、エージェントとして配置したいものに適している
-


## KubernetesでElasticsearchとKibanaを構築→fluentdをDeamonSetで→ログが収集できているか確認

[./logging_on_kubenetes/elasticsearch.yaml](./logging_on_kubenetes/elasticsearch.yaml)    
`$ kubectl apply -f elasticsearch.yaml`


[./logging_on_kubenetes/kibana.yaml](./logging_on_kubenetes/kibana.yaml)    
`$ kubectl apply -f kibana.yaml`


[./logging_on_kubenetes/fluentd-deamonset.yaml](./logging_on_kubenetes/fluentd-deamonset.yaml)    
```
$ kubectl apply -f fluentd-deamonset.yaml
$ kubectl -n kube-system get pod -l app=fluentd-logging
NAME            READY   STATUS    RESTARTS   AGE
fluentd-dn6j2   1/1     Running   0          41s
```


[./logging_on_kubenetes/echo.yaml](./logging_on_kubenetes/echo.yaml)    

```
$ kubectl apply -f echo.yaml
service/echo created
deployment.apps/echo created

$ curl http://localhost:30080
Hello Docker!!⏎
```

## その他ツール
* Stackdriver
    * 標準出力されるJSONログを集計
* stern
    * ラベル指定で簡単にログを閲覧可能
    * `kubectl logs -f [PodのID]` → `stern -l app=echo` だけでよくなる
    * ラベルベースで実行できるのでPodが入れ替わってもそのままの状態でよい
    * `$ stern -l app=todoweb --context gke_gihyo_kube_xxxxxx_asia-xxxx`

## まとめ
* ロギングは全て標準出力
* Nginxなどのミドルウェアも全て標準出力できるように
* JOSN形式にして検索集計しやすく
* fluentdをDeamonSetで各Podに
* リソースにはラベルをつけて検索性を確保


