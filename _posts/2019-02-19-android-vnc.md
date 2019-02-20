---
layout: post
title:  "余ったスマートフォンをPCのサブディスプレイに"
categories: linux
---

最近、Androidスマートフォンが一台空いた。
今回はこれを有効活用する方法として、PCのサブディスプレイにしてみた。

頑張ってsystemdとかから使えるようにしたので本記事でやり方を紹介する。

### 1. スマートフォンをサブディスプレイにする

空いたのは3年ほど利用してきたGalaxy S7である。
酸性温泉に持っていきすぎたためか、Micro USB端子が錆びて充電がうまくいかなくなってしまった。

手持ちではAnkerのモバイルバッテリーに付属のケーブルを使った場合のみなぜか充電できる。フィット感の高いケーブル以外では充電できないようだ。

こいつが完全にダメになる前に、ということで新しい端末を購入したわけだが、
一方でスマートフォンがまだ使える状態で余ってしまった。
持ち歩かなければそれほど支障も無いので、なんとか活用できたら…。

どうやらスマートフォンを PC のサブディスプレイとして使う方法があるらしい。
私の PC 環境はデュアルディスプレイ環境だが、実はあと一枚くらい欲しいと思っていた。

現在、一枚は Emacs と Web ブラウザ、もう一枚はターミナルと音楽プレイヤー(動画を再生する Web ブラウザのことも多い) に使われている。前者はまあいいとして、後者、ターミナルと音楽プレイヤーが重なるのは少し不便に感じていた。

ターミナルか音楽プレイアーのいずれかがスマートフォンのディスプレイに収まってくれたらきっと便利だろう。

スマートフォンをサブディスプレイにするには、VNC を使うのが定番のようだ。
PC 側に VNC サーバーを立て、スマートフォンを VNC クライアントとして繋ぐ。

今回はその方法を試してみた。

### 2. 環境

#### PC (VNCサーバー) 側

OS:

```sh
% uname -a                   
Linux localhost.localdomain 4.19.15-300.fc29.x86_64 #1 SMP Mon Jan 14 16:32:35 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

ネットワーク:

```sh
% ifconfig enp3s0 | head -2
enp3s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.8  netmask 255.255.255.0  broadcast 192.168.0.255
```

#### Androidスマートフォン (VNCクライアント) 側

- Samsung Galaxy S7
- Android 8.0.0
- Wi-Fi接続 192.168.0.9
- 解像度 2560x1440

### 3. ファイヤーウォールの設定

ではさっそく VNC サーバーの構築を…といきたいところであるが、
先にファイヤーウォールで VNC 接続のポートを開放しておいてあげる。

まずは有効になっているゾーン名を調べる:

```sh
% sudo firewall-cmd --get-active-zone
public
  interfaces: enp3s0 wlp4s0
```

調べてわかった `public` ゾーンへ VNC を登録する。
サービス名は `vnc-server` として始めから定義されており、
ポート 5900-5903 が設定されている。以下はそれを定義しているファイルの中身:

```sh
% cat /lib/firewalld/services/vnc-server.xml 
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>Virtual Network Computing Server (VNC)</short>
  <description>A VNC server provides an external accessible X session. Enable this option if you plan to provide a VNC server with direct access. The access will be possible for displays :0 to :3. If you plan to provide access with SSH, do not open this option and use the via option of the VNC viewer.</description>
  <port protocol="tcp" port="5900-5903"/>
</service>
```

`vnc-server` サービスを常に利用可能にする:

```sh
% sudo firewall-cmd --permanent --add-service=vnc-server
success
```

`firewalld` に設定を反映する:

```sh
% sudo firewall-cmd --reload
```

### 4. VNC サーバーの設定

VNC サーバーは [TigerVNC](https://tigervnc.org/) を使った。
近道っぽかったからで、特に選んだ理由は無い。

インストールは DNF から出来るので便利だ。

```sh
% sudo dnf install tigervnc-server
```

とりあえず起動するところまでやってみる。

実は以前に一回だけ VNC を起動したことがあったようで、過去の設定が残っていた。
だからディレクトリごと先に消去してクリーンにした。

```sh
% rm -r ~/.vnc
```

VNC サーバーの初回起動時にはパスワードを設定させられる。

```sh
% vncserver :1 -geometry 1280x720 -alwaysshared

You will require a password to access your desktops.

Password:
Verify:
Would you like to enter a view-only password (y/n)? n

New 'localhost.localdomain:1 (fujii)' desktop is localhost.localdomain:1

Creating default startup script /home/fujii/.vnc/xstartup
Creating default config /home/fujii/.vnc/config
Starting applications specified in /home/fujii/.vnc/xstartup
Log file is /home/fujii/.vnc/localhost.localdomain:1.log
```

- `:1` が仮想ディスプレイの番号である。
- `-geometry` は仮想ディスプレイの解像度である。あまり大きくすると文字が小さくて見えなくなる。色々やってみたなかでは、スマートフォンの解像度の半分である1280x720が見易かった。
- `-alwaysshared` は複数クライアントで接続を共有できるかを設定するらしい。今回はサーバー、クライアントが常に一対一なので有っても無くてもいいオプションか。

ここまででもう、Android の VNC クライアントアプリから接続できるようになっている。

### 5. VNC クライアントからの接続

VNC Viewer というアプリを使った。これも定番アプリのようだ。

IPアドレスの後ろにディスプレイ番号を付与して接続する。

```
192.168.0.1:1
```

新規でディスプレイが立ち上がり、それが Android 端末上に表示された。
ターミナルだけがあるディスプレイだ。

なんか裏にダイアログが出ていて、Xfce4 の起動に失敗した的な気持ちを感じた。
なんで Xfce なのかは追いきれていない。Fedora インストール時に Xfce を使っていた名残が残っているんだと思う。

だが、今は Gnome を使っている。できたら新規のディスプレイでも Gnome が立ち上がって欲しい。その挙動は `~/.vnc/xstartup` で変更することができる。

xstartup を編集し、Gnome が起動するようにした:

```sh
% cat ~/.vnc/xstartup 
#!/bin/sh

unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
# exec /etc/X11/xinit/xinitrc

/usr/bin/vncconfig -iconic &
exec /usr/bin/gnome-session

/usr/bin/xsetroot -solid grey
/usr/bin/xterm -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &
```

### 6. キーボードの共有

めでたくスマートフォン上に PC の画面が表示されたわけだが、わりと使いにくい。

本体のディスプレイ (:0) とは別の仮想ディスプレイ (:1) であるから、隔離された世界になるからだ。具体的にはどう不便かというと、ウィンドウやマウスカーソルの移動、キーボード入力ができない。

これでは使いにくいが、せめてキーボード入力だけならば簡単に有効化できる。

[x2vnc](http://fredrik.hubbe.net/x2vnc.html) を使う。

Fedora の DNF で検索したところではヒットしなかったので、ソースからビルドした。

```sh
% tar xf x2vnc-1.7.2.tar.gz
% cd x2vnc-1.7.2
% ./configure --prefix=~/.local
% make
% make install
% which x2vnc
~/.local/bin/x2vnc
```

x2vnc を起動する:

```sh
% x2vnc -shared -south localhost:1
/home/fujii/.local/bin/x2vnc: VNC server supports protocol version 3.8 (viewer 3.3)
/home/fujii/.local/bin/x2vnc: VNC authentication succeeded
/home/fujii/.local/bin/x2vnc: Desktop name "localhost.localdomain:1 (fujii)"
/home/fujii/.local/bin/x2vnc: Connected to VNC server, using protocol version 3.3
/home/fujii/.local/bin/x2vnc: VNC server default format:
screen[0] pos=1853
screen[1] pos=1087
Xinerama detected, x2vnc will use screen 1.
/home/fujii/.local/bin/x2vnc: pointer multiplier: 0.968128
```

オプションについては、

- `-shared` キーボードとかを共有する。
- `-south` 仮想ディスプレイ (:1) を :0 の下に配置。他に `-north`、`-east`、`-west` が使える。
- `localhost:1` 共有にする対象のディスプレイ、

といった感じ。

起動がうまくいくと、PC 側とスマートフォン側のディスプレイの間でマウスカーソルが行ったり来たりできるようになった。
また、スマートフォン側にフォーカスが当った状態で PC のキーボードを操作すると、スマートフォン側のターミナルに文字が入力された。随分と便利になった。

なおマウスカーソルはなぜか左右に散ってしまい、ほとんど使うことはできなかった。
別にマウスカーソルは使えなくてもいいと思ったので、この問題は放置することにした。

スマートフォン上ではタッチによるカーソル操作が効くので、PC 側のマウスは特に必要無い。

### 7. サービス化

VNC が結構使えそうなことがわかったので、
サービスにして PC 起動時に自動的に VNC サーバーが立ち上がるようにする。

今回は vncserver だけでなく、x2vnc もサービスにした。
前者はともかく、後者をサービスにした記事は見当らない。

#### vncserver.service

まず vncserver.service は以下のようにした。

TigerVNC をインストールした時点でサービスのファイルが生成されたようだが、
そのままではうまく動かなかったためあちこち少しずつ編集した。

```sh
% cat /etc/systemd/system/vncserver@.service
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple
ExecStartPre=/sbin/runuser -l fujii -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/sbin/runuser -l fujii -c '/usr/bin/vncserver %i -geometry 1280x720 -alwaysshared -fg'
PIDFile=/home/fujii/.vnc/%H%i.pid
ExecStop=/sbin/runuser -l fujii -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'

[Install]
WantedBy=multi-user.target
```

わかっていると思うが、`fujii` は私のユーザー名なので適切に読み替えて欲しい。

仮想ディスプレイ (:1) を起動する場合には以下のように実行する。

```sh
# systemctl daemon-reload
# systemctl start vncserver@:1.service
```

#### x2vnc.service

次に x2vnc は以下のようにした。

```sh
[Unit]
Description=Helper service for sharing keyboard between VNC remote desktop
After=vncserver@.service

[Service]
Type=simple
ExecStart=/sbin/runuser -l fujii -c 'export DISPLAY=:0 && x2vnc -shared -passwdfile $HOME/.vnc/passwd -south %H%i'
ExecStop=killall x2vnc

[Install]
WantedBy=multi-user.target
```

`ExecStop` が `killall` になっているのは体力が尽きた。これでも動くからいいよね。

完成品を見ると大したこと無いが、selinux の制限に引っ掛かったり、XAUTHORITY 環境変数の引き継ぎができなくてうまくいかなかったりと、かなり苦労している。

なお、わかっていると思うが、`fujii` は私のユーザー名なので適切に読み替えて欲しい。

以下のように利用する。

```sh
# systemctl daemon-reload
# systemctl start x2vnc@:1.service
```

上記の2つがうまくいっていれば、仮想ディスプレイとキーボードを共有できているはず。

動作に満足いったら、サービスを自動起動にする。
 
```shared
# systemctl enable vncserver@:1.service
# systemctl enable x2vnc@:1.service
```

### 8. まとめ

スマホにターミナルが出ていると楽しい。
音楽を聴きたいときはターミナルから起動してやればいい。
このときにキーボード共有の力は大きい。

これまでは `cmus` を使って音楽を再生していたが、`clementine` をインストールしてみた。スマートフォン上ではタッチによる操作が可能なので GUI アプリは有効。

スマホスタンドを買っていい感じにしたい。

### 9. 参考情報

- [TigerVNC](https://tigervnc.org/)
- [TigerVNC \- ArchWiki](https://wiki.archlinux.org/index.php/TigerVNC)
- [x2vnc](http://fredrik.hubbe.net/x2vnc.html)
- [systemd\.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#)
- [How to install and Configure VNC \(TigerVNC\) server in CentOS / RHEL 7 – The Geek Diary](https://www.thegeekdiary.com/how-to-install-and-configure-vnc-tigervnc-server-in-centos-rhel-7/)
