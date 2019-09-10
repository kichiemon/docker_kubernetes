# 9章 より軽量なDockerイメージを作る

## なぜ軽量なDockderイメージを作るのか
* サービスインまでの時間が長くなる
* CI時間の増大
* Nodeディスクの消費
* トライアンドエラーのしにくさ
* 生産性の低下


## 9.2 軽量なベースイメージ

#### Scratch
* 空っぽのDockerイメージ    
* FROMでのみ参照可能
* DockerImageの始祖

* Sample : [hello_image/Dockerfile](Dockerfile)
```
$ docker image build -t ch09/hello:latest .
Sending build context to Docker daemon  2.001MB
Step 1/3 : FROM scratch
 --->
Step 2/3 : COPY hello /
 ---> 0ad82361f10a
Step 3/3 : CMD ["/hello"]
 ---> Running in c32591a9c54d
Removing intermediate container c32591a9c54d
 ---> cafeefcfdaa1
Successfully built cafeefcfdaa1
Successfully tagged ch09/hello:latest

$ docker container run -t ch09/hello:latest
hello, scratch!!
```

**Go言語のアプリをビルドすると、実行可能なバイナリのみを生成するんで、サイズ削減可能**

// ネイティブライブラリのリンク
ダイナミックリンクするような場合には注意が必要

* httpsの証明書もないので、COPYする必要がある: [https_web/Dockerfile](Dockerfile)


ただし、Scratchの利用は現実的ではない。
* 1から自分でビルド
* 全部インストールしてこないといけない 
* shがないのでログインできない

