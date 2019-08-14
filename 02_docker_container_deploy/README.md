# Memo

## Dockerイメージ
Dockerコンテナを構成するためのファイルシステムや、実行するアプリケーションや設定をまとめたもので、コンテナを作成するための利用されるテンプレートとなるもの

## Dockerコンテナ
Dockerイメージを基に作成され、具現化されたファイルシステムをアプリケーションが実行されている状態

## Dockerfile
イメージを構築するための手順を記述したファイル

* FROM: Dockerイメージのベースとなるイメージを指定
* RUN イメージのビルド時にDockerコンテナ内で実行するコマンド
* COPY Dockerコンテナ内にコピーする
* CMD Dockerコンテナとして実行する時に、コンテナ内で実行するコマンド
* ENV Dockerfileを基に生成したDockerコンテナないで使える環境変数を指定する
* ENTRYPOINT CMDと同じく、コンテナ内で実行するプロセスを指定できる。ENTRYPOINTによって、コンテナが実行するデフォルトのプロセスを指定できる

```
FROM golang:1.10

ENTRYPOINT ["go"]
CMD [""]
```

としておくと、goコマンドを引数で渡さなくてもgoを実行できる

```
$ docker container run ch02/golang:latest version
go version go1.10.3 linux/amd64
```

## Imageの操作

```
docker image build -t イメージ名[:タグ名] Dockerfile配置ディレクトリのパス
```

```
docker image build -t example/echo:latest .
```

* 基本的に、 `-t` で必ずタグ名を指定するようにしよう
* -fオプション -fをつけるとファイル名を指定できる
* --pullオプション docker imageのベースイメージを強制的に再取得する --pull=true

### Imageを探す

```
docker search [options] 検索キーワード
```

```
docker search --limit 5 mysql
```
* 名前空間であるオーナー名がついていないものは公式のイメージ


### Imagのタグ付け

Dockerイメージのバージョン == DockerイメージのID

```
docker image tag 元イメージ名[:タグ] 新イメージ名[:タグ]
```
```
docker image tag example/echo:latest example/echo:0.1.0
```

### Imagの公開

```
docker image push [options] リポジトリ名[:タグ]
```

## コンテナの操作

* 実行中・停止・破棄、という3つの状態のいずれかに分類される
* 実行が完了すると、停止の状態に移行します

### Run Docker Image
* docker container run に引数を与えることで、CMDを上書きできる

```
docker container run -it golang:1.10
```

* 名前付きコンテナ機能がある。コンテナの指定がしやすくなる
* 開発用とで使われるが、本番環境では基本使われない。既存の同名コンテナを削除してから、名前をつけて起動する必要があるため
```
docker container run -t -d --name example-echo example/echo:latest
```

* 頻出オプション
* -i -t --rm -v
    * -iは、コンテナがわの標準入力を繋ぎっぱなしにする
    * -tは擬似端末を有効にする。e.g. ターミナルでの接続を有効にする
    * --rmはコンテナ終了時にコンテナを破棄する
    * -vはホストとコンテナ間でディレクトリを共有する


### Stop Containers

-q はコンテナIDだけを抽出するためのオプション
```
docker container stop $(docker container ls -q)
```

### Remove Containers
```
docker container rm コンテナIDまたはコンテナ名
```

```
docker container ls --filter "status=exited"
```

### コンテナの標準出力取得
```
docker container logs [options] コンテナIDまたはコンテナ名
```


### コンテナでのコマンド実行
```
docker container exec [options] コンテナIDまたはコンテナ名 コンテナ内で実行するコマンド
```

### コンテナにファイルをコピーする/コンテナからファイルをコピーする
```
docker container cp [options] コンテナIDまたはコンテナ名:コンテナ内のコピー元 ホスト先のコピー
```
```
docker container cp [options] ホストのコピー元 コンテナIDまたはコンテナ名:コンテナ内のコピー元
```

### Port Forwarding

```
docker container run -d -p 9000:8080 example/echo:latest
```

## prune 破棄

```
# 実行していないコンテナを一括削除する
doocker container prune [options]
```

```
# 不要なImageをを一括削除する
doocker image prune [options]
```

## Docker Compose

docker-compose.yml に設定をかく

* volumeは共有する
* コンテナ間の通信は、link: 名前で解決する

