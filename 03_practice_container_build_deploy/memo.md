# Memo

実用的なコンテナの構築とデプロイについて

## コンテナの粒度

* 1つのコンテナにどれだけの役割を担わせるべきか？
* 細かく役割を分割しすぎて複雑になっていないか？

1コンテナ＝1プロセス？
* 定期実行アプリケーションだと、cron, APIを設ける必要があり複雑すぎる
→ 1コンテナ=1コンテナ＝1プロセスの方式より、無理せずひとつのコンテナで複数のプロセスを実行したほうがシンプルに完結するユースケースも多い

デプロイのしやすさを得るためにDockerを利用しているのに、子プロセスまで意識して曲げてしまうのでは本末転倒

*じゃあどういう考え方で分割すればいいの？*

Docker公式ドキュメント「Best practice for writing Dockerfiles」では、
「Each container should have only one concern」（コンテナはひとつの関心ごとだけに集中すべきだ）
と記載されている。

*ひとつの関心ごと*
→ひとつのコンテナはあるひとつの役割や問題領域のみにフォーカスされるべき


*「それぞれのコンテナが担うべき役割を適切に見定め、かつそれがレプリカとして複製された場合でも副作用なくスタックとして正しく動作できる状態になるか？」*


## コンテナのポータビリティ

完璧ではなくいくつかの例外が存在する
* 実行できるホストは、一定のCPUアーキテクチャやOSの前提の上に成立している
* アプリケーションが利用しているライブラリがダイナミックリンクの場合、そこがずれていると実行できない
    * →　極力スタティックリンクしてビルドすることを第一に考えよう
    * 避けられなかったり、サイズをお菊したくなければ、Imageビルドようのコンテナを設ける手がある


## Dockerフレンドリーなアプリケーション

必要な要素をコンテナの挙動の制御、設定という視点で考えると、、、

【環境変数の利用】Dockerコンテナで実行されるアプリケーションの挙動を制御する方法
* 実行時引数
    * 引数からのマッピング処理が複雑化
    * CMDやENTRYPOINTが煩雑で管理が難しくなる
* 設定ファイル
    * 設定ファイルごとにImageを作る → 環境ごとにImageを変える → ポータビリティが狭まる
    * 実行したい環境を増やすたびにImageをビルドし直す必要がある
    * これを採用するなら実行時に渡すようにしよう
* アプリケーションの挙動を環境変数で制御
    * かく環境で利用する環境変数を定義したファイルを集約したりリポジトリを作って管理するのが良い
    * Domposeであれば、docker-compose.ymlのenv属性に列挙
    * KubernetesやECSでも同様の仕組みがあるので、利用できる
    * Key-Valueのデータしか持てないデメリット
* 設定ファイルに環境変数を埋めこむ
    * フレームワークによって利用できたりできなかったりする

*アプリケーションの挙動を環境変数で制御する* に最も注目している


## 永続化データをどう扱うか
ステートフルなアプリケーションの場合、コンテナを破棄するとデータが失われる

→Data Volumeを利用
DataVolumeコンテナという永続化用のコンテナを起動する手もある

### Data Volume
Dockerコンテナないのディレクトリをディスクに永続化するための仕組み


Data Volumeの作成 -vを利用する
```
# docker container run [options] -v ホスト側ディレクトリ:コンテナ側ディレクトリ リポジトリ名[:タグ] [コマンド] [コマンド引数]
```
e.g.
```
docker container run -v ${PWD}:/workspace gihyodocker/imagemagic:latest convert -size 100x100 xc:#000000 /workspace/gihyo.jpg
```
→カレントディレクトリにコンテナで生成されたgihyo.jpgが共有されているはず

デメリット
* ホスト側の操作によってアプリケーションに副作用が起きる可能性
などのポータビリティの面で課題


### Data Volumeコンテナ
コンテナ間でディレクトリを共有する
e.g. MySQLの保持

* DockerfileでVOLUMEを設定
* 実行する側で--volumes-from msyql-data のように指定する

```
$ docker container run -d --rm --name mysql \
  -e "MYSQL_ALLOW_EMPTY_PASSWORD=yes" \
  -e "MYSQL_DATABASE=volume_test" \
  -e "MYSQL_USER=example" \
  -e "MYSQL_PASSWORD=example" \
  --volumes-from mysql-data \
  mysql:5.7
```




## コンテナ配置戦略
コンテナオーケストレーション

* Docker Swarm
    * 複数のDockerホストを束ねてクラスタ化するためのツール

コンテナでのオーケストレーションに関わる名称
|名称|役割|コマンド|
|---|---|---|
|Compose|複数のコンテナを使うDockerアプリケーションの管理。主にシングルホスト|docker-compose|
|Swarm|クラスタの構築や管理を担う。主にマルチホスト|docker swarm|
|Service|Swarnの前提、クラスタ内のServiceを管理する|docker service|
|Stack|Swarmmの前提、複数のServiceをまとめたアプリケーション全体の管理|docker stack|

オーケストレーションを試すには、
Dockerには、Docker in Dockerという仕組みがあるのでそれを利用すると楽。dind

### Service

Serviceにレプリカ数の制御を指示すると、自動でコンテナを複製し、複数のノードにまたがって適切に配置してくれる

Stackが扱うアプリケーションの粒度はComposeと同様
StackによってデプロイされるService群は、overlayネットワークに所属


### Stack

* deploy
* ls
* ps
* rm
* services


