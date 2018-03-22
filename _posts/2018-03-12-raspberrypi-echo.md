---
layout: post
title:  "Sony MESH Hubでミクさんにおかえりを言ってもらった"
categories: smart-home
---

![MESH Miku Tag](/images/screenshots/2018-03-21-mikutag.png)

ちょっと前に [RaspberryPiにSony MESH Hubを設定した](/smart-home/2018/01/20/meshhub.html)。

Sony MESH Hub は Raspberry Pi 上で動作する Sony MESH ゲートウェイサーバである。
Raspberry Pi を常時電源に接続しておくことで MESH のレシピを24時間動作させることができる。

我が家には MESH の人感センサータグと Phillips Hue があり、これらを接続するレシピを組んで、玄関の人感センサー付きライトを実現している。既製品と比べレシピの組み方でいくらでも自分にあったチューニングが効くところが便利だ。


さて最近、Raspberry Pi に初音ミクの声のしゃべってもらうという記事を見つけた。

[Open JTalkで初音ミクの声でおしゃべりさせる on Mac/Linux/Raspberry Pi](http://karaage.hatenadiary.jp/entry/2016/07/22/073000)

この記事では、Raspberry Pi 上でミクさんの声を合成し、Raspberry Pi に接続したスピーカーから音声を出力させている。

Sony MESH Hub が Raspberry Pi 上で動作し、かつ Raspberry Pi でミクさんにしゃべってもらえるのならば、Sony MESH のレシピをトリガーにしてミクさんにしゃべってもらうことも当然できるはずだ。

ということは家に帰ったらミクさんにおかえりと言ってもらえるっていうのもできるはず。

そこで今回は、玄関においた Sony MESH の人感センサーが反応したら、ミクさんがおかえりを言ってくれるという仕組みを作ってみた。

そうは言うものの普通、MESH のレシピから実現できる機能は公式で提供する範囲に限定されているため、ミクさんにしゃべってもらうというのは簡単ではない。

本稿では、

- Raspberry Pi を Bluetooth スピーカーに接続
- ミクさんの声を合成しスピーカーから出力するアプリケーションを Raspberry Pi 上に構築
- Sony MESH Hub に接続された人感センサータグが反応したら
- アプリケーション実行して初音ミクにしゃべってもらう

というようにして、この仕組みを実現した。

### 1. Raspberry Pi を Echo Dot に接続

突然 Echo Dot が出てきたが、単なる Bluetooth スピーカーとして使う。
ちょうどいいところにあったから、というのの他、 Raspberry Pi をスマートスピーカーと連携させれば今後色々と夢が広がりそうだ。とはいえ本稿の範囲ではただの Bluetooth スピーカーでしかない。

Raspberry Pi を Echo Dot に接続する。

#### 前準備

例のごとくRaspberryPiに差さるキーボードが無いため、
SSHで繋いでCUIからBluetoothを設定する。

必要なライブラリをインストールして再起動。

```
$ sudo apt-get install bluetooth pi-bluetooth
$ sudo reboot
```

この時点では Bluetooth がうまく動かない。
`No default controller available` と言われているが、
Raspberry Pi 側の Bluetooth デバイスが見付からない、という意味だろう。

```
$ bluetoothctl 
[bluetooth]# scan on
No default controller available
```

なんとなく Bluetooth デーモンの状態も以下のようにして見てみた。

```
$ systemctl status bluetooth
● bluetooth.service - Bluetooth service
   Loaded: loaded (/lib/systemd/system/bluetooth.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2018-03-12 21:36:13 JST; 11min ago
     Docs: man:bluetoothd(8)
 Main PID: 3144 (bluetoothd)
   Status: "Running"
   CGroup: /system.slice/bluetooth.service
           └─3144 /usr/lib/bluetooth/bluetoothd

 3月 12 21:36:13 fg-rasp1 systemd[1]: Starting Bluetooth service...
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Bluetooth daemon 5.43
 3月 12 21:36:13 fg-rasp1 systemd[1]: Started Bluetooth service.
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Starting SDP server
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Bluetooth management interface 1.14 initialized
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Failed to obtain handles for "Service Changed" characteristic
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Sap driver initialization failed.
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: sap-server: Operation not permitted (1)
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Endpoint registered: sender=:1.26 path=/A2DP/SBC/Source/1
 3月 12 21:36:13 fg-rasp1 bluetoothd[3144]: Endpoint registered: sender=:1.26 path=/A2DP/SBC/Sink/1
```

ドライバの初期化に失敗した風なことが書いてあった。
詳しくはわからないが、このままではBluetoothを使えなそうであることは間違いない。

#### Bluetooth の権限を与える

ググっていたら以下の記事が見つかった。
デフォルトでは `bluetooth` グループしか Bluetooth のインターフェースに接続できないようになっているらしい。

[Bluetooth does not work with Raspbian Stretch and Raspberry Pi 3 - Raspberry Pi Slack Exchange](https://raspberrypi.stackexchange.com/questions/71333/bluetooth-does-not-work-with-raspbian-stretch-and-raspberry-pi-3)

Bluetooth の設定ファイルがあり、うちでも確認してみたところ、次のように書かれていた。

`bluetooth` グループは `allow` して、それ以外は `deny` する、という雰囲気がすごい伝わってくる。

```
$ cat /etc/dbus-1/system.d/bluetooth.conf

  <!-- allow users of bluetooth group to communicate -->
  <policy group="bluetooth">
    <allow send_destination="org.bluez"/>
  </policy>

  <policy context="default">
    <deny send_destination="org.bluez"/>
  </policy>
```

ということで `pi` ユーザを `bluetooth` グループに加えてやり、また再起動。

```
$ sudo usermod -a -G bluetooth pi
$ sudo reboot
```

再起動後 `pi` ユーザで `busctl` を実行してみると、HCIのインターフェースが表示された。これでOKそうだ。

```
$ busctl tree org.bluez
└─/org
  └─/org/bluez
    └─/org/bluez/hci0
```

ちなみにHCIとは Host Controller Interface の略で、

> Bluetoothにおいて、コントローラーとホストの間で使われる通信プロトコル。
> 
> Bluetoothモジュールを制御したり、データを無線で送受信するために使う、一番最下層のインターフェイスである。
> 有線でも例えばEthernetならEthernetコントローラーICなどを使い、その上にEthernetのプロトコルから上を実装することになるが、Bluetoothのような無線でも同様にハードウェアとソフトウェアの分担があり、その境界となるのがHCIである。
> [HCI (Bluetooth) - 通信用語の基礎知識](http://www.wdic.org/w/WDIC/HCI%20(Bluetooth))

とのことだ。
ハードウェア(Bluetoothコントローラー)とソフトウェア(OS)の間のインターフェースであるとみた。

#### Echo Dotを接続する

再び `bluetoothctl` を実行してみたところ何か出た。

```
$ bluetoothctl 
[NEW] Controller B8:27:EB:B9:E2:47 fg-rasp1 [default]
[NEW] Device E3:14:0A:18:80:AB E3-14-0A-18-80-AB
```

とりあえずデバイスをスキャンする。

この辺はちょっと苦労したのだが、しばらく粘っていたら突然 Echo Dot がリストに現れたんだったような気がする。
スマートフォンでAlexaアプリを起動し、Bluetooth接続のセットアップをしたような結局いらなかったような、とか結局どれが必須な手続きだったのか判然としない。

ともかく、見つかれば以下のようにデバイス一覧に Echo Dot が表示される。

```
[E3-14-0A-18-80-AB]# agent on
Agent registered
[E3-14-0A-18-80-AB]# scan on
Discovery started
[CHG] Controller B8:27:EB:B9:E2:47 Discovering: yes
[NEW] Device 40:CB:C0:D6:76:7F 40-CB-C0-D6-76-7F
[NEW] Device 31:9E:8A:69:4A:31 31-9E-8A-69-4A-31
[NEW] Device FC:65:DE:AA:E2:6B Echo Dot-69D
[CHG] Device 40:CB:C0:D6:76:7F RSSI: -100
```

Echo Dot を捕まえたら、すかさずペアリングする。

```
[E3-14-0A-18-80-AB]# pair FC:65:DE:AA:E2:6B
Attempting to pair with FC:65:DE:AA:E2:6B
Failed to pair: org.bluez.Error.AlreadyExists
[CHG] Device 40:CB:C0:D6:76:7F RSSI: -92
[CHG] Device A1:00:00:00:19:FB ManufacturerData Key: 0xd200
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x44
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x4c
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x47
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x2d
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x42
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x4c
[CHG] Device A1:00:00:00:19:FB ManufacturerData Value: 0x45
[CHG] Device 40:CB:C0:D6:76:7F RSSI: -100
```

接続する。

```
[E3-14-0A-18-80-AB]# connect FC:65:DE:AA:E2:6B
Attempting to connect to FC:65:DE:AA:E2:6B
[CHG] Device FC:65:DE:AA:E2:6B Connected: yes
[CHG] Device FC:65:DE:AA:E2:6B Modalias: usb:v1949p1200d5100
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 0000110a-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 0000110b-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 00001200-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 00001800-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B UUIDs: 00001801-0000-1000-8000-00805f9b34fb
[CHG] Device FC:65:DE:AA:E2:6B ServicesResolved: yes
Connection successful
[CHG] Device 40:CB:C0:D6:76:7F RSSI: -90
[CHG] Device A1:00:00:00:19:FB RSSI: -64
```

ここまでうまくいくと、Echoが突然しゃべった。
うちの場合は、「えふじーらすぷいちにせつぞくしました」としゃべっていた。

「えふじーらすぷいち」は私の Raspberry Pi のホスト名。

あと、自動的に接続されるようになるらしいので `trust` しておく。

```
[Echo Dot-69D]# trust FC:65:DE:AA:E2:6B
[CHG] Device FC:65:DE:AA:E2:6B Trusted: yes
Changing FC:65:DE:AA:E2:6B trust succeeded
```

今後はペアリング済みデバイスを下のようにして見ることができる。

```
[Echo Dot-69D]# paired-devices
Device FC:65:DE:AA:E2:6B Echo Dot-69D
```

Echo Dot との接続状態は、`info` で見られる。
`Connected: yes` となっているので、うまくいったのだと思う。

```
[Echo Dot-69D]# info FC:65:DE:AA:E2:6B
Device FC:65:DE:AA:E2:6B
	Name: Echo Dot-69D
	Alias: Echo Dot-69D
	Class: 0x2c0414
	Icon: audio-card
	Paired: yes
	Trusted: yes
	Blocked: no
	Connected: yes
	LegacyPairing: no
	UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
	UUID: Audio Sink                (0000110b-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
	UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
	UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
	UUID: Generic Access Profile    (00001800-0000-1000-8000-00805f9b34fb)
	UUID: Generic Attribute Profile (00001801-0000-1000-8000-00805f9b34fb)
	Modalias: usb:v1949p1200d5100
```

これで RaspberryPi と Echo Dot を接続できた。
次は Echo Dot を Bluetooth スピーカーとして使うための設定を行う。

#### Echo Dot をスピーカーにする

スピーカー、というか音声出力の設定は Pulse Audio を使う。
Pulse Audio はざっくり言うと音声入力と出力を繋ぐアプリケーションであり、
Linux 上ではサウンドサーバとして動作する。

ここは [Raspberry Pi 3B + BluetoothスピーカでAmazon Alexaを安く構築（１　まずは音を鳴らす）](https://qiita.com/onelittlenightmusic/items/05b262c60c4889c07ca9) で紹介されている手順を参考にした。

まずは Pulse Audio 関連のライブラリをインストールする。

```
$ sudo apt-get install pulseaudio pavucontrol pulseaudio-module-bluetooth
```

Pulse Audio を Systemd で管理するので `pulseaudio.service` ファイルを作っておく。

```
# cat /etc/systemd/system/pulseaudio.service 
[Unit]
Description=Pulse Audio

[Service]
Type=simple
ExecStart=/usr/bin/pulseaudio --system --disallow-exit --disable-shm

[Install]
WantedBy=multi-user.target
```


そしてサービスを起動＆うまくいくこと前提で自動起動ON。

```
# systemctl start pulseaudio.service
# systemctl enable pulseaudio.service
```

ステータス確認。

```
# systemctl status pulseaudio.service
● pulseaudio.service - Pulse Audio
   Loaded: loaded (/etc/systemd/system/pulseaudio.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-03-13 00:27:29 JST; 5s ago
 Main PID: 20901 (pulseaudio)
   CGroup: /system.slice/pulseaudio.service
           └─20901 /usr/bin/pulseaudio --system --disallow-exit --disable-shm

 3月 13 00:27:29 fg-rasp1 systemd[1]: Started Pulse Audio.
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] main.c: Running in system mode, but --disall
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: N: [pulseaudio] main.c: Running in system mode, forcibly dis
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] main.c: OK, so you are running PA in system 
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] main.c: Please read http://www.freedesktop.o
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] authkey.c: Failed to open cookie file '/var/
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] authkey.c: Failed to load authentication key
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] authkey.c: Failed to open cookie file '/var/
 3月 13 00:27:29 fg-rasp1 pulseaudio[20901]: W: [pulseaudio] authkey.c: Failed to load authentication key
```

`active (running)` だけど色々エラーが出ている場合もある。
このときは悩むより pulseaudio を再起動するだけで直ったりした。

再起動後↓

```
● pulseaudio.service - Pulse Audio
   Loaded: loaded (/etc/systemd/system/pulseaudio.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2018-03-14 08:03:27 JST; 2min 43s ago
 Main PID: 309 (pulseaudio)
   CGroup: /system.slice/pulseaudio.service
           └─309 /usr/bin/pulseaudio --system --disallow-exit --disable-shm

 3月 23 01:50:39 fg-rasp1 pulseaudio[8042]: W: [pulseaudio] main.c: Running in system mode, but --disallo-module-loading not set.
 3月 23 01:50:39 fg-rasp1 pulseaudio[8042]: N: [pulseaudio] main.c: Running in system mode, forcibly disaling exit idle time.
 3月 23 01:50:39 fg-rasp1 pulseaudio[8042]: W: [pulseaudio] main.c: OK, so you are running PA in system mde. Please make sure that you actually do want to do that.
 3月 23 01:50:39 fg-rasp1 pulseaudio[8042]: W: [pulseaudio] main.c: Please read http://www.freedesktop.og/wiki/Software/PulseAudio/Documentation/User/WhatIsWrongWithSystemWide/ for an explanation why system mode is usually a bad idea.
```

`--disallow-module-loading` が無い、とログに書いてある場合もあるが、
これは無視でよかった。逆にこのフラグを追加すると Echo Dot から音が出なくなった。


以下は Pulse Audio の音声出力先を Bluetooth スピーカーに向けるための設定。

先に挙げたページに従い実施してみた設定だが、あとで気付いたところ私のミスで色々と移し間違えていた。そうであるにも関わらず Echo Dot から再生できたので必須かどうかわからない。

```
$ sudo vi /etc/pulse/system.pa
$ tail -9 /etc/pulse/system.pa

## Load driver modules for Bluetooth hardware
.ifexists module-blue-tooth-policy.so
load-module module-bluetooth-policy
.endif

.ifexists module-bluetooth-discover.so
load-module module-bluetooth-discover
.endif
```

これは必須かもしれない。

世の記事では `/etc/dbus-1/system.d/pulseaudio-bluetooth.conf` に書いているようだが、面倒なので `/etc/dbus-1/system.d/bluetooth.conf` に追記した。`pulse` ユーザが `org.bruez` に接続することを許可する、そう言っている。

```
$ sudo vi /etc/dbus-1/system.d/bluetooth.conf 
$ tail -4 /etc/dbus-1/system.d/bluetooth.conf
  <policy group="pulse">
    <allow send_destination="org.bluez"/>
  </policy>
</busconfig>
```

これも必須かわからない。

```
$ sudo vi /etc/dbus-1/system.d/pulseaudio-system.conf
$ tail -10 /etc/dbus-1/system.d/pulseaudio-system.conf
<busconfig>
  <!-- System-wide PulseAudio runs as 'pulse' user. This fragment is
       not necessary for user PulseAudio instances. -->

  <policy user="pulse">
    <allow own="org.pulseaudio.Server"/>
  </policy>

</busconfig>
```

最後にローカルのWAVを再生してみる。
雨のような音が Echo から出たら成功である。

```
$ aplay /usr/share/sounds/alsa/Noise.wav
```

これでもよい。

```
$ aplay -D default /usr/share/sounds/alsa/Noise.wav
```

`default` って何ですか、というと Pulse Audio から再生するよ、ってことである。

```
$ aplay -L | grep -A1 default
default
    Playback/recording through the PulseAudio sound server
sysdefault:CARD=ALSA
    bcm2835 ALSA, bcm2835 ALSA
```

Echo Dot から音が出ているとき、Pulse Audio の設定は以下のようになっていた。

```
$ pactl info
Server String: /var/run/pulse/native
Library Protocol Version: 32
Server Protocol Version: 32
Is Local: yes
Client Index: 67
Tile Size: 65496
User Name: pulse
Host Name: fg-rasp1
Server Name: pulseaudio
Server Version: 10.0
Default Sample Specification: s16le 2ch 44100Hz
Default Channel Map: front-left,front-right
Default Sink: bluez_sink.FC_65_DE_AA_E2_6B.a2dp_sink
Default Source: bluez_sink.FC_65_DE_AA_E2_6B.a2dp_sink.monitor
Cookie: b519:85cf
```

いつのまにそうなったのか不明だが `Default Sink` が Echo Dot への Bluethtooth 出力になっている。
A2DPというのは音楽配信を得意とする Bluetooth のプロファイルだそうだ。

[ケータイ用語の基礎知識 第259回 : A2DPとは](https://k-tai.watch.impress.co.jp/cda/article/keyword/27461.html)

これまで意識していなかったけれども Echo Dot のヘルプにもちゃんと書いてあった。

> また、モバイル端末（スマートフォンやタブレットなど）からEcho Dotにオーディオをストリーミング再生することもできます。
> [Echo Dot向けの認定スピーカーとサポートされているBluetoothプロファイル](https://www.amazon.co.jp/gp/help/customer/display.html?nodeId=202011880)

うまくいかなくて困ったときには、そもそも Pulse Audio から Echo Dot が見えているかどうかを以下のコマンドで確認することができるかもしれない。

```
$ pactl list short sinks
0	alsa_output.platform-soc_audio.analog-stereo	module-alsa-card.c	s16le 2ch 48000Hz	SUSPENDED
1	bluez_sink.FC_65_DE_AA_E2_6B.a2dp_sink	module-bluez5-device.c	s16le 2ch 44100Hz	SUSPENDED
$ pactl list short cards
0	alsa_card.platform-soc_audio	module-alsa-card.c
1	bluez_card.FC_65_DE_AA_E2_6B	module-bluez5-device.c
```

さてなかなかに長い道程だったが Bluetooth とか Pulse Audio とかに少し詳しくなったところで、いよいよミクさんに出番である。

## 2. ミクさんにしゃべってもらうアプリケーション

ミクさんの声は音声合成する。
これには OpenJTalk を使った先例が下記にあったので真似をした。

[Open JTalkで初音ミクの声でおしゃべりさせる on Mac/Linux/Raspberry Pi](http://karaage.hatenadiary.jp/entry/2016/07/22/073000)

まずは OpenJTalk のライブラリをインストール。

```
$ sudo apt-get install open-jtalk open-jtalk-mecab-naist-jdic hts-voice-nitech-jp-atr503-m001
```

音声合成をするにあたり、OpenJTalk用のミクさん音声モデルを生成する必要がある。
生成すると .htsvoice ファイルが出来上がる。

この方法は上記ページにあり、全く同じ手順を取ったので本稿では割愛する。
会社のMacでこっそり作業して生成したが、特に苦労する点は無く速やかに完了した。

作った音声合成モデルを使い、ミクの声を合成する。
以下のようにすれば直ちに実行することができる。

```
$ echo "こんにちは、初音ミクです。" | open_jtalk -x /var/lib/mecab/dic/open-jtalk/naist-jdic -m /usr/share/hts-voice/miku/miku.htsvoice -r 1.0 -ow miku.wav
```

そして再生してみて、Echo の中からミクさんの声が聴こえてきたらOK。

```
$ aplay miku.wav
```

ついにミクさんが我が家に降臨された。

`open_jtalk` プログラムには、`-r 1.0` 等、様々な音声合成のパラメータがあるので、好みに合わせていじってみてもいいかもしれない。

ここでつい満足したくなるが、
Sony MESH のイベントをトリガーにして、ミクさんにしゃべってもらうところまでやってゴールだった。

言うだけなら簡単だけども、実際にミクさんにしゃべってもらうには Sony MESH から `aplay` までを繋がなければならない。普通の MESH の機能では当然これは実現できない。

どうしたものか。

### 3. Sony MESH から Raspberry Pi を動かす

MESH SDKというものがあるらしい。
これを使うと任意のJavaScriptを実行するカスタムタグを作ることができる。
JavaScript で Ajax を使い、Raspberry Pi 上の WebAPI を実行するようにすれば、MESH から Raspberry Pi 上で任意のプログラムを動かすことができそうだ。

そこで、次の構成で Sony MESH から `aplay` までを繋ぐことにした。

1. Raspberry Pi 上で WebAPI サーバを動かしておく
2. MESH のカスタムタグから WebAPI へ HTTP リクエスト発行
3. WebAPI の中で `open_jtalk` および `aplay` を呼び出す

#### WebAPI サーバを起動する

まずは WebAPI サーバ を作成する。
サーバは Flask を使って簡単に作ってしまう。

なお、今回は localhost のみから接続できればよい。

```
$ sudo pip install Flask
```

こんな Web アプリケーションをつくった。
POST で受け取ったテキストをミクさんに読み上げてもらう。

```
#!/usr/bin/perl
# -*- coding:utf-8 -*-

import subprocess

from flask import Flask, request
app = Flask(__name__)

def jtalk(outwav, speech):
  command = [
    '/usr/bin/open_jtalk',
    '-x', '/var/lib/mecab/dic/open-jtalk/naist-jdic',
    '-m', '/usr/share/hts-voice/miku/miku.htsvoice',
    '-ow', outwav
  ]
  p = subprocess.Popen(command, stdin=subprocess.PIPE)
  p.stdin.write(speech)
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

@app.route('/say', methods=['POST'])
def handleSay():
  speech = request.form['speech'].encode('utf-8')
  return say(speech)

def main():
  app.run('127.0.0.1')

if __name__ == '__main__':
  main()
```

WebAPI サーバを起動して、

```
$ python start.py
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

HTTPリクエストしてみる。

```
$ curl --data "speech=次は何を読もうかな？" http://localhost:5000/say
```

Echo Dot の中からミクさんの声が聴こえてきたらOK。

#### MESH カスタムタグから HTTP リクエストする

カスタムタグは[MESH SDKのページ](https://meshprj.com/sdk/mypage/)から作ることができる。

以下のようなものを作った。

![Mesk SDK](/images/screenshots/2018-03-21-meshsdk.png)

Functionを1つ用意する。「Function Name」は「しゃべる」とかでいいだろう。

##### Connector

「Input Connector」,「Output Connector」は 1つずつ。label は空欄のままでよい。

##### Property

「Property」は1つで、次のように設定した。

- Property Name: Text
- Reference Name: speech
- Type: Text
- Default Value: こんにちは、初音ミクです。

##### Code

「Execute」以外は空でよさそう。

次のようなプログラムを書いた。

```
var url = 'http://localhost:5000/say';
var speech = properties.speech;

ajax({
	url: url,
	type: 'post',
	data: { 'speech': speech },
	timeout: 5000,
	success: function(success) {
		log(success);
		callbackSuccess({
			resultType: 'continue'
		});
	},
	error: function(request, error) {
		log(error);
		callbackSuccess({
			resultType: 'continue'
		});
	}
});
return {
	resultType: 'pause'
};
```

ここまでやった後、MESH SDK のページで画面上部の 「Save」ボタンを押す。

そして MESH Hub のレシピ作成画面で「カスタム」とあるところの「追加」ボタンを押すと、
今作ったカスタムタグ「ミク」が現われるはず。

とりあえずレシピ上で人感タグとミクタグを接続してみる。
この状態で人感タグに反応させて、Echo Dot から「こんにちは、初音ミクです。」と聴こえてきたら成功だ！

### 4. おかえりレシピを作成する

今回つくったレシピでは、人感センサータグとミクさんタグを利用する。

毎日18時以降、初めて人を感知したときにミクさんに「おかえり」を言ってもらいたいと思う。

以下のようなレシピを作った。

![MESH Miku Recipe](/images/screenshots/2018-03-21-mikurecipe.png)

人感センサーが反応したら Hue を点灯させる。
同時に、ミクさんに「おかえり！」としゃべってもらう。

ここでおかえりは1回だけしゃべってもらえれば十分なため、
スイッチを使って2回目以降のセンサー反応時にはミクタグへ行かないガードを作った。

さらに左下のタイマーは平日18時に発火する設定にした。
18時になるとスイッチがリセットされるのでガードが解除され、
次回のセンサー反応時にはまた、ミクタグが発火することになる。

### おわりに

ここまでで機能としては十分だが、実のところ残念ながらミクさんはほとんどしゃべってくれない。
MESH Hub と Echo Dot の Bluetooth 接続がよく切れてしまうためと思われる。

ミクさんのいる生活を送るためには、この辺維持するための工夫とかが必要だ。
また改良ができたらこのサイトで取り上げたい。

なお、ミクタグのアイコンは、普段使いのFedora上でUnityを実行し、[MMD4Mecanim (Beta)](http://stereoarts.jp/)で[16bit式ミク](http://www.nicovideo.jp/watch/sm10791865)をFBX形式に変換して作った。
