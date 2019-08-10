---
layout: post
title:  "Nextcloud+Dockerを構築した"
categories: server
---

### 1. Nextcloud

Nextcloud は NAS とその Web UI を実現する Web アプリケーションである。
自前でアプリケーションをホストしてプライベートなオンラインストレージを作成できる。

Nextcloud GmbH のコミュニティにより開発されている、フリーかつオープンソースのプロジェクトである。

- [Nextcloud](https://nextcloud.com/)
- [GitHub \- nextcloud/server: ☁️ Nextcloud server, a safe home for all your data](https://github.com/nextcloud/server)
- [Nextcloud \- Wikipedia](https://ja.wikipedia.org/wiki/Nextcloud)

実装は PHP と JavaScript で記述されている。

Nextcloud はほとんど同じようなプロジェクトである ownCloud のフォークである。
ownCloud の開発者である Frank Karlitcheck により、2016 年に ownCloud から脱退する形で分裂した。
ownCloud と比べるとプロプライエタリなサポートが無いなど、ビジネス的な指向が低いようだ。

これまでは ownCloud を利用してきたが、
今回、ownCloud では音楽プレイヤー (Ampache) がいまいち動かなかったのと、
Nextcloud の方がオープンな開発コミュニティが活発そうという直感にのみ従い、
Nextcloud に移行してみた。

### 2. これまでの経緯

2年前より ownCloud を利用し、自宅サーバー上で運用してきた。

本ブログでは初回構築時から何本かの記事を掲載してきた:

- 2017/2/19 初回構築 [Docker\+Nginx\+ownCloud\+SSLで構築した](https://kikei.github.io/server/2017/02/19/owncloud.html) 
- 2017/2/25 Docker Compose へ移行 [Docker ComposeでownCloudを構築した](https://kikei.github.io/server/2017/02/25/docker-owncloud.html)
- 2017/4/10 Docker 向けのストレージ `/var/lib/docker` を専用ファイルシステムへ移動 [Docker\+ownCloudのデータ保存パスを引っ越しした](https://kikei.github.io/server/2017/04/10/docker-dir.html)

振り返ると、初回構築時には謎カスタマイズを施すなど色々と苦労したみたいだ。
今回はカスタマイズを全くせずにすんなり構築できたので満足している。

### 3. SSL 証明書の用意

SSL 証明書は Let's Encrypt で取得済みであるので、この記事では特に触れない。
証明書の実体はホスト側の `/etc/letsencrypt` に保存してあるとする。

```console
# ls /etc/letsencrypt 
accounts  archive  csr	keys  live  renewal  renewal-hooks
```

Let's Encrypt からの SSL 証明書取得方法は初回構築時の記事に書いたようだ:

[Docker\+Nginx\+ownCloud\+SSLで構築した](https://kikei.github.io/server/2017/02/19/owncloud.html) 


### 4. Docker コンテナの起動

Nextcloud と Nginx の Docker コンテナを起動し連携させる。

DB のコンテナは作成しない。Nextcloud のデータ層には SQLite、MariaDB、PostgreSQL のいずれかを使うことができるようだが、今回は Nextcloud イメージに含まれている SQLite を利用した。何も考えずにデフォルトのイメージから起動すると SQLite になる。

Nginx は Web のフロントサーバーとして利用する。

ファイル構成:

```
.
├── data
├── docker-compose.yml
└── nginx-proxy
     ├── Dockerfile
     └── default.conf
```

#### Nginx 設定

Nginx はベースのものから変更が必要なので、自前のイメージを作った。

ファイル構成:

```console
# ls nginx-proxy
default.conf Dockerfile
```

Nginx設定は以下のようにした。

nginx-proxy/default.conf:

```conf
server {
    listen       80;
    server_name  storage.example.net;
    return 301 https://storage.example.net$request_uri;
}
server {
    listen       443 ssl;
    server_name  storage.example.net;

    ssl_certificate     /etc/letsencrypt/live/storage.example.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/storage.example.net/privkey.pem;

    location / {
        proxy_set_header Host $host;
        proxy_pass http://app/;
    }

    error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

コメントすべき点は以下:

- ホスト名は storage.example.net とした。
- HTTP でリクエストされたら HTTPS に自動的に転送するようにした。
- Nextcloud に Proxy するとき、Host ヘッダを付与する必要がある。そうでないと Nextcloud のドメインチェックでエラーを返されてしまう。

Dockerfile は、イメージ作成時に上記 Nginx 設定ファイルをコピーするようにしただけ。

nginx-proxy/Dockerfile:

```
FROM nginx
COPY default.conf /etc/nginx/conf.d/default.conf
```

#### Docker Compose 設定

Docker Compose の設定を作成した。

docker-compose.yml:

```yaml
version: '3'
services:
  app:
    container_name: nextcloud-app
    image: nextcloud
    volumes:
      - ./data:/var/www/html
      - /mnt/storage:/mnt/storage
    restart: always
  nginx:
    container_name: nextcloud-nginx-proxy
    build: ./nginx-proxy
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt:ro
    links:
      - app:app
    depends_on:
      - app
    restart: always
```

Nextcloud (app) については以下の通り:

- Nextcloud のデータ保存先は、シンプルにただのディレクトリとした。
- ホスト側のストレージ `/mnt/storage` を Volume に指定して Nextcloud からアクセス可能にした。

Nginx (nginx) については以下の通り:

- Nginx はホストの Port 80, 443 を LISTEN する。
- SSL 証明書はディレクトリ `/etc/letsencrypt` を Volume に指定して Nginx からアクセス可能にする。

#### Docker コンテナの起動

Nginx イメージをビルドして、コンテナを起動する:

```console
# docker-compose build
# docker-compose up -d
# docker-compose ps 
        Name                       Command               State                    Ports                 
--------------------------------------------------------------------------------------------------------
nextcloud-app           /entrypoint.sh apache2-for ...   Up      0.0.0.0:8080->80/tcp                   
nextcloud-nginx-proxy   nginx -g daemon off;             Up      0.0.0.0:443->443/tcp,                  
                                                                 0.0.0.0:80->80/tcp
```

### 5. Nextcloud 設定

Nextcloud を Web ブラウザで開き、UI で設定を行う。
ここから先は人それぞれなので、以下は私の設定メモである。

#### 5.1. 初回セットアップ

Nextcloud は登録済のドメインの URL 以外で接続した場合にエラーとする機能がある。

初回のセットアップ時に、Nextcloud を利用可能な FQDN が自動的に登録されるため、
最終的に使う予定のURLで接続した方がよい。

つまり https://nas.example.net 等を Web ブラウザで開く、ということ。
例えば http://192.168.0.10 等 LAN 内の URL で接続してセットアップすると、それで登録されてしまい、あとで https://nas.example.net でリクエストした場合にエラーになってしまう。

#### 5.2. 管理用アカウントの作成

適当に管理用アカウントのユーザー名、パスワードを入力する。

すると勝手にセットアップが行われる。
私の環境では数分程度かかった。

#### 5.3. ユーザー、グループの作成

1. 右上の歯車マークからユーザーを選択。
2. 左側メニューの「グループを追加する」、「新しいユーザー」から適当に設定を行う。

#### 5.4. 外部ストレージの接続

docker-compose.yml で `/mnt/storage` としてマウントしたホスト側ストレージを、
Nextcloud の UI から表示できるようにする。

設定は外部ストレージとして行う:

1. 右上のマークからアプリを選択。
2. 右ペインのリストから「External storage support」を見つけ有効化する。
3. 右上のマークから設定を選択。
4. 左側メニューから「外部ストレージ」を選択。
5. 右ペインで適当に設定する。

設定内容は以下の通り:

- フォルダー名: ファイル UI 上で表示されるフォルダー名。
- 外部ストレージ: 適当に選択。ホストにマウントされたディレクトリなら「ローカル」でよい。
- 設定: 外部ストレージが「ローカル」ならディレクトリパス。例: 「`/mnt/storage`」
- 利用可能: 利用できるユーザーまたはグループを適切に設定。

#### 5.5. Music Player

Nextcloud に移行した最大の目的はこれである。
ファイルから音声ファイルを選んだら曲が再生されるようにしたい。

Nextcloud では Music アプリを有効化することにより、これを実現できる:

1. 右上のマークからアプリを選択。
2. 左側メニューから「マルチメディア」を選択。
3. 右ペインで「Music」を「ダウンロードして有効にする」。

これをやっておき UI 上で楽曲ファイルを選択すると、
勝手に Web プレイヤーが起動し再生が始まる。

### 6. おまけ LAN 内 DNS for スマートフォン

せっかく Nextcloud を構築したのだが、このままでは LAN の Wi-Fi にぶらさがったスマートフォンから接続できない。
デフォルトでは、ドメインをグローバルな DNS を使い解決するため、グローバル IP アドレスを引いてしまい、うまく LAN 内のサーバーに辿り着くことができないためだ。

いくつか方法があると思うが、LAN 向けの DNS サーバーも立ててしまうことにした。

dnsmasq と Docker を使うと DNS サーバーを簡単に構築することができる。
docker-compose.yml 例:

```yaml
ersion: '3'
services:
  dnsmasq:
    image: andyshinn/dnsmasq
    container_name: dnsmasq
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    extra_hosts:
      - "storage.example.net:192.168.0.2"
    cap_add:
      - NET_ADMIN
    restart: always
```

storage.example.net は Nextcloud に接続するためのドメインを書く。
それに続く 192.168.0.2 は LAN 内の Nextcloud サーバーの IP アドレスを記述する。

```console
# docker-compose up -d
```

あとはスマートフォンの Wi-Fi 設定でプライマリ DNS の IP アドレスを DNS サーバーの LAN 内 IP アドレスに設定すればよい。

[dockerでdnsサーバを立てる \- Qiita](https://qiita.com/tac0x2a/items/c6306b95dd27703bc975)

### 7. まとめ

Docker を使い、Nextcloud によるオンラインストレージを再構築した。

以前 ownCloud を構築したときと比べると全てが順調に進み、快適だった。

### 8. 参考

Nextcloud:

- [Nextcloud](https://nextcloud.com/)
- [GitHub \- nextcloud/server: ☁️ Nextcloud server, a safe home for all your data](https://github.com/nextcloud/server)
- [GitHub \- nextcloud/docker: ⛴ Docker image of Nextcloud](https://github.com/nextcloud/docker)
- [Nextcloud \- Wikipedia](https://ja.wikipedia.org/wiki/Nextcloud)

Nginx:

- [Nginx redirect Http to Https \- what's wrong here? \- Stack Overflow](https://stackoverflow.com/questions/15947646/nginx-redirect-http-to-https-whats-wrong-here)

Dnsmasq:

- [dockerでdnsサーバを立てる \- Qiita](https://qiita.com/tac0x2a/items/c6306b95dd27703bc975)

