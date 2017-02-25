---
layout: post
title:  "Docker ComposeでownCloudを構築した"
categories: server
---

本稿は [Docker+Nginx+ownCloud+SSLで構築した](https://kikei.github.io/server/2017/02/19/owncloud.html)の続きである。
Docker Compose を使って、ポータブルな ownCloud 環境を構築する。

過去記事では生の `docker` コマンドを使って Nginx+ownCloud+PostgreSQL による
NASシステムを構築したが、このときの方法では管理コストが大きく、
Docker を利用するメリットがあまり感じられなかった。

特にいけていなかったのは次の通り;

- Nginx, ownCloud, PostgreSQL を Docker で順番に起動する必要がある。
- ownCloud を再起動すると、Nginx も再起動しなければならない。
- ボリュームとしてマウントしているホスト側ディレクトリに触ると容易に壊れる。

で、いろいろ遊んでいたら、案の定 ownCloud と PostgreSQL の整合性が崩れて
あっさりシステム利用不可能になった。

そこで、本稿では復旧ついでに Docker Compose を使ってみた。

ちなみに、前回は Nginx+ownCloud+PostgreSQL で構築したが、
今回は Nginx+ownCloud+MariaDB で構築した。

ownCloud の公式を見たら、MySQL が推奨だといっていたのでそうしてみただけであり、
本題には特に関係ない。

### ディレクトリ構造

簡単。`owncloud/` はどこでもOK。

```
owncloud/
  + docker-compose.yml
  + nginx-proxy/
    + Dockerfile
	+ default.conf
```

### Nginx のカスタム Dockerfile を作成する

カスタム版 `default.conf` 入りの Nginx の材料を書く。

```
% mkdir nginx-proxy
% cd nginx-proxy
% vi default.conf
% vi Dockerfile
```

`default.conf` の中身はこんな感じ。

- 443番ポートで HTTPS を LISTEN する。
- 80番ポートはガン無視。
- Let's Encrypt な証明書を参照させる。

```
% cat default.conf
server {
    listen       443;
    server_name  <ホスト名>;

    ssl on;

    ssl_certificate     /etc/letsencrypt/live/<ホスト名>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<ホスト名>/privkey.pem;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://owncloud/;
    }
略
}
```

`Dockerfile` の中身はこんな感じ。

```
% cat Dockerfile
FROM nginx
COPY default.conf /etc/nginx/conf.d/default.conf
```

イメージの作成とかはあとで `docker-compose` コマンドをつかってやるので、
ここではやらなくていい。自信がなければ、 `docker` コマンドでやってみてもいい。

```
% docker build -t owncloud_nginx .
% docker run --rm -ti -p 80:80 -p 443:443 owncloud_nginx
```

的な。

### Docker Compose の設定ファイルを作成する

`docker-compose.yml` を書くだけ。

```
% cd ..
% vi docker-compose.yml
```

こんな感じ。

```
% cat docker-compose.yml
version: '3'

services:
  mariadb:
    container_name: owncloud-mariadb
    image: mariadb:10.1.21
    volumes:
      - mariadb-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: <MYSQLのパスワード>
    restart: always
  owncloud:
    container_name: owncloud
    image: owncloud:latest
    links:
      - mariadb:mariadb
    volumes:
      - owncloud-data:/var/www/html
      - /house:/house
    depends_on:
      - mariadb
    restart: always
  nginx:
    container_name: owncloud-nginx-proxy
    build: ./nginx-proxy
    links:
      - owncloud:owncloud
    ports:
      - "443:443"
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - owncloud
    restart: always

volumes:
  mariadb-data:
    driver: local
  owncloud-data:
    driver: local
```

#### データボリュームコンテナを使う。

前回、ボリュームにはホストOSのディレクトリを指定したが、
今度はデータボリュームコンテナを使う。

データボリュームコンテナはデータ格納専用の Docker コンテナである。
解説はどこかよそのもっと知っている方に譲るとして、要するに、
アプリケーションのデータを、アプリケーションのコンテナとは別の独立した
コンテナに持たせる。そうすることで、

- アプリケーション本体を綺麗でポータブルな状態に保ち、
- アプリケーションのデータは永続化しつつ、
- データのポータビリティも確保する

というテクニックである。

データボリュームコンテナは、`docker-compose.yml` のトップレベルに、
`volumes` を書くことで勝手に作成される。

今回は、MariaDB 向けに `mariadb-data`、
ownCloud 向けに `owncloud-data` を個別に用意した。

#### カスタム Dockerfile を使う

Dockerfile を指定するときは、
`build: <Dockerfileのあるディレクトリへのパス>` と書く。

これだけ。

あとは、前回 `docker` に渡していたものを、適当に直して移植した。

### Nginx のイメージ作成

`docker-compose.yml` に `build` があると、自動的にイメージが作成される。

```
% docker-compose build
```

`default.conf` や `Dockerfile` を編集し、再ビルドしたい場合にも、
ただ `build` するだけでよい。

### MariaDB と ownCloud のイメージ取得

`docker-compose.yml` に `image` があると、自動的にイメージを取得する。

```
% docker-compose pull
```

### コンテナ起動

これもコマンド一発でやってくれる。

```
% docker-compose up -d
```

ここまでで、MariaDB、ownCloud、Nginx が立ち上がっているはず。

`https://<サーバ名>/` にアクセスしてみて、
問題なく ownCloud の設定画面が表示されればめでたく完了である。

ownCloud の設定とかが知りたい方は前回の記事の該当部分を見たらいいと思う。

[Docker+Nginx+ownCloud+SSLで構築した](https://kikei.github.io/server/2017/02/19/owncloud.html)

今回の場合、初期設定は次のようにする。

- user: 任意の管理ユーザ名
- password: 任意の管理パスワード

- login: root
- password: `docker-compose.yml` の `MYSQL_ROOT_PASSWORD` に書いたパスワード
- db: owncloud
- hostname: mariadb

### コンテナ終了

```
% docker-compose stop
```

### イメージの削除

```
% docker-compose rm
```

ボリュームコンテナごと削除したい場合には、

```
% docker-compose rm -v
```

### 感想

Docker Compose 便利。

### 参考

- [Compose ファイル・リファレンス - Docker-docs-ja 1.13.RC ドキュメント](http://docs.docker.jp/compose/compose-file.html)
- [Compose file version 3 reference / Volume configuration reference - Docker](https://docs.docker.com/compose/compose-file/#/volume-configuration-reference)
- [コンテナでデータを管理する / データ・ボリューム・コンテナの作成とマウント - Docker-docs-ja 1.9.0b ドキュメント](http://docs.docker.jp/engine/userguide/dockervolumes.html#id8)
- [Docker データボリュームコンテナをつくる](http://unskilled.site/docker-データボリュームコンテナをつくる/)
- [docker-composeを使うと複数コンテナの管理が便利に](http://qiita.com/y_hokkey/items/d51e69c6ff4015e85fce)
