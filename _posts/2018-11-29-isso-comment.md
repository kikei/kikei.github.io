---
layout: post
title:  "GitHub Pages+JekyllにIssoコメントフォームを追加した"
categories: blog
---

本ブログは GitHub Pages+Jekyll で運用している。
今回、Isso を使ったコメントフォームを追加した。

本ブログは技術的な話題の多いブログであるが、おそらく記述の間違いとかがあったりするはずだ。その指摘を受け付ける窓口が無いのはよくないと以前から思っていた。コメントフォームがあれば、親切な人がいてそこで正誤を教えてくれることもあるだろう。

実は、前の記事で Disqus を導入したのだが、気に食わなかったので一晩で消してしまった。

- [GitHub Pages\+JekyllにDisqusコメントフォームを追加した](https://kikei.github.io/blog/2018/11/26/disqus-comment.htm)

### 1. Disqus のネガキャンコーナー

最も気に入らなかった点は、匿名での投稿が難しいことだ。

Disqus のコメントフォーム基本的に匿名で投稿できない。
コメントする人は、SNS でログインして Disqus アカウントに紐付けるか、
Disqus アカウントを作って Disqus にログインするかを選ぶことになる。

一応ゲスト投稿機能を有効化することはできるが、非常に小さい「ゲスト投稿する」ボタンを
クリックしたあと、「Disqus 規約」と「Disqus に IP アドレス等の情報を提供すること」に
同意しなければならない。

その他、

- メールアドレスを入力しないとコメントできなかったり
- コメントをいちいち許可 (moderate) しないと表示されないし、それを無効にできないとか
- 許可リストへのコメント反映がやけに遅い、
- コメントリストのロードが妙に遅い、
- ブログ来訪者をトラッキングします感つよすぎてキモい

などなどいろいろあるが、匿名で気軽にコメントできないのがとにかく致命的なので止めた。

イメージしていたのは、もっと適当にコメントを残してさっと立ち去れるようなやつだった。
Yahoo! ブログみたいな感じでもいいし、昔の掲示板みたいなやつでもいいし。

代替になるものを探していたら、結構同じような意見の人達がいるみたいだった。

- [Replacing Disqus with GitHub Comments \| Hacker News](https://news.ycombinator.com/item?id=14170041)
- [How we added comments to our Jekyll site \| Savas Labs](https://savaslabs.com/2016/04/20/squabble-comments.html)
- [What's wrong with Disqus? - Isso](https://posativ.org/isso/docs/#what-s-wrong-with-disqus)
- [Dmitri Shuralyov \- Blog](https://dmitri.shuralyov.com/blog/23)

> It seems Valley companies really love tracking.

的な。個人的な用途をおいてもなんかキモい。

### 2. Isso を選んだ

では簡単に Disqus を置き換えられるかというと、Jekyll では実はやや面倒くさい。
Jekyll における記事は静的ページだからだ。旧来の CGI のようにはいかない。
結局、静的ページに JavaScript を埋め込んで、一部だけ動的に外部からデータを
取得してくるんでしょ、という感じなのだが、そう都合よくコメント部分だけ
取得させてくれるサービスは多くはないだろう。やっぱり Disqus くらい本気でトラッキング
しないと食事に困ってしまうのだ。

でもコメントは欲しい。じゃあ仕方無い自前で作るかー、
みたいなところで [Isso – a commenting server similar to Disqus](https://posativ.org/isso/) が見つかった。

自分でコメント用の Web サーバーを立てて、ブログから JavaScript で連携する方式だ。
せっかくブログ側を GitHub に出したのに、また自前サーバーに返ってくるのかー、とか
思ったけどよく考えたら、GitHub Pages を使うことにした理由は Git で
記事を管理できるからであって、別にサーバー管理しなくていいからとかじゃなかった。
じゃあいいのか、自前サーバーが壊れたらコメントが止まるけど、ブログ的には記事が
ちゃんと見られるなら何の問題も無い。

というわけで今回、Isso を使ってコメントサーバーを立てることにした。

Isso は GitHub 上で開発されているプロジェクトだ。
サーバー側は Python、クライアント側は JavaScript で実装されている。

気に入った点はだいたい以下である:

- 匿名での投稿が可能 (必須)
- メール通知の機能がある (初期装備であると嬉しい)
- 返信機能がある (あった方がよい)
- 管理 UI がある (あった方がよい)
- コメント許可機能 (moderate) がある (実際に使うかはわからない)
- オープンソースである (まあ重要)
- プロジェクトが現在も動いている (まあまあ重要)、
- 軽量っぽい

というわけで、この記事では Isso を利用して GitHub Pages 上でコメント機能を実現する方法を書く。

### 3. システム概要

最近は、こういうのは AWS で立てちゃうよ、なんて言う人も少くないと思うが、
せっかくサーバーのリソースがまだ余っているので自分で管理するサーバーに構築した。

私のサーバーは CentOS で、Apache が起動している。
Apache の VirtualHost 機能を利用し `isso.xaxxi.net` の URL でサーバーを動かすことにした。

Isso は WSGI として動作させることができる。そこで Apache の mod_wsgi を利用する。

Isso サーバーは Web アプリケーションとしてコメントデータの登録、取得を行う機能と、
JavaScript、CSS、SVG アイコンを静的にホストする機能を持つ。

GitHub Pages+Jekyll のブログ側では、静的 HTML ページの中で Isso サーバーから
JavaScript をダウンロードするようにし、そのスクリプトの処理で Isso サーバーの
API を実行してコメントの取得、表示をしたり、誰かが書き込んでくれたコメントを登録したりする。

Isso はデータをサーバー上の SQLite データベースに記録する。

- [4. Isso の構築](#4-isso-の構築)
- [5. Apache の設定](#5-apache-の設定)
- [6. Jekyll の設定](#6-jekyll-の設定)

Isso のドキュメントは割と散らかっており、結構苦労した。
というか不明点を解決できなかったので、ソースコードを読みながら自己解決した。

### 4. Isso の構築

#### 4.1. ユーザー作成

Isso 専用のユーザーを作り、そのユーザー権限でサーバーを動かすことにした。
まあ普通の方法にやるだけ。

```shell
$ sudo su - 
# groupadd isso
# useradd isso -g isso -m 
# usermod -aG linux isso
# passwd isso
Changing password for user isso.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
# chmod 0755 /home/isso
```

グループとかは各自適当にやればよい。

Isso 自体はここで特別な設定を必要としない。

```shell
# su - isso
```

ディレクトリ構造は以下のようにした。ほぼ好みである。

```
.
├── logs
│   ├── access.log              Apache のアクセスログ
│   └── error.log               Apache のエラーログ
└── web
     ├── app.wsgi               Apache から起動する WSGI ファイル
     ├── isso.cfg               Isso のユーザー設定ファイル
     ├── logs
     │   └── isso.log           Isso のログファイル
     ├── data
     │   └── comments.db        Isso の SQLite データベース
     └── isso
          ├── bin               VirtualEnv が実行ファイルを生成する場所
          ├── lib               Pip でパッケージをインストールする場所
          ├── node_modules      Node JS モジュールをインストールする場所
          ├── setup.py          Python サーバーをビルドするのに使う
          ├── Makefile          JavaScript をビルドするのに使う
          ├── share
          │   └── isso.conf     Isso のデフォルト設定
          ├── isso
          │   ├── __init__.py   Isso の Python 実装はここにある
          │   ├── css           CSS を配信する
          │   ├── js            配信する JavaScript はここに生成される
		  :   :
```

というわけで次の手順に向けディレクトリを作っておく。

```shell
$ mkdir logs
$ chmod 0755 logs
$ mkdir web
$ cd web
$ mkdir -m 0777 data
$ chmod 0777 data
$ mkdir -m 0666 logs
```

#### 4.2. ソースコードの取得

まずは Isso のプログラムを取得する。

取得方法には大きく2通りの方法があり、`pip` でインストールするか、
ソースコードを GitHub からクローンしてきてビルドするかである。

今回、初めから改造する気マンマンだったため、迷わずソースコード取得を選んだ。

```shell
$ git clone https://github.com/posativ/isso.git
$ cd isso
```

#### 4.3. サーバーのビルド

今回、公式も推奨しているので VirtualEnv を使った。

ちゃんと isso ユーザー権限でインストールした `virtualenv` コマンドを使わないと
はまるので注意。4時間くらいはまった。

そこさえ注意すれば、正しくコマンドを実行していけばあっさり進むはず。

```shell
$ python3 install --user virtualenv
$ ~/.local/bin/virtualenv .
$ source bin/activate
(isso) $ python3 setup.py --user install
```

ここまで来た時点で `isso` コマンドが生成されている。

```shell
(isso)$ which isso
~/web/isso/bin/isso
```

すぐに起動してみてもいいが、JavaScript が無いとあまり使えない。

#### 4.4. JavaScript のビルド

必要な NodeJS ライブラリをインストール。

Isso 公式ではグローバルにインストールしているが、これもユーザーディレクトリにイントールしておきたい。

```shell
(isso)$ npm install bower requirejs uglify-js jade
```

さっきのインストールで `bower`、`r.js` が実行可能ファイルとして生成されている。

Isso 公式ではパスを通せと言っているが、
面倒くさいのでコマンド実行中だけパスを設定しちゃう。

JavaScript を全体的に生成。

```shell
(isso)$ env PATH=$PATH:/home/isso/web/isso/node_modules/bower/bin make init
```

minify したやつとかを生成。
微妙の PATH の中身を変えているので注意。

```shell
(isso)$ env PATH=$PATH:/home/isso/web/isso/node_modules/requirejs/bin make js
(isso)$ make js
```

もう一回 Python サーバーのインストールをやっておく。

なぜか JavaScript が配信されない問題に当って30分くらい悩んだが、
下記コマンドを実行したら直った。

```shell
(isso)$ python3 setup.py install
```

#### 4.5. 設定ファイルの配置

基本的な `share/isso.conf` に書かれていて、それとの差分だけをユーザー設定の
設定ファイルとして作成する。

パスはあとでちゃんと合っていればいいが、私はなんとなく isso のディレクトリの一段上で
書いた。

```shell
(isso)$ vi ../isso.cfg
```

設定例を示す

```ini
(isso)$ cat ../isso.cfg
[general]
# SQLite3 データベースファイルのパス
dbpath = /home/isso/web/data/comments.db
# Access-Control-Allow-Origin ヘッダに記載するホスト
host =
    https://kikei.github.io

# ログをファイルに書き出す
log-file = /home/isso/web/logs/isso.log
# コメントが有ったらメールで送る
notify = smtp

[smtp]
# コメントが有ったらメールで送る場合の SMTP 設定
username = smtpUserName
password = smtpPassword
host = smtpServer
port = 587
security = starttls
to = report_comment@example.net
from = admin@example.net
timeout = 10

[admin]
# 管理画面の有効化とログインパスワード
enabled = true
password = d6b807704b4544977e967cfd9cce0b62

[moderation]
# 許可されたコメントだけ表示する機能
enabled = true

[guard]
# コメント時に名前を必須にする
require-author = true
# 1つの記事に対し直接ぶら下がる返信数の上限
direct-reply = 10
# 自分のコメントに返信してもよい
reply-to-self = true

[markup]
# Markdown で有効にする機能
# [Misaka -- Misaka 2.1.0 documentation](http://misaka.61924.nl/#extensions)
options = strikethrough, autolink, fenced-code, no-intra-emphasis, highlight, quote, math

[hash]
# メールアドレスから一意な ID を作るためのハッシュ 誰にも真似できないやつを書く
salt = 6aca646e5cdfda3ae51fc964dfe23cdb
```

ここまでやったら Isso を試験起動してみてもよい。

```shell
(isso)$ isso run
```

起動できたら http://localhost:8080/admin とかに接続してみる。
ソーナンスが出たら起動成功している。

あと、Isso とは別に、ブログのダミーとしてテスト用の Web サーバーを立て、
コメント表示と投稿を試してみてもよい。

例えば以下のような index.html をローカルとかでもいいので置き、

```html
<!doctype html>
<html>
<meta charset="utf-8" />
<script src="http://localhost:8080/js/embed.min.js"
     data-isso="http://localhost:8080/" 
     data-isso-css="true" 
     data-isso-lang="en"
     data-isso-reply-to-self="true"
     data-isso-require-author="true"
     data-isso-require-email="false"
     data-isso-max-comments-top="20"
     data-isso-max-comments-nested="5"
     data-isso-reveal-on-click="5"
     data-isso-vote="true">
</script>
<section id="isso-thread" data-isso-id="test" data-title="コメントテスト">
</section>
</html>
```

HTML ファイルがあるディレクトリで、例えば Python3 を使い HTTP サーバーを起動する。

```shell
$ python3 -m http.server 8081
```

で、http://locahost:8081/index.html を Web ブラウザで開く。

コメントフォームが出てきて、かつコメントを送信できたら成功している。

`<section#isso-thread/>` の中に `data-title="..."` を書いておくと、
それがメール通知のタイトルになるのでおすすめ。

ちゃんと `<meta charset="utf-8" />` とか書いてエンコーディングを指定しておかないと、メールの日本語が文字化けするので注意。

ここまでの動作確認で満足できたら、Isso を Apache 経由で使えるようにしていく。

```shell
(isso)$ deactivate
$ su - apache
```

### 5. Apache の設定

#### 5.1. WSGI ファイルの用意

Apache から Python アプリを起動するなら大体 WSGI を使うことが多い。

wsgi ファイルとしてアプリケーションを起動するための Python スクリプトを書き、
そのパスを Apache の設定ファイルに書く、という流れになる。

app.wsgi というファイルを作成し、以下みたいなことをやるスクリプトを書いた。

- VirtualEnv を使っているので site-packages を教えてあげる。
- `source bin/activate` 相当の処理をスクリプト内で実行。
- Isso の Web アプリケーション本体をロードして呼び出し。

こう:

```python
import os
import site
import sys

cwd = os.path.dirname(os.path.abspath(__file__))

user_conf = os.path.join(cwd, 'isso.cfg')
default_conf = os.path.join(cwd, 'isso/share/isso.conf')
activate = os.path.join(cwd, 'isso/bin/activate_this.py')
site_packages = os.path.join(cwd, 'lib/python3.5/site-packages')

site.addsitedir(site_packages)

with open(activate) as f:
    exec(f.read(), dict(__file__=activate))

from isso import make_app, config
application = make_app(config.load(default_conf, user_conf))
```

公式ページにある例だとうまくいかなかったので結構がんばって書いた。

#### 5.2. SSL 証明書の更新

突然だが以下は自分用のメモなのでスルーしてほしい。

SSL 証明書の作成とインストール。

```shell
$ cd ~/lab/ssl
$ sudo service httpd stop
$ vi create.sh
$ sh create.sh
$ sudo service httpd start
```

#### 5.3. Apache 設定の更新

Apache の Virtual Host 設定と WSGI 設定の全容については、今回は触れない。

私の環境はその辺で既に実績があり、今回作業する必要が無かったからだ。
作業していない以上、自信を持って書けることも無い。っていうか面倒くさい。

すなわち、以下の設定を追記し Apache を再起動しただけだ。

```shell
$ cat conf.d/vhost.conf
略
<VirtualHost *:443>
    ServerAdmin fujii@xaxxi.net
    ServerName isso.xaxxi.net:443
    ErrorLog /home/isso/logs/error_log
    CustomLog /home/isso/logs/access_log common
    WSGIScriptAlias / /home/isso/web/app.wsgi
    WSGIDaemonProcess isso processes=1 threads=1 python-path=/home/isso/web/isso
    WSGIProcessGroup isso
    WSGIScriptReloading On
    WSGIPassAuthorization On
    RemoveHandler .py
    AddDefaultCharset UTF-8
    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/privkey.pem
    SSLCertificateChainFile /path/to/chain.pem
</VirtualHost>
略
```

Apache 再起動。

```shell
sudo service httpd restart
```

Web ブラウザで https://isso.xaxxi.net/admin を開いてみたり、
4.5. で作った index.html の中身を修正していろいろ動作確認してみればいいと思う。

これで動いたらほぼほぼ出来ている。
もう難しいことは何一つ無い。ありがてえ。

### 6. Jekyll の設定

あとは Jekyll 側のテンプレートを更新し、さっきまでの index.html で書いていたことを
ブログ上で実行されるようにするだけである。

_layout/post.html に以下を追記。

```html
{% raw %}
{% include comments.html %}
{% endraw %}
```

_include/comments.html を新規に作成する。
中身はこんな感じ。

```html
{% raw %}
{% if page.comments != false %}

<script src="https://isso.xaxxi.net/js/embed.min.js"
     data-isso="https://isso.xaxxi.net/" 
     data-isso-css="true" 
     data-isso-lang="en"
     data-isso-reply-to-self="true"
     data-isso-require-author="true"
     data-isso-require-email="false"
     data-isso-max-comments-top="20"
     data-isso-max-comments-nested="5"
     data-isso-reveal-on-click="5"
     data-isso-vote="true">
</script>
<section id="isso-thread" data-isso-id="{{ page.url | absolute_url }}" data-title="{{ page.title }}">
</section>

{% endif %}
{% endraw %}
```

`<script />` の属性がやや増量されているが、[Client Configuration](https://posativ.org/isso/docs/configuration/client/) を参考にして好きなように書けばよい。

`<section />` の属性には実際の記事の情報が入るようにした。

あーちゃんとエスケープされるようにしないとダメだ。したら直します。

コメントフォームが出たらゴールです。

### 7. まとめ

コメントフォームが出せてよかった。そして無事、脱 Disqus を果たした。

結構苦労したっていうか時間かかった。
でも Virtual Env を使ったアプリケーションをデプロイするのは初めてだったので。

あと Isso に何か手を入れてプロジェクトに貢献しようと思ったけど、特に思い付かなかった。

### 8. 参考

本文中に記載。
