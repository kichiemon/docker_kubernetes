# Kubernetes入門

Kubernetesとは、Google社主導で開発された、コンテナの運用を自動化するためのコンテナオーケストレーションシステム。

- コンテナオーケストレーションを実現・管理するための統合的なシステム
- 操作のためのAPIやCLIツールも提供する
- デプロイなど、運用管理の自動化を実現できる
    - Serverのリソースを考慮したコンテナ配置、スケーリングなどなど
- Dockerコンテナを管理するという意味ではKubernetesはSwarmとほぼ同じ立ち位置
- Kubernetes入門はCompose/Stack/Swarmの機能を統合しつつ、より高度に管理できるもの


## 環境


Kubernetesのインストール

```
$ curl -LO https://storage.googleapis.com/kubernetesrelease/release/v1.10.4/bin/darwin/amd64/kubectl \
  && chmod +x kubectl\
  && mv kubectl /usr/local/bin/
```

Kubernetesのデプロイ
e.g. ダッシュボード https://github.com/kubernetes/dashboard
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
```
$ kubectl proxy
```

## Kubernetes概念
Kubernetesクラスタ内のリソースには↓がある

- Node
    - クラスタが持つ最も大きな概念
    - クラスタ管理下に登録されているDockerホストのこと。コンテナのデプロイのために利用される
    - クラスタには全体を管理するサーバであるMasterが少なくとも一つが配置される
    - GCPではGCE、AWSではEC2がNode
    - `$ kubectl get nodes`
- Namespace
- Pod
- ReplicaSet
- Deployment
- Service
- Ingress
- ConfigMap
- PersistentVolume
- PersistentVolumeClaim
- StorageClass
- StatefulSet
- Job
- CronJob
- Secret
- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding
- ServiceAccount


