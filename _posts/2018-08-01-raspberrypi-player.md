---
layout: post
title:  "Raspberry Piでネットワークオーディオ"
categories: smart-home
---

今回は技術的に新しいことは特に無い。次の展開のためのメモである。

これまで Raspberry Pi を使って初音ミクさんにしゃべってもらう試みを行ってきた。

- [RaspberryPiにSony MESH Hubを設定した](/smart-home/2018/01/20/meshhub.html)
- [Sony MESH Hubでミクさんにおかえりを言ってもらった](/smart-home/2018/03/12/raspberrypi-echo.html)
- [Sony MESH Hubでミクさんにおかえりを言ってもらった その2](smart-home/2018/06/13/raspberry-miku.html)

せっかくのミクさんなのでやっぱり歌って欲しいよね、というところだが
これまたせっかくなので自宅の NAS 上の初音ミクさん楽曲を直接歌ってもらおうと思う。
この記事には NAS のマウント、および MP4 の再生を行うコマンドを書く。

本当はいい感じのタイミングで歌ってもらおうと思っているけれども、
設計がまだなので歌ってもらうコマンドだけ。

### 1. NAS のマウント

普通にマウントできた。

```
$ sudo mount.cifs //192.168.0.100/library /mnt -o user=****,pass=****
```

毎回コマンドを書くのは面倒くさいので fstab に書いた。
4行目がそれ。

```
$ cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=ce03f841-01  /boot           vfat    defaults          0       2
PARTUUID=ce03f841-02  /               ext4    defaults,noatime  0       1
//192.168.0.100/library /home/pi/Network cifs   defaults,user,rw,username=****,password=****,iocharset=utf8,uid=1000,gid=1000 0 2
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
```

マウントポイントを作成してから、マウントする。

```
$ mkdir Network
$ sudo mount Network
$ ls Network
```

NAS 上のファイルが表示されたら成功。

### 2. 歌ってもらう

MP4 を再生するだけ。

プレイヤーは `mplayer` を使った。
MP4 を再生できるだけでなく、出力先を PulseAudio にできるので今回都合がよい。

```
mplayer -msglevel all=0 -af volume=-20 -nolirc -noconsolecontrols Network/music//初音ミク/雪あかりの夜想曲/01\ -\ 四角い地球を丸くする.flac 
```

Echo 等、所望のスピーカーから音楽が再生されたら成功。

