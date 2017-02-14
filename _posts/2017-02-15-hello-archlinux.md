---
layout: post
title:  "OpenIndiana+ZFSからArchLinux+BTRFSに移行した"
categories: server
---

3年前に OpenIndiana を使って自宅サーバを構築したが、
今回、ArchLinux に移行することにした。

OpenIndiana を選んだのは、ZFS と KVM の両方が使えるからだったと思う。

ただ、3年運用してみて、Solaris系は更新頻度が少なくていまいち燃えなかったのと、
ドキュメントが少なくて作業がいちいち辛かったので、飽きてきた。

今回、残ディスク容量も全体の 50% を切ったし、
BTRFS も実用的になってきたらしく使ってみたいし、
あとグラフィックボードを増築することにしたので、
思い切ってOSも入れ替えてみることにした。

ローリングリリースというのにも興味があったので、移行先としてArchLinuxを選んだ。

### ディスクの確認 (OpenIndiana)

作業前に、各ディスクの仕様状態を確認した。

ここでちゃんとやっておかないと、大事に蓄えてきたデータが飛ぶので慎重に。

認識しているディスクを確認。

```
root@fg-hyper:~# format
Searching for disks...done

c4t0d0: configured with capacity of 2794.52GB
c4t1d0: configured with capacity of 2794.52GB


AVAILABLE DISK SELECTIONS:
       0. c4t0d0 <ATA-TOSHIBA DT01ACA3-ABB0-2.73TB>
          /pci@0,0/pci1849,1e02@1f,2/disk@0,0
       1. c4t1d0 <ATA-ST3000DM008-2DM1-CC26-2.73TB>
          /pci@0,0/pci1849,1e02@1f,2/disk@1,0
       2. c4t2d0 <ATA-WDCWD2000JD-19H-2D08 cyl 24318 alt 2 hd 255 sec 63>
          /pci@0,0/pci1849,1e02@1f,2/disk@2,0
       3. c4t3d0 <ATA-ST3000DM001-1CH1-CC27-2.73TB>
          /pci@0,0/pci1849,1e02@1f,2/disk@3,0
       4. c4t5d0 <ATA-WDC WD30EZRX-00D-0A80-2.73TB>
          /pci@0,0/pci1849,1e02@1f,2/disk@5,0
```

`c4t0d0` と `c4t1d0` が新しく追加したHDD。

`c4t0d0` の東芝 3TB は秋葉原のツクモで買った。
1万円いかなかったのでつくもたんクリアファイルがもらえなかったのが残念。

`c4t1d0` のSeagate 3TB は秋葉原のソフマップで買った。

`c4t3d0` と `c4t5d0` が ZFS で RAID1 にして運用しきたHDD。

Zpool は以下のようにして確認。

```
root@fg-hyper:~# zpool list -v
NAME         SIZE  ALLOC   FREE  EXPANDSZ    CAP  DEDUP  HEALTH  ALTROOT
house       2.72T  1.55T  1.17T         -    57%  1.00x  ONLINE  -
  mirror    2.72T  1.55T  1.17T         -
    c4t3d0      -      -      -         -
    c4t5d0      -      -      -         -
rpool        186G  8.53G   177G         -     4%  1.00x  ONLINE  -
  c4t2d0s0   186G  8.53G   177G         -
```

ディスクの利用率が 50 % を超えたので、HDD増築しよう、っていうわけ。

せっかくなのでヘルスチェックでもしておく。

```
root@fg-hyper:~# zpool status -v house
  pool: house
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        house       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            c4t3d0  ONLINE       0     0     0
            c4t5d0  ONLINE       0     0     0

errors: No known data errors
root@fg-hyper:~# zpool status -v rpool
  pool: rpool
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        rpool       ONLINE       0     0     0
          c4t2d0s0  ONLINE       0     0     0

errors: No known data errors
```

問題なさそう。

### スナップショットの作成 (OpenIndiana)

念のため、スナップショットをとっておく。

```
zfs list -t snapshot
zfs snapshot house/library@snapshot-20170212
zfs snapshot house/living@snapshot-20170212
```

OpenIndiana はここまで。

### ArchLinuxのインストール

公式ページを参考にした。

https://wiki.archlinuxjp.org/index.php/インストールガイド

UEFIブートのところだけよくわからなかったので、下記ページも参考になった。

http://note.kurodigi.com/archlinux-uefi-install/

`/dev/sda` に ArchLinux をインストールする。

```
# fdisk -l

# gdisk

Command (? for help):n
Permission number: 1
First sector     : 2048
Last sector      : +512M
Hex code or GUID : EF00

| Number | Start (sector) | End (sector) | Size      | Code | Name       |
|:------:|---------------:|-------------:|----------:|:-----|:-----------|
| 1      | 2048           | 1050623      | 512.0 MiB | EF00 | EFI System |
| 2      | 1050624        | 5244927      | 2.0 GiB   | 8200 | Linux swap |
| 3      | 5244928        | 47187967     | 20.0 GiB  | 8300 | Linux filesystem |
| 4      | 47187968       | 256903167    | 100.0 GiB | 8300 | Linux filesystem |
```

`1` が `EFI System` をするのがポイント。

それぞれファイルシステムの作成。

ESP (EFI System Partition)

```
# mkfs.vfat -F32 /dev/sda1
```

スワップ

```
# mkswap /dev/sda2
# swapon /dev/sda2
```

その他。

```
# mkfs.ext4 /dev/sda3
# mkfs.ext4 /dev/sda4
```

今回、`/` と `/boot` 、 `/home` でパーティションを分けてみた。

```
# mkdir boot
# mkdir home

# mount /dev/sda3 /mnt
# mount /dev/sda1 /mnt/boot
# mount /dev/sda4 /mnt/home
```

ネットワークの確認

```
# ping archlinuxjp.org
```

システム時計の設定

```
# timedatectl set-ntp true
# timedatectl status
```

パッケージの設定

```
# vi /etc/pacman.d/mirrorlist
# pacstrap /mnt base base-devel
```

`fstab`の作成

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

ここまで来て `chroot`。

```
# arch-chroot /mnt
```

タイムゾーンの設定。

```
# rm /etc/localtime
# ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

# hwclock --systohc --utc
# date
```

ロケールの設定。

```
# vi /etc/locale.gen
# locale-gen

# echo LANG=en_US.UTF-8 > /etc/locale.conf
```

ホスト名。

```
# echo fg-arch > /etc/hostname

# vi /etc/hosts
127.0.0.1	localhost.localdomain	localhost
::1		localhost.localdomain	localhost
127.0.1.1	fg-arch.localdomain	fg-arch
```

ネットワーク設定の再確認。

```
# lspci -v
```

```
# dmesg | grep tg3
```

`tg3` が `enp4s0`。

```
# dmesg | grep e1000e
```

`e1000e` が `enp9s0`。

パスワード設定。

```
# passwd
```

ブートプログラムのインストール。

```
# bootctl --path=/boot install
```

ブートローダの設定。
`arch.conf` を見にいくようにした。

```
# vi /boot/loader/loader.conf

default  arch
timeout  4
editor  0
```

`/` をマウントするパーティションの UUID 確認。

```
# blkid | grep /dev/sda3
```

`arch.conf` を書く。

```
# vi /boot/loader/entries/arch.conf

title          Arch Linux
linux          /vmlinuz-linux
initrd         /initramfs-linux.img
options        root=PARTUUID=a6d1d780-81c3-4ba9-acf6-5d7ab28acc24 rw
```

あとは再起動。

```
# reboot
```

起動後、ネットワークに接続されていなかったので、
とりあえず DHCP を実行してみたら、接続された。

```
# systemctl enable dhcpcd
```

`ifconfig` が無いとやっぱり困るのでインストール。

```
# pacman -S net-tools
```

いつまでも `tty` でやっていても不便なので、SSHのインストール。

```
pacman -S openssh
vi /etc/ssh/sshd_config
```

`root` で SSH ログインできないようにするが、
ユーザがいないので作成するが、
`zsh` を使いたいのでインストール。

```
pacman -S zsh
```

で、ユーザ追加。

```
groupadd fujii
useradd -g fujii -G wheel -m -s /bin/zsh fujii
su - fujii
systemctl enable sshd.service
```

あと `wheel` グループは `sudo` が使えるようにした。

```
visudo
```

SSH接続するのに、IPアドレスが変ったら不便なので固定。
その前にDHCPを無効化しておく。短い付き合いだったね。

```
systemctl stop dhcpcd
systemctl disable dhcpcd
```

`enp8s0` のIPアドレスを固定する設定。

```
vi /etc/netctl/enp8s0_profile

Description='Intel Corporation 82574L Gigabit Network Connection'
Interface=enp8s0
Connection=ethernet
IP=static
Address=('192.168.0.100/24')
#Routes=('192.168.0.0/24 via 192.168.1.2')
Gateway='192.168.0.1'
DNS=('192.168.0.1')

## For IPv6 autoconfiguration
#IP6=stateless

## For IPv6 static address configuration
#IP6=static
#Address6=('1234:5678:9abc:def::1/64' '1234:3456::123/96')
#Routes6=('abcd::1234')
#Gateway6='1234:0:123::abcd'
```

設定を反映させて `netctl` を実行。

```
netctl start enp8so_profile
netctl enable enp8so_profile
```

`enp8s0` のIPアドレスが変わっていない場合は、何も考えずに `reboot`。


### ZFSのインストール

ZFSからデータを取り出すので、当然 ZFS が使えないといけない。

以下のコマンドは、ArchでZFSを有効化する一般的な方法だが、
今回はうまくいかなかった。

kernel バージョンが 2.9.8 であるのに対し、
zfs-linux-git は kernel バージョン 2.9.7 に依存する。

いろいろ試してみたが、力及ばず `pacman` 経由ではインストールできず。


```
# vi /etc/pacman.conf
[archzfs]
Server = http://archzfs.com/$repo/$arch

# pacman-key -r 5E1ABF240EE7A126
# pacman-key --lsign-key 5E1ABF240EE7A126

# pacman -Sd zfs-linux-git
```

ということで、ソースからビルドすることにした。

zfs のインストールには、先に spl をインストールする必要がある。

GitHubからソースを引いてきて、適当にビルドしていたらうまくいった。

```
git clone https://github.com/zfsonlinux/spl.git
cd spl
./autogen.sh
make
make install
```

zfs のソースもGitHubから引いてきて、適当にビルド。

```
git clone https://github.com/zfsonlinux/zfs.git
cd zfs
./autogen.sh
make
make install
```

ところどころ引っかかったが、とりあえず `make` たたいてたらなんとかなった気がする。

最終的に、実行ファイルとカーネルオブジェクトファイルが生成されて
所定の位置に収まっていれば問題ない。

```
[root@fg-arch house]# which zpool
/usr/local/sbin/zpool
[root@fg-arch house]# which zfs
/usr/local/sbin/zfs
[root@fg-arch house]# ls /lib/modules/4.9.8-1-ARCH/extra/zfs/zfs.ko.gz
/lib/modules/4.9.8-1-ARCH/extra/zfs/zfs.ko.gz
```

あとはモジュールをロードする。

```
depmod -a
modprobe zfs
```

これでZFSが利用可能になった。

### ZFSのインポート

とりあえず pool を一覧表示してみたところ、OpenIndiana で見たことがある内容が出た。

```
[root@fg-arch house]# zpool list -v
NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
house  2.72T  1.55T  1.17T         -      -    57%  1.00x  ONLINE  -
  mirror  2.72T  1.55T  1.17T         -      -    57%
    sdd      -      -      -         -      -      -
    sde      -      -      -         -      -      -
```

インポートしてみた。

```
zpool import house
```

ディスクの一覧を表示してみた。

```
# mount
house/library on /house/library type zfs (rw,xattr,noacl)
house/living on /house/living type zfs (rw,xattr,noacl)
```


### BTRFSへのコピー

まずはコピー先のパーティションを作成する。

```
gdisk /dev/sda
gdisk /dev/sdb
```

`/dev/sda5` と `/dev/sdb1` を作成した。

作成しただけでは、利用可能にならなかった。

更新のために、`partprobe` した。

```
pacman -Su parted
partprobe
```

作成した2つのパーティションを使い、RAID1 のディスクを作成。

```
pacman -Su btrfs-progs

mkfs.btrfs -L house -d raid1 /dev/sda5 /dev/sdb1
```

作成結果を確認。

```
[root@fg-arch house]# btrfs filesystem show
Label: 'house'  uuid: 962205ef-673e-4b93-be7e-58bd23d68095
	Total devices 2 FS bytes used 0TiB
	devid    1 size 2.61TiB used 0TiB path /dev/sda5
	devid    2 size 2.73TiB used 0TiB path /dev/sdb1
```

マウントする。上記 `house` を構成するデバイスのいずれかを指定したらできた。

```
mount /dev/sdb1 /mnt
```

元々のファイル構造をなぞるようにして、サブボリュームを作成。

サブボリューム単位でスナップショットがとれるらしい。

とりあえずおためしで `music` サブボリュームを作った。

```
btrfs subvolume create /mnt/library
btrfs subvolume create /mnt/library/music
```

`rsync` を使ってデータをコピーする。

```
pacman -Su rsync

rsync --daemon
rsync -av /house/library/music/ /mnt/library/music/
```

他のデータもコピー。

めんどくさいのでやっつけスクリプトを書いた。

```
#/bin/sh

dosync() {
  if [ $(btrfs subvolume list /mnt/library | grep $1 | wc -l) == 0 ]; then
    btrfs subvolume create /mnt/library/$1
  fi
  rsync -av /house/library/$1/ /mnt/library/$1/
}

dosync anime
dosync backup
dosync books
dosync entrance
dosync etc
dosync game
dosync graphics
dosync mail
dosync movie
dosync music
dosync photos
...
```

`rsync` が出したサマリをまとめてみた。

```
total size is 1,615,241,547,964 speedup is 2,026,951.98
total size is   354,722,611,858	speedup is 401,060.77
total size is   253,324,311,594	speedup is 287,710.30
total size is    51,492,765,748	speedup is 225,604.03
total size is    47,324,780,298	speedup is 2,246.76
total size is    41,739,921,373	speedup is 447,795.58
total size is    35,350,848,244	speedup is 11,630.88
total size is    23,757,758,236	speedup is 889,503.85
total size is    17,470,684,367	speedup is 488,389.92
total size is    11,823,088,373	speedup is 697,733.16
total size is     6,333,702,077	speedup is 68,640.90
total size is     5,536,714,037	speedup is 2,827.22
total size is     2,221,411,428	speedup is 458,969.30
total size is       369,602,921	speedup is 3,409.46
total size is       366,503,342	speedup is 15,655.18
total size is       314,958,330	speedup is 68,588.49
total size is        25,356,516	speedup is 162,541.77
```

合計したら `2,467,416,566,706 bytes`。
すなわち、 `2.47T` になった。ちゃんとコピーできた雰囲気がある。

```
[root@fg-arch house]# btrfs filesystem show
Label: 'house'  uuid: 962205ef-673e-4b93-be7e-58bd23d68095
	Total devices 2 FS bytes used 2.28TiB
	devid    1 size 2.61TiB used 2.28TiB path /dev/sda5
	devid    2 size 2.73TiB used 2.28TiB path /dev/sdb1
[root@fg-arch house]# df --block-size=1G
ファイルシス   1G-ブロック  使用 使用可 使用% マウント位置
dev                      4     0      4    0% /dev
run                      4     1      4    1% /run
/dev/sda3               20     2     17   11% /
tmpfs                    4     0      4    0% /dev/shm
tmpfs                    4     0      4    0% /sys/fs/cgroup
tmpfs                    4     0      4    0% /tmp
/dev/sda1                1     1      1    7% /boot
/dev/sda4               98     1     93    1% /home
tmpfs                    1     0      1    0% /run/user/1000
/dev/sda5             2734  2338    334   88% /mnt
house/library         2648  1541   1107   59% /house/library
house/living          1131    25   1107    3% /house/living
```

うーん、全体のデータサイズは大きくなったかな？

今回はこれらの数字の違いについては気にしないことにする。

```

`house/living/` もコピー。

```
btrfs subvolume create /mnt/living
rsync -av /house/living/ /mnt/living/
```

おわり。

### 今後

しばらくこのままで使って、新しいデバイスの様子を見る。

こなれてきたところで、ZFSになっている旧デバイス 3T×2 のデータを削除し、
BTRFSに加える。RAIDを組んでいるディスクにあとからパーティション追加が可能らしい。

### 感想

Linux はツール類も揃っているし、自分も慣れているのでやりやすかった。
