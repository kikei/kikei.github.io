---
layout: post
title:  "Linux NAS サーバが起動しなくなったので直した"
categories: linux
---

何が起きたのか。気が付くと NAS サーバが起動しなくなっていた。

確か日中までは使っていたはず。それから夕方になって酒を飲んでいたところ、サーバに接続できないことに気付いた。ハードウェア故障かとも思ったが、見てみると電源コードが抜けていただけだった。何かの弾みに外れてしまったのだろう。何だ良かったと思い電源を投入した。

ところが、一体どういうわけか起動しない。焦ってモニタを接続してみると "GRUB" と表示されるだけだ。そこから進まなくなってしまった。この NAS には 10 年分の色々なデータを記録している。取り出せなくなったら、まあなんだ。悲しい。

この記事では復旧の手順を記録する。なお呑んだ直後の頭では直せなかったので、一晩おいて冷静になってから作業した。

### 1. 環境

OS: Arch Linux

```console
$ uname -a
Linux fg-arch 6.5.3-arch1-1 #1 SMP PREEMPT_DYNAMIC Wed, 13 Sep 2023 08:37:40 +0000 x86_64 GNU/Linux
```

HDD:

* 3 TB × 1, 起動・ホームディレクトリ用
* 3 TB × 1, BTRFS RAID1
* 8 TB × 2, BTRFS RAID1
* 200 GB × 1, Docker ストレージ専用

Mother board: ASRock Z77 Extreme6
BTRFS: v6.5.1

### 2. カーネルファイルの復旧

いきなり結論を書くと `/boot` ディレクトリの中身が空になっていた。ここには本来、ブートローダが起動するカーネルの実行バイナリが置かれている。このファイルが無ければ当然、起動できない。

以下は調査と復旧の手順。

#### Live USB の作成

"GRUB" から進まないのだから、ブートローダや起動イメージに問題が起きていることは明らか。よって、とにかく HDD を覗いてみないことには始まらない。こういう時には Live USB を作成し、暫定的に別の OS を起動する。

私の手元にはメイン利用の Linux デスクトップ (Fedora Workstation) がある。この PC で作成する。

[USB インストールメディア - ArchWiki](https://wiki.archlinux.jp/index.php/USB_%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%83%A1%E3%83%87%E3%82%A3%E3%82%A2#.E5.9F.BA.E6.9C.AC.E7.9A.84.E3.81.AA.E3.82.B3.E3.83.9E.E3.83.B3.E3.83.89.E3.83.A9.E3.82.A4.E3.83.B3.E3.83.A6.E3.83.BC.E3.83.86.E3.82.A3.E3.83.AA.E3.83.86.E3.82.A3.E3.82.92.E4.BD.BF.E3.81.86) に従って ISO をダウンロード。十分な容量の USB メモリに書き込む。

書き込む時は `dd` …と思いきや `cat` や `cp`、`pv` を使うべきだと Wiki には記述してある。いずれも `dd` と同様のことができるし、より安全だからというのが理由のようだ。

素直に `cat` を使った。最後に `sync` して全部書き込んだことを保証する。

```
> cat archlinux-2023.09.01-x86_64.iso > /dev/sdb
> sync
```

これで Live USB は完成。

#### /boot を覗く

NAS サーバに USB メモリを差す。電源を投入して F11 を連打。表示されたメニューにて USB メモリを選ぶ。

![Boot menu](/images/photos/2023-09-17-bootdevices.jpg)

USB メモリの項目が2つ現われている。どちらを選んでも起動できるが、UEFI と書かれた側を選ばないと後のブートローダ書き込みの時に失敗するので注意。`UEFI: MF-AU2A` の方。

![Arch Linux](/images/photos/2023-09-17-archlinux.jpg)

Arch Linux の起動メニューを表示して…

![Boot with Live USB](/images/photos/2023-09-17-live.jpg)

起動した。この画面が出ただけで、洞窟で明りが灯ったような安心感がある。実際のところ、まだデータの無事はまだ判らないのだけども。

早速、挿さっている HDD を調べた。

```
# fdisk -l
(略)
Disk /dev/sda: 2.73 TiB 3000592982016 bytes, 58605331668 sectors
Disk model: TOSHIBA DT01ACA3
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: FE444121B8-8077-4616-9AB6-0B1FC263D9EF

Device         Start        End    Sectors  Size Type
/dev/sda1       2048    1050623    1048576  512M EFI System
/dev/sda2    1050624    5244927    4194304    2G Linux swap
/dev/sda3    5244928  256903167  251658240  120G Linux filesystem
/dev/sda5  256903168 5860533134 5603629967  2.6T Linux filesystem
(略)
```

EFI データが入っているのは `/dev/sda1` のようだ。
過去の作業 [Arch Linuxでrootパーティションのサイズ変更をした](https://kikei.github.io/linux/2019/09/24/partition.html) によると、root パーティションは `/dev/sda3` で、ここに `/boot` ディレクトリがある。

そこで root パーティションをマウントし `chroot` する。[arch-chroot - ArchWiki](https://wiki.archlinux.org/title/chroot#Using_arch-chroot) は素の `chroot` のラッパーだそうだ。

```console
# mkdir /mnt/arch
# mount -t auto /dev/sda3 /mnt/arch
# arch-chroot /mnt/arch
```

これで仮想的に元々のディスクを使い起動した状態を再現できた。

早速 `/boot` の中身を列挙すると…

```console
# ls /boot/
efi/ grub/
```

空の `efi/` ディレクトリと `grub/` しか無い。
OS によって若干の差分はあれど、ここに `vmlinuz-linux` と `initramfs` が必要だ。

#### Linux カーネルのインストール

`vmlinuz-linux` と `initramfs` を得るためにはどうするのか。`pacman` で `linux` をインストールすれば良いだけだった。凄い。

```console
# pacman -S linux
# ls /boot
efi  grub  initramfs-linux-fallback.img  initramfs-linux.img  vmlinuz-linux
```

#### ブートローダの再インストール

もしかしたら試行錯誤したためかもしれないが、ブートローダをインストールし直す手順が加えて必要だった。

初めに作業に必要なパッケージをインストールする。

```console
# pacman -S os-prober grub efibootmgr
```

ここからは [GRUB - ArchWiki](https://wiki.archlinux.jp/index.php/GRUB#.E3.82.A4.E3.83.B3.E3.82.B9.E3.83.88.E3.83.BC.E3.83.AB) に従い作業していく。

EFI システムパーティションをマウント。EFI システムパーティションは前項で見た通り `/dev/sda1` だ。マウントポイントのパスには `/boot/efi` 派と `/efi` 派があるようだが `/boot/efi` を選んだ。

```console
# mount -t auto /dev/sda1 /boot/efi
```

次に `os-prober` しておく。「インストールされている他のシステムを検索して自動的にメニューに追加する」場合に使うので今回の場合は不要だったかもしれなかった。実行してみても何も表示されない。エラーが出なければ問題無いということにした。

```console
# os-prober
```

`grub.cfg` を生成する。おなじみ GRUB が起動する時に読む設定ファイルだ。

```
# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  initramfs-linux-fallback.img
Warning: os-prober will not be executed to detect other bootable partitions.
IIts output will be used to detect bootable binaries on them and create new boot entries.
Adding boot menu entry for UEFI Firmware Settings ...
done
```

前項でインストールした `vmlinuz-linux` と `initramfs-linux.img`、`initramfs-linux-fallback.img` を見つけてくれたことがわかる。

それらのファイルが無い場合には Found... の行が表示されなかった。

Live USB からの起動時のブートデバイスメニューの画面にて「UEFI と書かれた側を選ばないと後のブートローダ書き込みの時に失敗する」と書いた。選ばなかった場合には、ここで `EFI variables are not supported on this system` というエラーになった。

最後にブートローダを書き込む。Web を検索すると色々なオプション指定が見つかるが、先の ArchWiki の方法を採用し上手くいった。

```console
# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
Installing for x86_64-efi platform.
Installation finished. No error reported.
```

うまくいったようなので `chroot` から抜け再起動。USB メモリは良い感じのタイミングで抜いておく。

```console
# exit
# reboot
```

#### やったー!!

![Grub menu](/images/photos/2023-09-17-grub.jpg)
![Booting](/images/photos/2023-09-17-booting.jpg)

絵は地味だが。

### 3. 故障ディスクの除去

しかし二回戦があった。Emergency mode で起動してしまう。デバイスエラーのようだ。

```
Timed out waiting for device /dev/disk/by-uuid/80ce0f6d-71f2-458f-895a-21fd49943f55
```

実のところ気付いていた。先程までの `os-prober` でもエラーを出していたし、今は HDD 1台がカチカチ賑やかに鳴っているからだ。特に何かやったつもりは無いのだが、ハードウェア作業にとどめを差したのだろう。音から該当の HDD を判別して SATA ケーブルを抜き取った。

![Western Digital WD2000](/images/photos/2023-09-17-wd2000.jpg)

WDC WD2000JD。遥か昔、実家で使っていたものを引き取った。2004 年と書いてある。20 年間回り続けたということになる。天晴れではないか。

ストレージ容量はたったの 200 GB しかない。今なら未使用でも 1,000 円あれば買えそうだ。

`fdisk` 上の表示はこうなっている。

```
Disk /dev/sdc: 186.32 GiB, 200049647616 bytes, 390721968 sectors
Disk model: WDC WD2000JD-19H
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: BA150377-B1F7-44F6-9978-C28ED7A053F3

Device     Start       End   Sectors   Size Type
/dev/sdc1   2048 390721934 390719887 186.3G Linux filesystem
```

古いけれど、せっかくあってまだ使えるので Docker ストレージ用に使用していたのだ。

```console
# grep 80ce0f6d /etc/fstab
UUID=80ce0f6d-71f2-458f-895a-21fd49943f55     /var/lib/docker ext5            rw,relatime,data=ordered       0 0
```

代わりに `/dev/sdd` を充てることにした。`/dev/sdd` は既に BTRFS の RAID に組み込んでしまっている。しかしバランシングの結果、BTRFS から戦力外扱いされたのか何も書き込まれていない。このまま組んだままにしても意味が無いので外してしまうことにした。

```
Disk /dev/sdd: 2.73 TiB, 3000592982016 bytes, 5860533168 sectors
Disk model: WDC WD30EZRX-00D
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: BAF87B85-75E9-4293-8792-93504442972A

Device     Start        End    Sectors  Size Type
/dev/sdd1   2048 5860533134 5860531087  2.7T Linux filesystem
```

実のところ、このデバイスはもうじき故障する。S.M.A.R.T. のエラーを頻発するようになり久しい。どうせ Docker ストレージのデータは使い捨てなので、ここで余命を使い切ってしまおう。

BTRFS の RAID から `/dev/sdd1` を除去する。

```
btrfs device remove /dev/sdd1 /house/library
```

EXT4 でフォーマットし直す。

```
mkfs.ext4 /dev/sdd1
```

`/etc/fstab` から旧デバイスの設定を外し、新デバイスを追加する。

```
# vim /etc/fstab
# grep /var/lib/docker /etc/fstab
UUID=b291bc8c-7aec-4f51-9c40-97a22eab003c       /var/lib/docker ext4            rw,relatime,data=ordered        0 0
```

systemd では `/etc/fstab` から生成した systemd ユニットファイルを使用する。

```
# systemctl daemon-reload
```

マウントできたら、再起動する。

```
# mount /var/lib/docker
# reboot
```

これで無事に起動できた。解決。

### 4. まとめ

そもそもなぜカーネルイメージだけピンポイントに消えてしまったのだろうか。何かやらかしたか。

ただピンポイントなおかげで復旧も簡単にできたのは救いだった。
