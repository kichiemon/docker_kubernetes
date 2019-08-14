# Swarnによる実践的なアプリケーション構築


## Webアプリケーションの構築
- MySQL
- API
- Web
- Nginx

## 配置戦略
Swarnクラスタ上でそれぞれのServiceを展開する

- MySQL
    - mysql_master
    - mysql_slave
- Application
    - app_nginx
    - app_api
- Frontend
    - frontend_nginx
    - frontend_web

```
$ git clone https://github.com/gihyodocker/tododb
```
