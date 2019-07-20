# 手順


```
docker image pull gihyodocker/echo:latest
docker container run -t -p 9000:8000 gihyodocker/echo:latest
curl http://localhost:9000
docker container stop (docker container ls -q)
```


