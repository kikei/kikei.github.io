---
layout: post
title: "Synergyクライアント起動 SELinux編"
categories: smartphone
---

この記事は Linux+Android で Synergy を使えるようになるまでの道程を記述した連載の第3編である。Synergy は PC と PC 間、あるいは PC とスマートフォン間でマウスやキーボードを共有できるようになる技術である。今回は PC (Synergyサーバー) に接続したマウスやキーボードで、Android スマートフォン (Synergy クライアント) を操作することを目指している。

Synergyはよくできた技術であるのだが、主要な対象環境は Windows-Windows や Windows-mac 等、PC 間での入力デバイス共有のようであるからか、Linux+Android みたいな少し変わった構成では苦労することがある。

### 1. これまでの記事

まず初めの記事で [Linux PC 上で Synergy サーバーを正常に起動](https://kikei.github.io/smartphone/2019/03/21/root-galaxys7.html) することに成功した。この記事完了時点で Synergy クライアントアプリから Synergy サーバーへ接続できる状態まで進めた。だがログ上接続が成功したように見えるだけで、スマートフォン上にマウスカーソルが表示されることはなかった。

次の記事で [Android スマートフォンの root 権限を取得](https://kikei.github.io/smartphone/2019/03/21/root-galaxys7.html) した。Android スマートフォンを Synergy クライアントにするためには、仕組み上 Android 端末の root 権限が必要である。この記事では `adb shell` でスマートフォンに接続し、`su` コマンドを実行するとスマートフォン画面上に確認ダイアログが出現し、許可することで root 権限が取得できるようになった。

今回の記事は第3番目の記事である。最後までいくとついに、スマートフォン上の画面上にマウスカーソルが表示されたり、PC に繋いだキーボードの入力が端末に反映されるようになる。

### 2. サーバー・クライアントの起動

これまでの記事と重複するが、前準備的なところを書く。

#### 2.1. Synergy サーバー起動

PC 側で Synergy サーバーを起動する。
私は Galaxy s7 スマートフォンの root 権限で苦労して時間が経過したため、サーバーの起動は久しぶりとなる。

下記コマンドを Linux PC で実行する。

```sh
$ synergy-core --server -c ~/synergy.conf -d DEBUG -l server.log
```

synergy.conf には今見たら次のように書いてあった:

```sh
cat ~/synergy.conf 
section: screens
	fg-galaxy:
		halfDuplexCapsLock = false
		halfDuplexNumLock = false
		halfDuplexScrollLock = false
		xtestIsXineramaUnaware = false
		switchCorners = none 
		switchCornerSize = 0
	localhost.localdomain:
		halfDuplexCapsLock = false
		halfDuplexNumLock = false
		halfDuplexScrollLock = false
		xtestIsXineramaUnaware = false
		switchCorners = none 
		switchCornerSize = 0
end

section: aliases
end

section: links
	fg-galaxy:
		right = localhost.localdomain
	localhost.localdomain:
		left = fg-galaxy
end

section: options
	relativeMouseMoves = true
	screenSaverSync = true
	win32KeepForeground = false
	clipboardSharing = true
	switchCorners = none 
	switchCornerSize = 0
end
```

Firewall の設定とか忘れないでね:

```sh
$ cat /lib/firewalld/services/synergy.xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>Synergy</short>
  <description>Synergy lets you easily share your mouse and keyboard between multiple computers, where each computer has its own display. No special hardware is required, all you need is a local area network. Synergy is supported on Windows, Mac OS X and Linux. Redirecting the mouse and keyboard is as simple as moving the mouse off the edge of your screen.</description>
  <port protocol="tcp" port="24800"/>
</service>
$ sudo firewall-cmd --permanent --add-service=synergy
$ sudo firewall-cmd --reload
```

#### 2.2. Synergy クライアントの起動

スマートフォン端末の root 権限を取得した時、以前使用していたアプリは全て消去されてしまったので、Synergy アプリを再度インストールする。

アプリは Google Play で公開されていないため、公式の GitHub からダウンロードしてくる。

[symless/synergy\-android\-7: Synergy for Android client with support for android 7\+](https://github.com/symless/synergy-android-7)


```sh
$ cd synergy-android-7
$ adb install -g -t ./synergy.apk
```

スマートフォン端末上に青色と黄緑色の輪っかのアイコンが出現したら、それが Synergy アプリである。

とりあえずアプリを起動する。

Magisk 等、root 権限の取得時に許可ダイアログが出てくるように設定している場合、
ここで確認ダイアログが表示される。root 権限の許可は必須である。
Magisk ならば以下の画像のように表示される:

![synergy android](/images/screenshots/2019-03-24-magiskmanager.png)

### 3. Synergy アプリの接続

アプリを起動すると、4つの入力覧が表示される (2019年3月22時点) ので、それぞれ例えば以下のように埋める。

- "Synergy Client name:" Synergy サーバー側で設定したクライアント名。先の synergy.conf でいけば "fg-galaxy" が正解である。
- "Server IP/Hostname:" Synergy サーバーの IP アドレスを入力しておけば安全。"192.168.0.8" とか。
- "Server Port:" デフォルトで 24800 。
- "Input device name:" 何でもいい適当な英数の文字列。強いていうなら 256 文字以下。デフォルトで "touchscreen" になっているのでそれでいい。

そして "Connect" ボタンを押す。

このときおそらく3種類の反応があるはずだ。

1. 何も反応しない。
2. 端末のバイブが振動するが、それだけで何も起きない。
3. マウスカーソルが表示される。


1. だったら、root 権限を取得できていない可能性が高い。
別のアプリを使ったり、`adb shell su` してみたりしてちゃんと root 権限を取得できるか試してみるとよい。


3. だったら、あなたの環境はとてもいい環境である。もしくはよくわからないけどすごく幸運に恵まれている。この記事を読み進める必要も多分無い。


2. だったら私と同じ状況かもしれない。
`adb logcat` してみるとうまくいってそうなログが出力されているかもしれないが、私の場合は次のような行があった。

```
03-05 03:35:45.389 18308 18308 F Synergy : failed to open uinput_fd == -1
03-05 03:35:45.389  3107  3107 E audit   : type=1400 audit(1551724545.382:18449): avc:  denied  { write } for  pid=18308 comm="org.synergy" name="uinput" dev="tmpfs" ino=13553 scontext=u:r:untrusted_app:s0:c512,c768 tcontext=u:object_r:uhid_device:s0 tclass=chr_file permissive=0 SEPF_SM-G930F_8.0.0_0018 audit_filtered
```

着目すべきは2行目の audit ログである。これは SELinux のログである。
Synergy アプリ (`org.synergy`) が `uinput` への書き込みを行おうとしたが SELinux のポリシーにより拒否されたことを現わしている。

具体的には、Synergy アプリは `untrusted_app` というロール、`uinput` には `uhid_device` というロールが設定されていて、`untrusted_app` から `uinput` に対する `chr_file` クラスの操作が SELinux に設定されたルールにより拒否された。そのことがログからわかる。

ちなみに1行目は自分で Synergy アプリを改造して仕込んだデバッグログであるから表示されなくて大丈夫。

以下のように書いてデバッグした:

synergy-android-7/app/src/main/jni/suinput.c

```c
 int suinput_open(const char* device_name, const struct input_id* id)
 {
   int original_errno = 0;
   int uinput_fd = -1;
   struct uinput_user_dev user_dev;
   int i;
   for (i = 0; i < UINPUT_FILEPATHS_COUNT; ++i) {
     uinput_fd = open(UINPUT_FILEPATHS[i], O_WRONLY);
     if (uinput_fd != -1)
       break;
   }
 
   if (uinput_fd == -1) {
+    __android_log_print (ANDROID_LOG_FATAL, DEBUG_TAG, "failed to open uinput_fd == -1");
     return -1;
   }
...略...
}
```

コードが出てきたので突然アプリの内部的な原理を語る。

### 4. Android 版 Synergy アプリの原理:

Synergy アプリは、Linux Input Subsystem を用いている。
Linux Input Subsystem は、Linux に多様な入力デバイスを接続し動作させるための機構である。Android でもその機能が生きており、Synergy アプリではその機能を利用している。

その方法は、`/dev/uinput` に仮想的なマウスデバイスを作成し、アプリからデバイスへイベントを流し込むことにより、マウスを操作する。ここでアプリはドライバとして動作する。

アプリは PC 側に接続した本物のマウス情報の操作を受信し、それを模倣するイベントを仮想デバイスへ書き込むため、まるで本物のマウスでスマートフォン端末を操作しているように見える、という構造である。キーボードについても然り。

### 5. Linux 制限の回避

SELinux による制限の回避方法として、私はルールの追加を行った。ルールが無いから作る、という解決法である。とても勇ましい。

より良い手法は Synergy アプリを `untrusted_app` でなくすることだ。
これについては調査中である。
ちゃんと署名した上で Synergy をシステムアプリとしてインストールできれば、多分うまくいく。

あと、SELinux? そんなもの無効にすればいいじゃん、と考えるかもしれない。
残念ながら Android では Shell の `root` からでは SELinux を無効化できない。

`setenforce 0` しても Enforcing に戻されてしまう:

```sh
$ su
# getenforce
Enforcing
# setenforce 0
# getenforce
Enforcing
```

どうやら `config_always_enforcing` という設定が施されているらしい:

```sh
$ adb dmesg | grep audit
03-25 03:13:42.214  3112  3112 E audit   : type=1404 audit(1553451222.203:306123): config_always_enforce - true; enforcing=1 old_enforcing=1 auid=4294967295 ses=4294967295
03-25 03:13:44.502  3098  3098 E audit   : avc:  received setenforce notice (enforcing=1)
```

さて困った、となってから数時間、`magiskpolicy` というコマンドを唐突に見つけてしまった。ヘルプを見ると、SELinux のポリシーを操作できそうな顔をしている:

```sh
# which magiskpolicy
/sbin/magiskpolicy
# magiskpolicy --help                                                                    
One policy statement should be treated as one parameter;
this means a full policy statement should be enclosed in quotes;
multiple policy statements can be provided in a single command

The statements has a format of "<rule_name> [args...]"
Multiple types and permissions can be grouped into collections
wrapped in curly brackets.
'*' represents a collection containing all valid matches.

Supported policy statements:

Type 1:
"<rule_name> source_type target_type class perm_set"
Rules: allow, deny, auditallow, dontaudit

Type 2:
"<rule_name> source_type target_type class operation xperm_set"
Rules: allowxperm, auditallowxperm, dontauditxperm
* The only supported operation is ioctl
* The only supported xperm_set format is range ([low-high])

Type 3:
"<rule_name> class"
Rules: create, permissive, enforcing

Type 4:
"attradd class attribute"

Type 5:
"<rule_name> source_type target_type class default_type"
Rules: type_transition, type_change, type_member

Type 6:
"name_transition source_type target_type class default_type object_name"

Notes:
* Type 4 - 6 does not support collections
* Object classes cannot be collections
* source_type and target_type can also be attributes

Example: allow { s1 s2 } { t1 t2 } class *
Will be expanded to:

allow s1 t1 class { all permissions }
allow s1 t2 class { all permissions }
allow s2 t1 class { all permissions }
allow s2 t2 class { all permissions }
```

というわけで、次のように実行してみた。
見た通りだが、`untrusted_app` から `uhid_device` に向けて、`chr_file` を許可した。

```sh
# magiskpolicy --live "allow untrusted_app uhid_device chr_file *"
```

セキュリティ的な観点で残念なのは承知である。
その辺は今後試行錯誤しながらより良い方法を見付け、ここに追記したい。

そして、うまくいってしまった。Magisk お前すごいな:

![Synergy works](/images/photos/2019-03-24-synergy.jpg)

スマートフォン端末の下部にある Micro USB ポートに何も差さっていないのに、
物理キーボードが差さっている時のメニューが表示されたり、マウスカーソルが表示されていることがわかる。

画面の下半分の空欄に表示された文字列は PC のキーボードから入力したものである。
さらに、そこに表示されたマウスカーソルは PC に接続したマウスで操作している。

### 6. 今後の展望

現在の方法では実用するには足りないところがいくつかある。

例えば SELinux の追加ポリシーはスマートフォンを再起動すると多分消えてしまう。
これでは不便だ。
そしてこの設定自体 SELinux の趣旨に反している。もうちょっと何とかならないのか、という声が聴こえてくるようだ。
magiskpolicy をもっと上手く使うとよさそう、とかアプリをシステムアプリとしてインストールすれば SELinux の制御自体要らなそう、等の可能性を試してみるつもりである。

あと Synergy にはクリップボード共有の機能もあるはずだが、今まだ動作させられていない。
クリップボードの共有が使えるととても便利になるのでこれも調査したい。

### 7. 参考

#### Linux Input Subsystem:

- [Using the Input Subsystem, Part II \| Linux Journal](https://www.linuxjournal.com/article/6429)
- [Android Input Subsystemを用いたハードイベントの記録と再現 \- Qiita](https://qiita.com/tanaka512/items/93d77295fc4ecaf2d4ec)
- [uinput \- おなかすいたWiki！](http://wiki.onakasuita.org/pukiwiki/?uinput)
- [Linux の入力デバイスをカスタマイズ \- Qiita](https://qiita.com/propella/items/a73fc333c95d14d06835)
- [linux uinput: simple example? \- Stack Overflow](https://stackoverflow.com/questions/26693280/linux-uinput-simple-example)

#### SELinux:

- [SELinuxがEnforcingのままシステムファイルを書き換えたい \- がじぇったほりっく](https://jagadgetaholic.blogspot.com/2017/03/selinux.html)
- [creams@nexus: Android 5\.1 Lollipop SELinux policy injection for system apps\.](http://creamsnexus.blogspot.com/2015/07/android-51-lollipop-selinux-policy.html)
- [Frida, Magisk and SELinux – serializethoughts](https://serializethoughts.com/2018/07/23/frida-magisk-and-selinux/)
- [Customizing SELinux  \|  Android Open Source Project](https://source.android.com/security/selinux/customize)
- [Android gives apps full access to your network activity\.](https://hackernoon.com/android-gives-apps-full-access-to-your-network-activity-f0fd92ee1824)
- [HowTos/SELinux \- CentOS Wiki](https://wiki.centos.org/HowTos/SELinux)

#### Magisk:

- [topjohnwu/magiskpolicy: sepolicy patching tool](https://github.com/topjohnwu/magiskpolicy)

#### Android:

- [Android 実機から apk を探して取得 \- clock\-up\-blog](https://blog.clock-up.jp/entry/2015/02/28/android-pull-apk)
