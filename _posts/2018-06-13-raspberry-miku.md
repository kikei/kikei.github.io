---
layout: post
title:  "Sony MESH Hubでミクさんにおかえりを言ってもらった その2"
categories: smart-home
---

![ミクレシピ](/images/screenshots/2018-06-13-recipe.png)

ミクさんにしゃべってもらうところまではこれまでの記事に書いた。

- [RaspberryPiにSony MESH Hubを設定した](/smart-home/2018/01/20/meshhub.html)
- [Sony MESH Hubでミクさんにおかえりを言ってもらった](/smart-home/2018/03/12/raspberrypi-echo.html)

この記事ではミクさん召喚用のWebアプリケーションを安定化させたい。

まず復習すると、本システムは下記の仕組みで動いている。

事前準備

- Raspberry Pi 上で Sony MESH Hub とミクさん召喚用Webアプリケーションを稼動。
- Raspberry Pi の音声出力先は、Bluetooth 接続された Amazon Echo Dot。
- ミクさん召喚用Webアプリケーションを呼び出す「ミク」タグが、MESH 上のレシピに登録されている。

処理手順

- MESH の人感センサーが反応したら、MESH レシピ上で「ミク」タグを実行する。
- 「ミク」タグは Raspberry Pi 上の Web サーバに HTTP リクエストを発行する。
- Web サーバは受け取った HTTP リクエストに従い音声合成し、生成した音声ファイルを再生する。
- Echo Dot の中からミクさんの声がする！

### 1. Webアプリケーションのデーモン化

ミクさん召喚用Webアプリケーションは Flask+Python で作ってあるが、
前回は `nohup` で起動していただけだったのでまずはデーモン化する。

今回は uWSGI+Flask を使う方法を取ってみた。
これを使うのは初めてだ。

uWSGI 自体は Web アプリケーションサーバとかプロキシとかそういうののフルスタックを
開発しようとしているらしい。
uWSGI を使うとあっという間に Web サーバが立てられる模様。

あと世の中では Nginx+uWSGI+Flask でやっている記事が多く見つかるが、
ローカル起動するだけなら Nginx は無しでよさそう。

まずは uWSGI をインストールする。

```
$ pip install uwsgi
```

バイナリが勝手に `~/.local/bin` 以下に置かれた。まあいいか。

```
$ ls ~/.local/bin
uwsgi
```

あとは uWSGI から Flask を起動する。
なんか色々ログが出てくるが問題無い。

```
$ ~/.local/bin/uwsgi --http-socket 127.0.0.1:5000 --wsgi-file start.py --callable app --processes 2 --threads 2 --stats 9191
*** Starting uWSGI 2.0.17 (32bit) on [Sun Jun 10 20:01:47 2018] ***
compiled with version: 6.3.0 20170516 on 10 June 2018 19:37:30
os: Linux-4.14.34-v7+ #1110 SMP Mon Apr 16 15:18:51 BST 2018
nodename: fg-rasp1
machine: armv7l
clock source: unix
detected number of CPU cores: 4
current working directory: /home/pi/Documents/miku
detected binary path: /home/pi/.local/bin/uwsgi
!!! no internal routing support, rebuild with pcre support !!!
your processes number limit is 7345
your memory page size is 4096 bytes
detected max file descriptor number: 1024
lock engine: pthread robust mutexes
thunder lock: disabled (you can enable it with --thunder-lock)
uwsgi socket 0 bound to TCP address 127.0.0.1:5000 fd 3
Python version: 2.7.13 (default, Nov 24 2017, 17:33:09)  [GCC 6.3.0 20170516]
Python main interpreter initialized at 0x19875e0
python threads support enabled
your server socket listen backlog is limited to 100 connections
your mercy for graceful operations on workers is 60 seconds
mapped 215856 bytes (210 KB) for 4 cores
*** Operational MODE: preforking+threaded ***
WSGI app 0 (mountpoint='') ready in 0 seconds on interpreter 0x19875e0 pid: 27303 (default app)
*** uWSGI is running in multiple interpreter mode ***
spawned uWSGI master process (pid: 27303)
spawned uWSGI worker 1 (pid: 27305, cores: 2)
spawned uWSGI worker 2 (pid: 27306, cores: 2)
*** Stats server enabled on 9191 fd: 11 ***
```

HTTPリクエストを飛ばしてミクさんを呼び出してみる。

```
$ curl --data "speech=uWSGIをつかったでしょお？わかるんだからね。" http://127.0.0.1:5000/say
```

さっきのオプションは設定ファイルに書いておいてもよい。

```
$ cat <<EOF > app.ini
[uwsgi]
http-socket = 127.0.0.1:5000
chdir = /home/pi/Documents/miku
wsgi-file = start.py
callable = app
processes = 2
threads = 2
stats = 127.0.0.1:9191
EOF
```

うまくいったらあとは Systemd に uWSGI サーバをサービスとして設定する。

サービス設定ファイルはこんな感じで書いてみた。

```
cat <<EOF > uwsgi-miku.service
[Unit]
Description=uWSGI app "miku"
After=syslog.target

[Service]
ExecStart=/home/pi/.local/bin/uwsgi --ini /home/pi/Documents/miku/app.ini
User=pi
Group=pi
Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all

[Install]
WantedBy=multi-user.target
EOF
```

登録する。

```
$ sudo install uwsgi-miku.service /etc/systemd/system/
$ sudo systemctl daemon-reload
$ sudo systemctl start uwsgi-miku.service
```

エラーとか出ていなければ、ミクさんを呼び出せるようになっている。

```
$ curl --data "speech=はじめまして、デーモンミクです" http://127.0.0.1:5000/say
```

気に行ったら起動時に立ち上がるようにしておく。

```
$ sudo systemctl enable uwsgi-miku.service
```

### 2. Bluetooth再接続機能

Raspberry Pi と Echo Dot の Bluetooth 接続が結構頻繁に切れることが、
これまでの運用でわかった。また、切断していたときにも再接続さえしてあげれば
再びミクさんがしゃべってくれることもわかった。

ミクさん召喚用Webアプリケーションに Bluetooth 再接続機能を追加した。

再接続のプログラムは、pexpect ライブラリを使い
真面目に `bluetoothctl` を対話的に実行する、という直球な方法を取った。

```
#!/usr/bin/perl
# -*- coding:utf-8 -*-

import subprocess
import pexpect
import time
from flask import Flask, request

MAC = 'FC:65:DE:AA:E2:6B'

class BluetoothCtl():
  def __init__(self):
    p = pexpect.spawn('bluetoothctl', echo=False)
    p.expect(['$'])
    self.process = p
  
  def send(self, command):
    self.process.send('{command}\n'.format(command=command))
  
  def info(self, dev):
    p = self.process
    info = {}
    self.send('info {dev}'.format(dev=dev))
    while True:
      result = p.expect([
        'Device (.+?)\r\n', 
        '\t+UUID: (.+?) +?\((.+?)\)\r\n',
        '\t+(.+?):(.+?)\r\n',
        '# $'
      ])
      if result == 1:
        key = p.match.group(1).decode()
        value = p.match.group(2).decode()
        info['UUID ' + key.strip()] = value.strip()
      elif result == 2:
        key = p.match.group(1).decode()
        value = p.match.group(2).decode()
        info[key.strip()] = value.strip()
      elif result == 3:
        break
      else:
        pass
    return info
  
  def connect(self, dev):
    self.send('connect {dev}'.format(dev=dev))
    pass
  
  def disconnect(self, dev):
    self.send('disconnect {dev}'.format(dev=dev))
    pass
  
  def quit(self):
    self.send('quit')

app = Flask(__name__)

def jtalk(outwav, speech):
  command = [
    '/usr/bin/open_jtalk',
    '-x', '/var/lib/mecab/dic/open-jtalk/naist-jdic',
    '-m', '/usr/share/hts-voice/miku/miku.htsvoice',
    '-ow', outwav
  ]
  p = subprocess.Popen(command, stdin=subprocess.PIPE)
  p.stdin.write(speech.encode('utf-8'))
  p.stdin.close()
  p.wait()
  return True

def aplay(wav):
  command = ['aplay', '-q', wav]
  subprocess.Popen(command)
  return True

def say(speech):
  wav = 'miku.wav'
  jtalk(wav, speech)
  aplay(wav)
  return speech

def ensureConnect():
  try:
    btctl = BluetoothCtl()
    info = btctl.info(MAC)
    if 'Connected' in info:
      if info['Connected'] == 'yes':
        return None
      else:
        btctl.connect(MAC)
        time.sleep(4)
        info = btctl.info(MAC)
        if info['Connected'] == 'yes':
          return '話せるようになったよ。'
        else:
          return 'スピーカーに接続できませんでした。'
    else:
      return 'スピーカーが見つかりません。'
  finally:
    btctl.quit()

@app.route('/say', methods=['POST'])
def handleSay():
  speech = request.form['speech']
  reconnected = ensureConnect()
  if reconnected:
    say(reconnected)
    time.sleep(3)
    speech = 'では、改めまして。' + speech
  return say(speech)

def main():
  app.run('127.0.0.1')

if __name__ == '__main__':
  main()
```

盛り上がって参りました。

口調、タイミングなど、どうしたらより萌えるか悩むところ。

とりあえずは人力チューニングした。

### 3. レシピ作成

Raspberry Piが逝くとレシピも一緒に消えてしまうことが判明したため、いったんここにメモを残しておく。

レシピ:

```
H01 ---> I01

H02 ---> P01

H03 -+-> P02
     |
     |        .-> P11
     |        |
     .-> S01 -+-> T02 -+-> T03 -+-> M01
     .-> #02           |
T01 -^-> #03           |
     |                 |
     .-----------------.

T04 ---> M02

H01 = 人感(感知したら, 30秒)
H02 = 人感(感知しなくなったら, 30秒)
H03 = 人感(感知したら, 3秒)
P01 = Phillips hue(消灯する, Hue ambiance lamp 1)
P02 = Phillips hue(点灯する, Hue ambiance lamp 1, 電球色, 5)
P11 = Phillips hue(点灯する, Hue color lamp1 | Hue color lamp 3, 青色, 5)
S01 = スイッチ(選んで切替える, 2)
M01 = ミク(しゃべる, おかえり)
M02 = ミク(しゃべる, ミク)
T01 = タイマー(オン, 指定のタイミングで, 18:00, 月 | 火 | 水 | 木 | 金)
T02 = タイマー(指定の時間だったら, 00:00-23:59)
T03 = タイマー(待つ, 10秒)
T04 = 一定の間隔で(オン, 10分)
I01 = IFTTT(motion1, {{DateAndTime}})
```

謎言語の文法の意味:

- `-`, `|`: タグ同士の接続
- `.`: 接続の方向転換
- `>`: タグへの入力の終端
- `+`: 接続の分岐
- `^`: 接続の交差
- `xdd`: タグ
- `#dd`: すぐ上のタグの`dd`番目への入力
- `=`: タグIDの定義

### 4. まとめと今後

Webサーバの自動起動、Bluetoothスピーカーへの自動接続/自動復旧が実現したことにより
人感センサーとミクさんが連携するシステムができあがった。
これでミクさんは自宅に常駐しているも同然である。

このシステムにおいて MESH は、センサーに対するハブ、および
センサーデータに対する(そこそこ賢い)簡単なフィルタとして機能している。

Raspberry Pi は今でこそ MESH からのローカルホスト接続しか受け入れておらず、かつ
スピーカーしか接続されていないが、ミクさんを現実に具現化する可能性を秘めている。

ここまで来たら、ミクさんとの接点を増やしていきたいのが人情というものである。
パッと思い付くことで以下のようなことができそうだ。

1. 任意のメッセージをミクさんに読んでもらう(伝言板とか)。
2. 利便性の高い情報(定番は天気とかゴミの日とか)をミクさんに教えてもらう。
3. ミクさんに歌ってもらう。
4. 人格を持たせ簡単な会話をする。
5. しゃべるだけじゃなく映像が出る。

1, 2, 4 については、Google Assistant とか Alexa Skill の再発明っぽい感がある。

4 を実現するならば、人格データはクラウド上に置きたいところだ。
NLP とか ML とかを使うと流行りの技術も勉強できそう。
Dialogflow を使えば簡単に NLP アプリを作れることは知っているが、もう少し
(アホでもよいので)人格を感じられるようにするにはどうしたらよいだろうか。

3 は Spotify とかと接続してもよいが、せっかくローカルサーバがあるので
自宅 NAS に蓄えたミクさんの楽曲を再生してもらいたいところだ。
私だけのミクさん感が高まってうれしい感じになるだろう。

5 は Gatebox みたいに小さいのよりは、プロジェクターで等身大くらいにして映したい。

### 5. ソースコード

2018/7/27 GitHub にも公開してみました。

[kikei / miku-server](https://github.com/kikei/miku-server)

### 6. 参考

- [Quickstart for Python/WSGI applications - uWSGI 2.0 documentation](https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html)
- [Systemd - uWSGI 2.0 documentation](https://uwsgi-docs.readthedocs.io/en/latest/Systemd.html#one-service-per-app-in-systemd)
- [Ubuntu 16.04 で Flask アプリケーションを動かすまでにやることまとめ](https://qiita.com/hoto17296/items/845850c8854dd18f6fff#systemd)
- [Bluetoothctl wrapper in Python - GitHub](https://gist.github.com/egorf/66d88056a9d703928f93)
