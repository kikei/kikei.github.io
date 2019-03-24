---
layout: post
title:  "Galaxy s7 Androidスマートフォンのroot権限取得"
categories: smartphone
---

Galaxy s7 (SM-G930FD) の root 権限を取得した。

目的は [Synergy](https://symless.com/synergy) の Android 版クライアントアプリを動かすことである。
このアプリを使うと PC に接続されたマウスを使い、
PC からシームレスにスマートフォンの操作ができるようになる。

実際には、マウスカーソルが PC からスマートフォンに移動するのでは無く、
スマートフォンに制御を写したら PC 側ではマウスカーソルを隠し、
スマートフォン上ではマウスカーソルをエミュレートするようになっている。

スマートフォンにマウスを差したときをイメージすれば少しわかりやすい。
Synergy アプリでは、アプリが仮想的なマウスになる感じである。
仮想マウスを PC 側に接続したマウスから Synergy サーバーを経由して動かす。
そんなことできたのか、という感じだが実際に可能である。
**ただしスマートフォン端末の root 権限が必要。**

そこで本記事では Galaxy s7 で root を取れるようになるまでの手順を案内する。

root 権限取得のためのアプリケーションとしては、[Magisk](https://magiskmanager.com/) を使う。
Magisk は Android の root 取得界隈の最新技術である。
これを使うことによりネットバンキングなど root 検知機能がついているアプリと
要 root 権限アプリを同一の端末上で共存させることができるらしい。

なお、今回作業したスマートフォンは Galaxy s7 (SM-G930FD) である。
これは日本では販売されなかったグローバルモデルの端末であり、
ドコモ等から発売されている Galaxy s7 Edge とは違うので注意されたし。
とはいえ手順はそれほど違わないはずなので役には立つことだろう。

#### 関連記事:

この記事は一応 [SynergyでLinuxからAndroidへマウス共有できるまで](https://kikei.github.io) の続きである。
ただし、本稿では Synergy 関連はあまり触れない。

1. [Synergy サーバーの起動](https://kikei.github.io/smartphone/2019/03/21/root-galaxys7.html)
2. この記事: [Synergy クライアントの起動 Galaxy s7 権限ルート取得編](https://kikei.github.io/smartphone/2019/03/21/root-galaxys7.html)
3. [Synergyクライアント起動 SELinux編](https://kikei.github.io/smartphone/2019/03/22/synergy-selinux.html)

#### 念のための注記:

ここに書かれた方法や、その方法を参考にするのは自己責任で実行すること。
端末を文鎮にしてしまったり、その他、不幸なことが起きたとしても一切の責任を負いません。

#### 全体的な手順:

1. ハードウェア, ソフトウェアの用意
2. Stock ROM を焼
3. リカバリに TWRP を焼く
4. Magisk をインストールする

### 1. 用意したもの

#### ハードウェア:

- スマートフォン端末 Galaxy s7 (SM-G930FD)
- Windows 10 PC
- USB - micro USB ケーブル 1本

今回 Galaxy s7 の各バージョンは以下のようになっていた。

![Versions](/images/photos/2019-03-21-versions.jpg)

#### ソフトウェア:

- FM-G930FD Stock ROM: G930FXXU3ESA3_G930FOZS3ERK3_TGY.zip 
- ADB (Android SDK)
- Odin3 v3.13.1 
- Magisk v18.1
- no-verity-opt-encrypt-6.0.zip
- twrp-3.2.3-0-herolte.img.tar
- remove encryption.zip
- Magisk Manager v18.1

集めるのが大変だった。

### 2. Stock ROM を焼く

Stock ROM というのはつまりファームウェアである。
真っ新なファームウェアを端末に書き込み、端末を真の製造時の状態へ戻す。

#### 2.1. Stock ROM の取得

Stock ROM は Samusung の公式ページよりダウンロードすることができる。
以下のリンク先から見つけた。

[Firmware download for Galaxy S7 SM\-G930FD](https://www.sammobile.com/firmwares/galaxy-s7/SM-G930FD/)

たくさんの種類のファームウェアが置いてある。
どれを使えばよいかは、設定 > 端末情報 > ソフトウェア情報と辿るとわかる。

私の端末では以下のように書いてあった。

> ビルド番号
> R16NW G930FXXU3ESA3

ビルド番号と同じ文字の羅列を先のページより探すと香港版ファームウェアが見つかった。

> Hong Kong 2019-02-25 8.0.0 G930FXXU3ESA3 G930FOZS3ERK3

台湾版も同じビルド番号だったが、この端末は香港から来た子だった気がするので香港を選んだ。

なお、私の場合は無駄にダウンロードに苦労した。
ファームウェアのダウンロードが始まらなかったのは、ブラウザのアドブロックに引掛っていたようだ。そのことに気付かずにページ内をしばらく右往左往していた。

さらにファイルサイズは2.4GBあるにも関わらず、下り速度が2.4Mbpsしか無かったため
ダウンロードには約3時間もかかった。早くするには Samsung に金を払わなければならない。

ダウンロードしたファイルは展開すると幾つかのバイナリファイルが得られる:

```sh
$ unzip G930FXXU3ESA3_G930FOZS3ERK3_TGY.zip 
Archive:  G930FXXU3ESA3_G930FOZS3ERK3_TGY.zip
  inflating: BL_G930FXXU3ESA3_CL14843133_QB21430938_REV00_user_low_ship.tar.md5
  inflating: AP_G930FXXU3ESA3_CL14843133_QB21430938_REV00_user_low_ship_meta.tar.md5
  inflating: CP_G930FXXU3ERL3_CP11453879_CL14843133_QB21134944_REV00_user_low_ship.tar.md5
  inflating: HOME_CSC_OZS_G930FOZS3ERK3_CL13630734_QB20780064_REV00_user_low_ship.tar.md5
  inflating: CSC_OZS_G930FOZS3ERK3_CL13630734_QB20780064_REV00_user_low_ship.tar.md5
```

#### 2.2. Odin3 の起動

書き込みは Odin3 というソフトウェアを使う。

残念ながらこれは Windows 専用のプログラムである。
JOdin3 という Java 実装版もあり、これならば Linux でも起動することはできるが、
バージョンが古く、私は書き込みが成功しなかった。
どうも端末側も日々進化しており、書き込めるソフトも常に追従していかなければならないようだ。

Windows には adb もインストールしておく。
当然ながら端末側も adb が利用できるように諸々の設定をしておく必要がある。
この手順は普通であるから省略する。

あと端末の開発者向けオプションの中で OEM ロック解除を有効にしておく必要もあるらしい。

Odin3 を起動し、先程取得したファームウェアのパスを Odin3 に設定する。
5つのファイルパスを指定する必要があるが、いずれもZIPファイルから得られる。
どの枠にどのファイルを指定すればいいかについても、ファイル名を見れば明らかである。
全て指定すれば以下の図のようになる:

![Odin3 x Galaxy Stock ROM](/images/photos/2019-03-21-odin1.jpg**

Start ボタンはまだ押さない。押しても何も起きない。
端末をダウンロードモードで接続してから押す。

#### 2.3. 端末のダウンロードモード起動

次に端末をダウンロードモードで起動する。
ダウンロードモードというのはカスタム OS の書き込むことができるモードのようだ。

ダウンロードモードで端末を起動するためには、まず一旦端末の電源を切り、
それから音量アップキーとホームボタンを同時に押しながら電源キーを押す。
うまくできれば以下のような画面が出て起動するので、さらに音量アップキーを押せば
ダウンロードモードになる。

![Download mode](/images/photos/2019-03-21-downloadmode.jpg)

この操作はこの後も幾度となく実行することになるので、
「ダウンロードモードの構え」と呼ぶことにする。

**ダウンロードモードの構え**: 音量アップキー、ホームボタン、電源キーの同時押し

この記事を大体書き終わってから思ったが「リカバリモード起動の構え」の方が適切な呼称の気もする。本稿ではそのままにした。

#### 2.4. Stock ROM の書き込み

Odin3 で Start ボタンを押す。
うまく始まればいかにも書き込んでます的な表示に変わり、
端末の画面にも Downloading... と表示される。

少しだけ時間がかかるので完了するまでまったり待機すればよい。

![Downloading...](/images/photos/2019-03-21-downloadmode2.jpg)

完了するとそのままの画面で止まるけれどそれで問題無い。
次の操作までこのまま放置しておく。

### 3. TWRP の書き込み

TWRP を端末のリカバリ領域に書き込む。
TWRP は端末のシステムを書き換えるために便利な機能を搭載したリカバリ OS である。
TWRP の書き込みも Stock ROM と同じ要領で実施する。

#### 3.1. Odin3 に TWRP を指定

Odin3 に戻る。

Odin3 の Reset ボタンを押し、各種指定を消去する。
そして TWRP のバイナリを AP に指定する。これは TAR のままでよい。

今回は twrp-3.2.3-0-herolte.img.tar を AP に指定した。

![Odin3 x TWRP](/images/photos/2019-03-21-odin2.jpg)

#### 3.2. 端末をダウンロードモードで起動

Stock ROM を書き込み完了した状態で放置していた端末に戻る。
これから再びダウンロードにするため、電源ボタンを長押しして端末を終了し、
ダウンロードモードの構えにて端末を起動する。
うまくいけばまた例の水色画面を表示しながら端末が起動するので、
音量アップキーを押せばダウンロードモードになる。

もしかしたら端末の画面が消えたら素早くダウンロードモードの構えに持ち替える必要が
あったかもしれないけど忘れた。

#### 3.3. TWRP の書き込み

Odin3 で Start ボタンをクリックする。

うまくいけばまた Downloading... の画面になるので完了するまで待機する。

完了後、迂闊に端末を再起動してしまうと初期化が始まってしまうので注意。
やっちゃった場合には、Stock ROM から再度焼き直すのが無難だろう。

### 4. Magisk の書き込み

ここまで来て Magisk を書き込めるようになる。

#### 4.2. TWRP の起動

電源ボタンを長押しして端末を終了し、
端末の画面が消えたら素早くダウンロードモードの構えに持ち帰る。
うまくいけば次のように TWRP が起動する。

ここでボーっとやっていると端末初期化が実行されてしまうので注意。

![TWRP startup](/images/photos/2019-03-21-twrp1.jpg)

水色のバーを右にスワイプすればメニュー画面に進む。

![TWRP menu](/images/photos/2019-03-21-twrp2.jpg)

#### 4.3. データフォーマット

まずはデータフォーマットをする。ここからは画面が多いので画像を省略する。

- Swipe を選択。
- 次の画面で Format Data を選択。
- 次の画面で確認されるので、yes を入力。

以下の画面が出てフォーマットが実行される。
`Unable to mount` 等いろいろエラーが出ているが大丈夫そうだったと記憶している。

![TWRP wipe](/images/photos/2019-03-21-twrp4.jpg)

- 戻るボタンを使ってメニュー画面に戻る。
- Reboot を選択。
- Recovery を選択。

端末が再起動され、また TWRP に戻ってくる。

#### 4.4. Magisk の転送

インストールするためにプログラムが端末内にある必要がある。
PC から ADB でファイルを転送する。

最終的な目的は Magisk をインストールすることなのだが、
端末のシステムディスクチェックを回避するためのプログラムもインストールする。
それらも一緒に端末へ送っておく。

効果の真偽は正直よくわからない。自己責任で入れること。
ただ無ければ端末の起動後に警告が出てきたりするようだ。

```sh
> adb.exe push "remove encryption.zip" /sdcard
> adb.exe push no-verity-opt-encrypt-6.0.zip /sdcard
> adb.exe push Magisk-v18.1.zip /sdcard
```

それぞれ ZIP のままでよい。

#### 4.5. Magisk の書き込み

端末で TWRP のメニュー画面から操作を開始する。

- nstall を選択。
- /sdcard から remove encryption.zip を選択。
- Swipe to confirm Flash に従い水色のバーを右にスライドする。

インストールが始まる。no-verify-opt-encrypt と Magisk にも同じ要領でフラッシュする。

以下は Magisk フラッシュ時の思い出:

![TWRP flash](/images/photos/2019-03-21-twrp5.jpg)

### 5. Magisk Manager のインストール

TWRP からでもいいし、電源キーからでもいいので、端末を普通に起動すると、端末の初期化が開始する。今回は英語版で起動した。微妙にうまくいっていないときは中国語で起動したりもしていた気がする。

![TWRP flash](/images/photos/2019-03-21-boot.jpg)

さて、XDA に書かれている手順によると、これで起動が完了すると Magisk アプリが表示されるとあるのだが、私の環境ではどうしてもうまくいかなかった。なぜかアイコンが出ない。
それが成功なのか失敗なのかはわからない。

とりあえず不安だったので、私はこのあと Magisk Manager を別途書き込んだ。

以下をスマートフォンの Web ブラウザで開き、Magisk Manager.zip をダウンロードしたあと、再びダウンロードモードの構えをとることで TWRP を起動する。

[Download Magisk Manager Latest Version 7\.0\.0 For Android 2019](https://magiskmanager.com/)

そして先程 Magisk 等をインストールした時と同じ手順で Magisk Manager をフラッシュする。また端末を起動すると、Magisk Manager アイコンがホームに表示されている。

Magisk Manager アプリを起動すると、Magisk のインストール等もできるが、必要そうならやればいいし、必要そうでなければ何もしなくてよいだろう。この辺は何が最低限の手順かよくわからなかった。ただ結局はうまくいったようだ。

うまくいっていれば例えば adb で接続して `su` すると root 権限を許可するか選ぶダイアログが表示される。そこで許可をして実際に root になれたならば、目的は達したことになる。

### 6. Synergy アプリへの root 権限許可

ここからは端末の root 権限取得とは関係無い。
だが私の目的は Synergy アプリを使えるようにすることだったので、ここでアプリを起動してroot 権限取得を許可してみる。

アプリは Google Play で公開されていないので、公式の GitHub より拾ってくる:

[symless/synergy\-android\-7: Synergy for Android client with support for android 7\+](https://github.com/symless/synergy-android-7)

`adb` を使ってインストール:

```sh
$ adb devices
List of devices attached
988633454855353147	device

$ adb install -g -t synergy.apk
```

スマートフォン端末上に青色と黄緑色の輪っかのアイコンが出現したら、それが Synergy アプリである。

Magisk の設定が成功していると、アプリケーションを起動したときに root 権限取得ダイアログが出てくる。
ここで許可を選べば以後、Synergy アプリは root 権限を利用することができるようになる。

![synergy android](/images/screenshots/2019-03-24-magiskmanager.png)

これで root 権限取得は完了したわけだが、Synergy アプリを利用してマウス、キーボードの共有をするにはもう一枚の障壁がある。SELinux である。

SELinux はシステム内のアクセス制御機能をする Linux 機能である。
ポリシーという単位でアプリケーションからシステムへのアクセスを制御することができ、
root 権限を取得したといても制御の対象である。いやむしろそれが主目的である。

というわけで続きは [SELinux編](https://kikei.github.io/smartphone/2019/03/22/synergy-selinux.html) 編で書く。

### 7. 参考

#### 各種ソフト入手元

Odin3:
[Download Odin 3\.13\.1 \- Samsung Odin download with ROM Flashing Tool](https://odindownload.com/)

Galaxy Stock ROM:
- [Firmwares \- SamMobile](https://www.sammobile.com/firmwares/)

Encryption Disabler:
- [Root for 8\.0 Oreo \- Post \#37](https://forum.xda-developers.com/showpost.php?p=76564735&postcount=37)

no-verify-opt-encrypt:
- [Index of /android\-tools/no\-verity\-opt\-encrypt/](https://build.nethunter.com/android-tools/no-verity-opt-encrypt/)

Magisk, Magisk Manager:
- [Download Magisk Manager Latest Version 7\.0\.0 For Android 2019](https://magiskmanager.com/)

JOdin3:
- [\[Release\] JOdin3 CASUAL Cross Platform and Web\-Based Flashing For Samsung Phones](https://forum.xda-developers.com/showthread.php?t=2598203)
- [Download Odin for Mac and Linux with JOdin3 \| ConsumingTech](https://consumingtech.com/download-odin-for-mac-and-linux-with-jodin3/)
[\[Utility\] Odin for Linux \!\!\! \(JOdin3 CASUAL\) \| Android Development and Hacking](https://forum.xda-developers.com/android/general/utility-odin-linux-jodin3-casual-t3777030)

#### 手順

基本的には以下の手順に従った:
- [Guide How to root Android 8\.0 Oreo Stock ROM… \| Samsung Galaxy S7 Edge](https://forum.xda-developers.com/s7-edge/how-to/guide-how-to-root-android-8-0-oreo-t3840271)

部分的に少し新しい手順を参考にしている:
- [How To decrypt and flash Magisk S7 Edge Oreo Stock Rom \- Post \#47](https://forum.xda-developers.com/showpost.php?p=77000849&postcount=47)

> 1. Go to Download mode; Open Odin;
> 2. Make sure to uncheck Auto Reboot
> 3. Flash via AP > twrp-3.2.x-x-hero2lte.img.tar
> 4. Wait for Odin to Pass and exit the Download Mode again with (Volume Down+Home+Power)
> 5. Then directly boot intro recovery (Volume Up+Home+Power)
> 6. Swipe for Modifications and Wipe Data, confirm with typing "yes" and reboot again into recovery
> 7. Install only the remove encryption.zip and reboot into your system
> 8. Setup your phone and after you did reboot again into the recovery
> 9 Install Magisk and reboot your phone; Thats all now your phone is running Magisk fully decrypted

#### 知識

- [What is a stock ROM? \- Quora](https://www.quora.com/What-is-a-stock-ROM)
- [\[Galaxy S\] PITファイルの中身 \| Gagdet is not Gadget\.](https://gagdet.wordpress.com/2011/02/12/galaxy-s-pit%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%81%AE%E4%B8%AD%E8%BA%AB/)
- [Magisk とは何か ― その機能と導入のメリット \| neggly\.org](https://neggly.org/post/what-is-magisk/)
