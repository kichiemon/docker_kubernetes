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











