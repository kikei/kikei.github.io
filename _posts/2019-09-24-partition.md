---
layout: post
title:  "Arch Linuxでrootパーティションのサイズ変更をした"
categories: linux
---

前回 [Btrfs NASのHDDが壊れたので交換した](https://kikei.github.io/linux/2019/09/22/btrfs.html) の記事にて Btrfs に組み込んでいる HDD を果敢に交換したわけだが、
実は別の目的でメンテナンスしようとしていた。より深刻な問題に気付いてしまったので緊急で HDD 交換を行ったわけだ。

元々の目的は root パーティション拡張作業を行うことだった。
パーティションの容量が一杯になってしまったため、Pacman でパッケージの更新が行えない状態になっている。今回はもともとやりたかった作業、root パーティションのサイズ変更を行った。

なお root パーティションのファイルシステムは ext4 なので Btrfs は関係無い。

念のため書いておくが、この手順通りに作業しても必ずうまくいくという保証は無い。また、失敗したとしても私は一切の責任を負うことができない。

### 1. サーバー構成

Arch Linux:

```console
# uname -a
Linux fg-arch 5.2.8-arch1-1-ARCH #1 SMP PREEMPT Fri Aug 9 21:36:07 UTC 2019 x86_64 GNU/Linux
```

ディスク一覧:

以下のように root パーティションの容量が残り 2.1 GB になった。
このため `pacman -Syu` でパッケージを更新しようとしてもエラーで止まってしまう。

```console
# df -Th | grep sda
/dev/sda3      ext4       20G   17G  2.1G  90% /
/dev/sda1      vfat      511M   45M  467M   9% /boot
/dev/sda5      btrfs     5.4T  2.7T  2.8T  49% /house/entrance
/dev/sda5      btrfs     5.4T  2.7T  2.8T  49% /house/living
/dev/sda5      btrfs     5.4T  2.7T  2.8T  49% /house/library
/dev/sda4      ext4       98G  3.8G   90G   5% /home
```

/boot、/home、root パーティションはいずれも同じ /dev/sda 上に作成されている。
/dev/sda5 は今回関係無いので見る必要は無い。

ディスク詳細:

```console
# fdisk -l /dev/sda
Disk /dev/sda: 2.75 TiB, 3000592982016 bytes, 5860533168 sectors
Disk model: TOSHIBA DT01ACA3
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: FE3121B8-8077-4616-9AB6-0B1FC263D9EF

Device         Start        End    Sectors  Size Type
/dev/sda1       2048    1050623    1048576  512M EFI System
/dev/sda2    1050624    5244927    4194304    2G Linux swap
/dev/sda3    5244928   47187967   41943040   20G Linux filesystem
/dev/sda4   47187968  256903167  209715200  100G Linux filesystem
/dev/sda5  256903168 5860533134 5603629967  2.6T Linux filesystem
```

見てみると sda4 の容量が 100GB あって、まだ 90GB も空いている。
ちょうど sda3 の直後にあり実に都合が良い。
そこで sda4 を削除し、sda3 に結合してしまうことにした。

実際の作業手順は、

- sda3 と sda4 のパーティションを削除し、
- sda3 のパーティションのサイズを変更して、
- sda3 のファイルシステムをパーティションに合わせる

の流れでいく。

### 2. レスキューモード起動

パーティションのサイズを変更するためには、対象のディスクのマウントを解除する必要がある。普通に OS を起動したのでは自動的にマウントされてしまうし、起動中に解除もできない。そこでUSBメモリに Live OS を入れ、それをセーフモードで起動する。

今回はちょうど別の PC のセットアップ時に用意した Fedora Workstation のUSBメモリがあったのでそれを利用した。

[第3世代Ryzen CPUで新しいPCを組んだ](https://kikei.github.io/linux/2019/09/14/ryzen.html#21-fedora-%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB)

USB メモリから起動し、GRUBの画面で e とかを入力すると起動オプションを編集できる。
レスキューモードで起動するには、`ro rhgb quiet` とかなっているところの最後に ` rescue` を付け加えればよい。この辺りは記録をとらなかったけど死ぬところじゃないし、適当にやればいいと思う。

### 3. パーティション変更

レスキューモードで起動できたら、パーティションテーブルの編集を行う。

ここが今回の作業の肝となる。
しかし失敗しても即死ではなく、冷静にパーティションテーブルを元通りに戻せば復旧できる可能性がある。
そのためにも元の全パーティションの Start, End セクタをちゃんと記録しておくべきである。

もちろん、データのバックアップは必ずとっておかなければならない。

```console
# fdisk /dev/sda
```

`p` でパーティションテーブルの情報を表示できる。

![fdisk](/images/photos/2019-09-24-fdisk1.jpg)

次に sda3, sda4 を削除する。ここは写真が残っていないが、`d` を使う。
sda3 は一度削除してから、所望のサイズで作り直す。

sda3 の削除:

```console
Command (m for help): d
Partition number (1-5): 3
```

sda4 の削除:

```console
Command (m for help): d
Partition number (1-5): 4
```

そして sda3 を作成する。無駄なくディスク容量を使うため、
Start は元の sda3 の Start と同じにして、End は元の sda4 の End と同じにした。

パーティションタイプは普通に `Linux filesystem` にしておけばよい。

```console
Command (m for help): n
Partition number (3,4,6-128, default 3): 3
First sector (5244928-256903167, default 5244928): 5244928
Last sector, +/-sectors or +/-size{K,M,G,T,P} (5244928-256903167, default 256903167): 256903167

Created a new partition of 3 of type 'Linux filesystem' and of size 120 GiB.
Partition #3 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The sigunature will be removed by a write command.
```

![fdisk](/images/photos/2019-09-24-fdisk2.jpg)

最後に `w` で書き込む。

```console
Command (m for help): w
```

```console
$ sudo fdisk -l /dev/sda
Disk /dev/sda: 2.75 TiB, 3000592982016 bytes, 5860533168 sectors
Disk model: TOSHIBA DT01ACA3
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: FE3121B8-8077-4616-9AB6-0B1FC263D9EF

Device         Start        End    Sectors  Size Type
/dev/sda1       2048    1050623    1048576  512M EFI System
/dev/sda2    1050624    5244927    4194304    2G Linux swap
/dev/sda3    5244928  256903167  251658240  120G Linux filesystem
/dev/sda5  256903168 5860533134 5603629967  2.6T Linux filesystem
```

### 4. ファイルシステムのサイズ変更

パーティションのサイズ変更は完了したが、ファイルシステムのサイズも合わせる必要がある。また、その前にファイルシステムのチェックをしておくことになっている。

ファイルシステムのチェックには `e2fsck` コマンドを使う。

```console
# e2fsck -f /dev/sda3
```

エラー部分がある場合、修復するか訊かれる。私の場合はかなりたくさんのエラーが検出されたが、男前に全部修復させておいた。それが適切な行動だったかどうかはわからない。

ファイルシステムの拡大は `resize2fs` コマンドを使う。これはファイルシステムをパーティションに合わせて拡大・縮小してくれる。

```console
# resize2fs /dev/sda3
```

### 5. 起動パーティション設定の変更

この手順は検索した中に出てきていなかったが、私の Arch Linux では必要だった。

理由は起動パーティションの設定が PARTUUID で指定されていたからである。
PARTUUID はパーティションに割り当てられた UUID である。
先程の作業で sda3 を削除して作り直したので、PARTUUID が変わってしまった。

Arch Linux の起動に失敗すると、以下のように表示されて何もできなくなった。

```console
ERROR: device 'PARTUUID=a6d1d780-81c3-4ba9-acf6-5d7ab28acc24' not found. Skipping fsck.
:: mounting 'PARTUUID=a6d1d780-81c3-4ba9-acf6-5d7ab28acc24' on real root
mount: /new_root: can't find PARTUUID=a6d1d780-81c3-4ba9-acf6-5d7ab28acc24.
You are now begin dropped into an emergency shell.
sh: can't access tty: job control turned off
```

![Device not found](/images/photos/2019-09-24-devicenotfound.jpg)

ビビるが、よく読むと何のことはない。該当する PARTUUID のパーティションが見付からないと言っているだけだ。

再度レスキューモードで起動し、まず /dev/sda3 をマウントする。
それから、/boot/loader/entries/arch.conf を修正してあげる。

```console
# mount /dev/sda3 /mnt
# cat /mnt/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=PARTUUID=a6d1d780-81c3-4ba9-acf6-5d7ab28acc24 rw
```

![partuuid](/images/photos/2019-09-24-partuuid.jpg)

上記のように、先程エラーになった PARTUUID が options root に記述されている。

これを実態に合わせて訂正した。

```console
# ls -l /dev/disk/by-partuuid | grep sda3
lrwxrwxrwx 1 root root 10 Sep 24 19:17 05ac717a-2bfb-3c4b-a937-f7fdb034ab11 -> ../../sda3
# vi /mnt/loader/entries/arch.conf
# cat /mnt/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=PARTUUID=05ac717a-2bfb-3c4b-a937-f7fdb034ab11 rw
```

ここまでやって、root パーティションのサイズを拡大し、無事に起動することに成功した。

```console
# df -Th | grep sda
/dev/sda3      ext4      118G   17G   92G  15% /
/dev/sda1      vfat      511M   45M  467M   9% /boot
/dev/sda5      btrfs     5.4T  2.7T  2.8T  49% /house/entrance
/dev/sda5      btrfs     5.4T  2.7T  2.8T  49% /house/living
/dev/sda5      btrfs     5.4T  2.7T  2.8T  49% /house/library
```

その後 `pacman -Syu` が実行できるようになり、目的を達成した。

### 6. まとめ

壊れなくてよかったです。

### 7. 参考ページ

- [fdiskでDisk容量を拡張する \- Qiita](https://qiita.com/2or3/items/501f206a6091a4ce895f)
- [How To Resize ext3 Partitions Without Losing Data](https://www.howtoforge.com/linux_resizing_ext3_partitions)
- [時羽金也の技術帳: resize2fs と fdisk で ext4 のパーティションを縮小する](http://tokikane-tec.blogspot.com/2015/04/resize2fs-fdisk-ext4.html)
