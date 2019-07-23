Docker in Dockerを利用して複数のホストを立てる

作成するコンテナ
* registry x 1
* manager x 1
* worker x 3

```
docker-compose up -d
docker container ls
```

managerにswarn initこまんどを投げる
```
docker container exec -it manager docker swarm init
# tokenが出力されるので、使う
# workerをswarmにjoinさせる
docker container exec -it worker03 docker swarm join \
--token SWMTKN-1-3nobpjltinukendqq6xc9dn41zvzv09cw801a6xmho5083pnj4-9wr1goxfuynst0kar639kdrmy manager:2377
docker container exec -it worker02 docker swarm join \
--token SWMTKN-1-3nobpjltinukendqq6xc9dn41zvzv09cw801a6xmho5083pnj4-9wr1goxfuynst0kar639kdrmy manager:2377
docker container exec -it worker01 docker swarm join \
--token SWMTKN-1-3nobpjltinukendqq6xc9dn41zvzv09cw801a6xmho5083pnj4-9wr1goxfuynst0kar639kdrmy manager:2377
docker container exec -it worker01 docker swarm join \
--token SWMTKN-1-3nobpjltinukendqq6xc9dn41zvzv09cw801a6xmho5083pnj4-9wr1goxfuynst0kar639kdrmy manger:2377

docker container exec -it manager docker node ls

# registoryにimageをpushする
docker image tag example/echo:latest localhost:5000/example/echo:latest
docker image push localhost:5000/example/echo:latest

# serviceを作る
docker container exec -it manager \
  docker service create --replicas 1 --publish 8000:8080 --name echo registry:5000/example/echo:latest
docker container exec -it manager \
  docker service ls
docker container exec -it manager \
  docker service scale echo=6
docker container exec -it manager \
  docker service ps echo
docker container exec -it manager \
  docker service rm echo
```

Stack操作
```
docker container exec -it manager docker network create --driver=overlay --attachable ch03

docker container exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo

docker container exec -it manager docker stack services echo

docker container exec -it manager docker stack deploy -c /stack/visualizer.yml visualizer

docker container exec -it manager docker stack deploy -c /stack/ch03-ingress.yml ingress

docker container exec -it manager docker service ls
```
