# コンテナの運用

## ロギングの運用

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


































