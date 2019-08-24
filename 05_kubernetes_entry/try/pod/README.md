```

$ kubectl apply -f simple-pod.yml --validate=false

$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
simple-echo   2/2     Running   0          38s
```

```
$ kubectl apply -f simple-replicaset.yml --validate=false
```

```
$ kubectl apply -f simple-deployment.yml --validate=false
deployment.apps/echo configured
```

```
$ kubectl get pod,replicaset,deployment --selector app=echo
NAME                        READY   STATUS    RESTARTS   AGE
pod/echo-5d69cfbc84-hq7bb   2/2     Running   0          23s
pod/echo-5d69cfbc84-kz2q9   2/2     Running   0          23s
pod/echo-5d69cfbc84-mtlhj   2/2     Running   0          23s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.extensions/echo-5d69cfbc84   3         3         3       23s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/echo   3/3     3            3           23s
```

```
$ kubectl rollout history deployment echo
deployment.extensions/echo
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --validate=false --record=true
```

// replicaの数を上げても新規で追加されるだけで、ReplicaSetは新規で作成されない

```
$ kubectl apply -f simple-deployment.yml --record --validate=false
deployment.apps/echo configured

$ kubectl get pod
NAME                    READY   STATUS              RESTARTS   AGE
echo-5d69cfbc84-hq7bb   2/2     Running             0          4m50s
echo-5d69cfbc84-kz2q9   2/2     Running             0          4m50s
echo-5d69cfbc84-mtlhj   2/2     Running             0          4m50s
echo-5d69cfbc84-ww8qn   0/2     ContainerCreating   0          6s
```


// コンテナ定義を更新すると、新しくらしく作成される
```
$ kubectl apply -f simple-deployment.yml --record --validate=false
deployment.apps/echo configured

$ kubectl rollout history deployment echo
deployment.extensions/echo
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --record=true --validate=false
2         kubectl apply --filename=simple-deployment.yml --record=true --validate=false
```

// revision指定すると、過去の設定も観れる
```
$ kubectl rollout history deployment echo --revision=1
deployment.extensions/echo with revision #1
Pod Template:
  Labels:	app=echo
	pod-template-hash=5d69cfbc84
  Annotations:	kubernetes.io/change-cause: kubectl apply --filename=simple-deployment.yml --record=true --validate=false
  Containers:
   nginx:
    Image:	gihyodocker/nginx-proxy:latest
    Port:	<none>
    Host Port:	<none>
    Environment:
      BACKEND_HOST:	localhost:8080
    Mounts:	<none>
   echo:
    Image:	gihyodocker/echo:latest
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>

$ kubectl rollout history deployment echo --revision=2
deployment.extensions/echo with revision #2
Pod Template:
  Labels:	app=echo
	pod-template-hash=5c445df584
  Annotations:	kubernetes.io/change-cause: kubectl apply --filename=simple-deployment.yml --record=true --validate=false
  Containers:
   nginx:
    Image:	gihyodocker/nginx-proxy:latest
    Port:	<none>
    Host Port:	<none>
    Environment:
      BACKEND_HOST:	localhost:8080
    Mounts:	<none>
   echo:
    Image:	gihyodocker/echo:patched
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```



// 複数のReplicaSetを起動
```
$ kubectl apply -f simple-replicaset-with-label.yml --validate=false
replicaset.apps/echo-spring configured
replicaset.apps/echo-summer created
```

// Service起動
```
$ kubectl apply -f simple-service.yml
service/echo created
```

// Service接続用のコンテナをデプロイ
```
$ kubectl run -i --rm --tty debug --image=gihyodocker/fundamental:0.1.0 --restart=Never -- bash -il
```


// NodePortで外からcurlできる
```
$ kubectl apply -f simple-replicaset-with-label.yml,simple-service-node-port.yml --validate=false

$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
echo         NodePort    10.102.161.214   <none>        80:30790/TCP   41s
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        10d

$ curl http://localhost:30790
Hello Docker!!⏎
```


// nginx ingressのデプロイ
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml # https://kubernetes.github.io/ingress-nginx/deploy/
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
```

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml
service/ingress-nginx created
```

```
$ kubectl -n ingress-nginx get service,pod
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.106.222.203   localhost     80:32407/TCP,443:30554/TCP   20s

NAME                                          READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-8544567f-bw877   0/1     Running   0          33s
```

```
$ kubectl get ingress
NAME   HOSTS              ADDRESS   PORTS   AGE
echo   ch05.gihyo.local             80      8s
```

```
$ curl http://localhost -H 'Host: ch05.gihyo.local'
```

```
$ curl http://localhost -H 'Host: ch05.gihyo.local' \
  -H 'User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X)a
  AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1'
```
