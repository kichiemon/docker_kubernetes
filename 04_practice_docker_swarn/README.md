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

## まとめ

コンテナオーケストレーションを活用したアプリケーション開発では、 MySQL・API・WebのServiceをそれぞれ連携させることで1つのアプリケーションを構築する。   
このスタイルではServiceに対するリクエストを所属するコンテナに分散させられる   
→耐久性の高いシステムを作れる
→また、適切なノードへのコンテナデプロイや、スケーリングをコンテナオーケストレーションの仕組みに任せられる
