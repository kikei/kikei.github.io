---
layout: post
title:  "Docker+ownCloudのデータ保存パスを引っ越しした"
categories: server
---

Docker 用のディレクトリを専用ディスク上に引っ越ししてみた。

### これまでのあらすじ

前回の記事で、 [Docker Compose を使い ownCloud を構築した](https://kikei.github.io/server/2017/02/25/docker-owncloud.html)。

この記事ではデータボリュームコンテナを使い、
ポータブルな ownCloud 環境をつくりあげることに成功した。

さらに、ホストPCにマウントしてある巨大なストレージ `/house` を用意し、
外部ストレージとして ownCloud から利用できるようにした。
外部ストレージがコンテナから見えるように、Docker のボリューム機能を使った。

で、この環境はおおむね期待通りに動いたが、一つの大きな問題により、
あまり役に立っていなかった。

### 問題点と方針

Docker 本体のデータが保存されたディスクが容量一杯になると、
せっかく潤沢な容量を持つ外部ストレージがあっても、
そこへファイルをアップをアップロードできない。

Docker のデータボリュームコンテナは自動的に、 `/var/lib/docker` 配下に作成される。
Docker 上で動く ownCloud 本体の作業が行われるディレクトリも同じ場所である。

だから、ownCloud にファイルをアップロードするときには、
このディレクトリが属するディスクの残容量のチェックが入るし、
ファイル削除時にファイルが移動されるゴミ箱も同じディスク上になってしまう。

今回のサーバでは、`/var` を、 ルート(`/`) と共用で容量は 10 GB しか無い。
これは、ownCloud にアップロードできるファイルサイズは 10 GB を超えることができないことになってしまう。

ということで、`/var/lib/docker` をもっと容量のあるディスクに引越しすることにした。

とは言ったものの、データボリュームコンテナが使うパスの変更は容易で無さそうだったため、少し大きなディスクを用意して `/var/lib/docker` にマウントすることにした。

#### Docker 用パーティションの用意

Docker が使う `/var/lib/docker` に専用のパーティションを割り当てることにする。

今回 `/dev/sdc1` が空いていたので、そいつをDocker 用のパーティションにすることにした。これも容量 200 GB しか無いが、まあ作業ディレクトリとしては十分な大きさと言えるだろう。

`ext4` のファイルシステムを作成する。`ext4` にしたのは、なんとなく。

```
[root@fg-arch server]# mkfs.ext4 /dev/sdc1 
mke2fs 1.43.4 (31-Jan-2017)
Creating filesystem with 48839985 4k blocks and 12214272 inodes
Filesystem UUID: 80ce0f6d-71f2-458f-895a-21fd49943f55
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

fstab に書き込んで簡単にマウントできるようにする。

先に `/dev/sdc1` の UUID を調べておき、
それを指定して `/var/lib/docker` へマウントするようにした。

```
[root@fg-arch server]# ls -l /dev/disk/by-uuid | grep sdc1
lrwxrwxrwx 1 root root 10 Apr 10 04:32 80ce0f6d-71f2-458f-895a-21fd49943f55 -> ../../sdc1
[root@fg-arch server]# echo "UUID=80ce0f6d-71f2-458f-895a-21fd49943f55 /var/lib/docker	ext4		rw,relatime,data=ordered0 0" >> /etc/fstab
```

さっそくマウントしたいところだが、
既存の `/var/lib/docker` のデータを回収しなければならない。

#### Docker ディレクトリの用意

まずは、既存ファイルをバックアップする。

```
[root@fg-arch ~]# cd /var/lib/
[root@fg-arch lib]# mv docker docker_bak
```

それから、新しいマウントポイントにさっき作ったパーティションをマウントする。


```
[root@fg-arch server]# mkdir docker
[root@fg-arch server]# mount /var/lib/docker/
```

そしてバックアップしておいたデータをコピー。

```
[root@fg-arch server]# cp -a /var/lib/docker_bak/* /var/lib/docker/
```

#### Docker デーモンの起動

うまくできていれば起動できるはず。

```
[root@fg-arch server]# systemctl start docker
```

### おまけ

はじめは btrfs 上のサブボリュームを `/var/lib/docker` にしようとしたんだけど、
Overlay2 は btrfs 上で使えない的なエラーが出たので素直に諦めた。

```
overlay2 is not supported over btrfs
```

そこまでの手順をここにメモしておく。

#### btrfs の状態確認

`house` というラベルがついたファイルシステムが、5個のディスクで組まれている。

`/dev/sdc1` がまったく仕事をしていないので、こいつを Docker 用にしようと思った。

```
[root@fg-arch server]# btrfs filesystem show
Label: 'house'  uuid: 962205ef-673e-4b93-be7e-58bd23d68095
	Total devices 5 FS bytes used 2.30TiB
	devid    1 size 2.61TiB used 1.07TiB path /dev/sda5
	devid    2 size 2.73TiB used 1.16TiB path /dev/sdb1
	devid    3 size 186.31GiB used 0.00B path /dev/sdc1
	devid    4 size 2.73TiB used 1.19TiB path /dev/sdd1
	devid    5 size 2.73TiB used 1.19TiB path /dev/sde1
```

`/dev/sdc1` は下のようにして取り外した。

```
[root@fg-arch server]# mount /dev/sda5 /mnt
[root@fg-arch server]# btrfs device delete /dev/sdc1 /mnt
[root@fg-arch server]# btrfs filesystem show
Label: 'house'  uuid: 962205ef-673e-4b93-be7e-58bd23d68095
	Total devices 4 FS bytes used 2.30TiB
	devid    1 size 2.61TiB used 1.07TiB path /dev/sda5
	devid    2 size 2.73TiB used 1.16TiB path /dev/sdb1
	devid    4 size 2.73TiB used 1.19TiB path /dev/sdd1
	devid    5 size 2.73TiB used 1.19TiB path /dev/sde1
```
