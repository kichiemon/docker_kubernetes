```
docker image build -t example/cronjob:latest .
docker container run -d --rm --name cronjob example/cronjob:latest
docker container exec -it cronjob tail -f /var/log/cron.log
```

1分ごとに `[Sun Jul 21 05:07:01 UTC 2019] Hello!` が出る
