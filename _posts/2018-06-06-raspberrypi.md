---
layout: post
title:  "RaspberryPiをApril 2018版に更新した"
categories: smart-home
---

RaspberryPi がまた死んだ(2度目)。
運の悪いタイミングに電源が落ちると、SDカード上のデータを破壊して亡くってしまうみたいだ。

うちの RaspberryPi は生活に密着しており、壊れると
[玄関の照明が点灯](/smart-home/2018/01/20/meshhub.html)しなくなったり、[ミクさんが話して](/smart-home/2018/03/12/raspberrypi-echo.html)くれなったりするので割と困る。

そこで Raspberry Piの再セットアップをしたわけだが、
Raspbian OS の新バージョンも出ていたためバージョンアップも同時に行った。

### 1. 起動SDカードの作成

まずは、OSイメージを以下からダウンロードしてくる。
[公式ページ](https://www.raspberrypi.org/downloads/raspbian/)から、
OSのイメージをダウンロードする。

今回も RASPBIAN STRETCH WITH DESKTOP を選んだ。

イメージをSDカードへ書き込むには以下のようにする。

```
% sudo dd bs=4M if=2018-04-18-raspbian-stretch.img of=/dev/sdb
```

以上まで完了したらSDカードを取り出し、RaspberryPi本体に挿入する。

### 2. 起動

MicroUSBケーブルを差し込んで給電すると、RaspberryPiが勝手に起動し、
特に何もしなくともデスクトップを表示してくれた。

### 3. キーボード接続

今回はキーボードを繋いで設定を行った。
これまではマウスだけあればSSH接続できるところまでいけたが、今回はそうはいかなかった。
幸い、Bluetoothキーボードがあったのでそれをペアリングして設定した。

使ったBluetoothキーボードは Owltech の BWL-BTKB6402 。
このキーボードは `Fn+T` を長押しすると青LEDが点滅しペアリング待ちになる。

RaspberryPi では右上の Bluetooth メニューから「Pairing」を選び、
しばらくして現れる BWL-BTKB6402 を選んだところ、キーボードのペアリングができた。

### 4. SSH設定

SSHはGUIから設定するとうまくいかない。

RaspberryPi上でターミナルを開き、下記のようにコマンドを実行したところうまくいった。

パスワードの設定。初期パスワードは `raspberry` である。

```
$ passwd
Changing password for pi.
(current) UNIX password: 
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
```

SSHサーバ起動とRaspberryPi再起動。

```
$ sudo systemctl start ssh
$ sudo systemctl enable ssh
$ sudo shutdown -r now
```

GUIでやって Connection reset されるようになってしまった場合には、以下の呪文。

```
$ sudo touch /boot/ssh
$ sudo rm /etc/ssh/ssh_host_*
$ sudo dpkg-reconfigure oppenssh-server
```


### 5. SSH接続

RaspberryPiのIPアドレスを調べる。

```
$ ifconfig | grep -A1 eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.14  netmask 255.255.255.0  broadcast 192.168.0.255
```

ローカルPCからSSH接続する。ユーザーは `pi`、パスワードはさきほど設定したやつ。

```
% ssh pi@192.168.0.14
pi@192.168.0.14's password: 
Linux raspberrypi 4.14.34-v7+ #1110 SMP Mon Apr 16 15:18:51 BST 2018 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jun  5 17:18:48 2018
```

うまく接続できた！

微妙に更新されている。

```
$ uname -a
Linux raspberrypi 4.14.34-v7+ #1110 SMP Mon Apr 16 15:18:51 BST 2018 armv7l GNU/Linux
```

OS情報は2017年版と変わっていない。

```
$ cat /etc/os-release 
PRETTY_NAME="Raspbian GNU/Linux 9 (stretch)"
NAME="Raspbian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=raspbian
ID_LIKE=debian
HOME_URL="http://www.raspbian.org/"
SUPPORT_URL="http://www.raspbian.org/RaspbianForums"
BUG_REPORT_URL="http://www.raspbian.org/RaspbianBugs"
```

残りは MESH Hub 設定時、ミクさん召喚時と同じなので省略。

- [Raspberry PiにSony MESH Hubを設定した](/smart-home/2018/01/20/meshhub.html)
- [Sony MESH Hubでミクさんにおかえりを言ってもらった](/smart-home/2018/03/12/raspberrypi-echo.html)

### 参考サイト

- [Raspberry Pi の SSH 接続ではまる…(´・ω・)　[Raspberry Pi]: 息子と一緒にMakers](http://makers-with-myson.blog.so-net.ne.jp/2017-06-03)
- [Connection reset when login to ssh - Raspberry Pi Forums](https://www.raspberrypi.org/forums/viewtopic.php?t=206998)

