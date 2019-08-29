# デプロイ戦略

Deploymentでは新しいPodに置き換えるための戦略を `.spec.strategy.type` で定義する

`.spec.strategy.type` には、 `RollingUpdate` or `Recreate` のどちらかを指定する

デフォルトは `RollingUpdate`

## RollingUpdate

古いバージョンのアプリケーションを実行した状態で新しいバージョンのアプリを起動し、順々にサービスインしていく仕組み。
- 新しいPodが実行状態になってから、既存のPodがTerminateし始めるので、ダウンタイムがなくて済む

#### RollingUpdateの挙動
-  maxUnavailable : 同時に削除できるPodの最大数。デフォルト 25%
-  maxSurge : 新しいPodを作る個数。デフォルト 25%

#### ヘルスチェックの挙動

livenessProbeとreadnessProbeの２つのヘルスチェックがある

- livenessProbe : アプリケーションの死活チェック
    - コンテナ内で煽が依存しているファイルや実行ファイルの存在をチェックできる
```
livenessProbe:
  exec:
   command:
   - cat
   - /live.txt
   initialDelaySeconds: 3
   periodSeconds: 5
```
- readnessProbe : HTTPなどのトラフィックを受けられる状態かどうかをチェックする
    - httpGet
```
readnessProbe:
 httpGet:
  path: /hc
  port: 8080
 timeoutSeconds: 3
 initialDelaySeconds: 15
```

#### BlueGreen Deployment
RollingUpdateの弱点として、新旧のPodが同時に存在する点が挙げられる    
意図せぬ副作用が発生する可能性がある。

BlueGreen Deploymentは、新旧二つを切り替えてデプロイする手法

メリット
- 瞬時に切り替えられる
- 用意にデプロイ前の状態に戻せる

デメリット
- RollingUpdateよりサーバー数が多くなるので抱えるリソースが多苦なる


BlueGreenの方法としては、2つのステップがある
- 新しいDeploymentを用意する
- Serviceがみるselectorのラベルを切り替える

新しいDeploymentを用意して、切り替える
```
$ kubectl apply -f echo-version-blue.yml
$ kubectl apply -f echo-version-green.yml
```

Serviceがみるselectorのラベルを切り替える
```
# selector.color: blue でblueを見る
$ kubectl apply -f echo-version-blue-green-service.yml
service/echo-version created

# selector.color: green でgreenを見る
$ kubectl apply -f echo-version-blue-green-service.yml
service/echo-version created
```

#### 安全にアプリを停止させてからPodを削除する

GracefulShutdown
- アプリが安全に停止してからPod内のコンテナが削除されるようにすること
    - ユーザーのリクエストの処理中
- Podに終了命令が下ると、コンテナ内のプロセスにSIGTERMが送信される
- terminationGracefulPeriodSeconds の値でシャットダウンまでの時間を調整できる
- lifecycle.preStopで、処理をフックできる
```
spec:
 terminationGracefulPeriodSeconds: 60
 containers:
  ....
```

#### デプロイして検証する
- [echo-version.yml](./deploy/echo-version.yml)
- [update-checker.yml](./deploy/update-checker.yml)

```
$ kubectl apply -f .
service/echo-version created
deployment.apps/echo-version created
pod/update-checker created

$ kubectl logs -f update-checker

$ kubectl patch deployment echo-version \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"echo-version","image":"gihyodocker/echo-version:0.2.0"}]}}}}'
deployment.extensions/echo-version patched
```

