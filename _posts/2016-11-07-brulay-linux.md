---
layout: post
title:  "Fedora25でBru-layを再生した"
categories: home
---

Fedora25でBru-layを再生するために、いくつか手順を踏んだので、そのメモです。

最終的には、

- Fedora25
- Bru-lay
- SMPlayer
- USB Audio

といったつくりで無事、コンテンツを楽しむことができました。

### 参考にしたページ

1. [Fedora 23 ： Blu-ray を再生・視聴する なんとかネット。](http://laniusbucephalus.blog49.fc2.com/blog-entry-571.html)


### 環境設定編

はじめは参考ページ1に従って設定を行いました。

#### 1. Negativo17 リポジトリの追加
まんま従っています。

```
$ sudo dnf config-manager --add-repo=http://negativo17.org/repos/fedora-handbrake.repo
```

#### 2. 必要なソフトウェアのインストール
これもまんまです。

```
$ sudo dnf -y install libbdplus makemkv
```

#### 3. `KEYDB.cfg`のインストール
AACS規格を使っているBlu-rayディスクのデータをデコードするためのキーなんだそうです。

```
$ mkdir ~/.config/aacs/
$ cd ~/.config/aacs
$ wget -c http://www.labdv.com/aacs/KEYDB.cfg

$ sudo cp -a ~/.config/aacs /etc/xdg
$ sudo chown -R root:root /etc/xdg/aacs
```

#### 4. bdplusのインストール
BD+への対応のようです。再生したいBrulayがBD+で保護されていなければ、この操作はいらないのかもしれません。

```
$ cd ~/.config
$ wget -c http://www.labdv.com/aacs/libbdplus/bdplus-vm0.bz2
$ tar -xvjf bdplus-vm0.bz2

$ sudo cp -a ~/.config/bdplus /etc/xdg
$ sudo chown -R root:root /etc/xdg/bdplus
```

#### 5. libaacsの設定
libaacsとして`libmmbd`を使うようにします。
これは、libmmbdがlibaacsの上位互換だということなんでしょうかね？

```
$ sudo ln -s /usr/lib64/libmmbd.so.0 /usr/lib64/libaacs.so.0
```

#### 6. 環境変数の設定
`libmmbd.so.0`を使うように変数を設定しています。

```
$ cat /etc/profile.d/makemkv.sh
export LIBBDPLUS_PATH=/usr/lib64/libmmbd.so.0
export LIBAACS_PATH=/usr/lib64/libmmbd.so.0

$ source /etc/profile.d/makemkv.sh

$ file $LIBBDPLUS_PATH
/usr/lib64/libmmbd.so.0: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=5affc702602a170981fbf4d1dc176507be52d28e, stripped
$ file $LIBAACS_PATH  
/usr/lib64/libmmbd.so.0: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=5affc702602a170981fbf4d1dc176507be52d28e, stripped

```

環境設定はこれだけです。

はじめlibmmbd.so.0を消してしまうというミスを犯しましたが、makemkvの再インストールからやり直して復活できました。

### Bru-lay再生編

#### 1. mplayerを使う
やや苦戦しましたが、マウントしてから`mplayer`に渡したら見られました。

```
$ sudo mount /dev/sr1 /media
$ mplayer -lavdopts threads=2 -ao alsa:device=hw=2.0 br:////media
```

![Bru-lay](/images/screenshots/2016-11-07-brulay.jpg)

##### USB Audio対応
オーディオがUSB Audioになっていなかったため、出力先を変えるオプションを付与しました。

```
-ao alsa:device=hw=2.0
```

`2.0`の部分は`aplay -l`を使って調べました。
カード2のデバイス1だから、`2.0`。
```
$ aplay -l
**** ハードウェアデバイス PLAYBACK のリスト ****
カード 0: HDMI [HDA Intel HDMI], デバイス 3: HDMI 0 [HDMI 0]
  サブデバイス: 1/1
  サブデバイス #0: subdevice #0
カード 0: HDMI [HDA Intel HDMI], デバイス 7: HDMI 1 [HDMI 1]
  サブデバイス: 1/1
  サブデバイス #0: subdevice #0
カード 0: HDMI [HDA Intel HDMI], デバイス 8: HDMI 2 [HDMI 2]
  サブデバイス: 1/1
  サブデバイス #0: subdevice #0
カード 1: PCH [HDA Intel PCH], デバイス 0: CX20751/2 Analog [CX20751/2 Analog]
  サブデバイス: 1/1
  サブデバイス #0: subdevice #0
カード 2: PMA50 [PMA-50], デバイス 0: USB Audio [USB Audio]
  サブデバイス: 1/1
  サブデバイス #0: subdevice #0
```

##### 音ズレ対策
音が絵よりも先に進んでいく現象が起きていましたが、次のオプションを付与したら改善しました。1コアでの再生では間に合わなかったようです。
```
-lavdopts threads=2
```

#### 2. SMPlayerを使う
やっぱりmplayerでは寂しかったので、SMPlayerを使ってみました。

```
$ sudo dnf install smplayer
```

右クリック>「開く」>「ディスク」>「フォルダーのBru-lay」を選択し、`/media`を開いたら、再生できました。

あくまでディスクではないというところは気になりましたが、今回はスルーにしました。

##### USB Audio対応
右クリック>「オプション」>「環境設定」>「全般」>「オーディオ」>「出力ドライバー」から、それっぽいものを選択しました。

```
pulse (4 - <alsa_output.usb-D___M_Holdings_Inc._PMA-50-00.analog-stereo>)
```

##### 音ズレ対策
同「環境設定」>「パフォーマンス」>「パフォーマンス」>「デコードのスレッド」を`2`に設定しました。

