---
layout: post
title:  "Docker+Keras+TensorFlow環境構築 その5"
categories: ai
---

我が家には日々ニューラルネットを訓練し続けている24時間起動 PC があるが、
この度、ブレーカーを落とされ PC の電源が強制終了してしまった。

よい機会と思い `pacman -Syu` したところ、ニューラルネットを訓練する Docker コンテナが起動しなくなった。この記事では頑張って復旧させた手順を書く。

とはいえ実際には環境を作り直したので結果的に、
2019/8/10 現在における最新の環境構築手順がここに記載されることとなった。

大まかなシステム構成は以下のようになっている:

- Arch Linux
- Docker Compose
- Docker
- Keras
- TensorFlow
- CUDA
- NVIDIA GeForce

### 1. これまでのいきさつ

2年前、[Keras の環境構築をした](https://kikei.github.io/ai/2017/03/04/keras.html) の記事にて Keras on TensorFlow on CUDA の環境を構築した。
少し前の記事 [Docker\+Keras\+TensorFlow環境構築その4](https://kikei.github.io/ai/2019/07/08/tensorflow.html) では Docker 経由で Keras を使い、かつ TensorFlow が GPU が使って計算するように設定した。Docker コンテナ一式を簡単に起動するため Docker Compose も使っている。

環境は以下の通り:

- GPU: GeForce GTX 1060 6GB

```console
# lspci -k | grep -A 2 -E "(VGA|3D)"
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v2/3rd Gen Core processor Graphics Controller (rev 09)
	Subsystem: ASRock Incorporation Motherboard
	Kernel driver in use: i915
--
02:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] (rev a1)
	Subsystem: ASUSTeK Computer Inc. GP106 [GeForce GTX 1060 6GB]
	Kernel driver in use: nouveau
```

- OS: Arch Linux

```console
$ uname -a
Linux fg-arch 5.2.7-arch1-1-ARCH #1 SMP PREEMPT Wed Aug 7 02:00:26 UTC 2019 x86_64 GNU/Linux
```

### 2. 旧環境の削除

Docker コンテナが起動しなかったのは、nvidia-docker プラグインが見つからない的なエラーによるものだったと記憶している。しかし控えなかったので最早わからない。

とにかく面倒そうなエラーだったので潔く環境を作り直すことにした。

まずは古いアプリケーションを削除する。

NVIDIA、CUDA のパッケージの削除:

```console
# pacman -Rsn nvidia
# pacman -Rsn cuda
# rm -r /opt/cuda
```

当時、CUDNN は Pacman を使わず手動でインストールしたようなので、
`/opt/cuda` も明示的に手動で削除しておいた。

Docker や Docker Compose については別でも使っているので削除しなかった。

### 3. ディスク容量の確保

この手順はスルーしてよい。

私の環境ではディスク容量がピンチな感じになっていて、調べたところ
Pacman のキャッシュが肥大化していることがわかったので掃除した。

以下のようにすることで、Pacman のキャッシュディレクトリから古く使っていないパッケージを削除することができた:

```console
# pacman -S --help | head -4 | grep -v dbpath
usage:  pacman {-S --sync} [options] [package(s)]
options:
  -c, --clean          remove old packages from cache directory (-cc for all)
# pacman -Sc
```

[How to clean up pacman download cache on Arch Linux \| Support Tips](https://supporttips.com/how-to-clean-up-pacman-download-cache-on-arch-linux/)

### 4. 新環境の構築

小手調べに Pacman の全パッケージをアップグレードした:

```console
# pacman -Syu
```

#### CUDA 導入

CUDA と CUDNN をインストールした:

```
# pacman -S cuda
# pacman -S cudnn
```

CUDNN は CUDA の Deep Neural Network 向けライブラリである。

#### NVIDIA ドライバ導入

次に NVIDIA ドライバをインストールする。
今思えばこれは CUDA より先にやるべきな気もするが、この順番でも問題無かった。

[NVIDIA \- ArchWiki](https://wiki.archlinux.org/index.php/NVIDIA) が GPU のモデルをちゃんと調べろと言っているので調べる。

> 1. If you do not know what graphics card you have, find out by issuing:
> 2. Determine the necessary driver version for your card by:
> 3. Install the appropriate driver for your card:

私の環境では GeForce GTX 1060 を使っている、というのは前述の通り。
そこで下記のページから GTX 1060 を探すと、コードネーム NV136 (NV130 Family) とのことである。

- Code name: NV136 (GP106)
- Official Name: GeForce GTX 1060

[CodeNames](https://nouveau.freedesktop.org/wiki/CodeNames/)

先程の ArchWiki によると、NV130 Family ならば普通に Pacman でインストールすればよいらしい:

```console
# pacman -S nvidia
```

インストールされた NVIDIA ドライバのバージョン:

```console
# nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2019 NVIDIA Corporation
Built on Wed_Apr_24_19:10:27_PDT_2019
Cuda compilation tools, release 10.1, V10.1.168
```

#### NVIDIA Container Tookit 導入

どうやら Docker 19.03 より、Docker 公式で NVIDIA GPU をサポートするようになったらしい。NVIDIA Container Toolkit を使えばいいよ、とのことだ。

[Docker \- ArchWiki](https://wiki.archlinux.org/index.php/Docker)

> Starting from Docker version 19.03, NVIDIA GPUs are natively supported as Docker devices. NVIDIA Container Toolkit is the recommended way of running containers that leverage NVIDIA GPUs.
> Install the nvidia-container-toolkit AUR package. Next, restart docker. You can now run containers that make use of NVIDIA GPUs using the --gpus option:

以下のようなコマンドは Deprecated になったらしい:

```console
# docker run --runtime=nvidia nvidia/cuda:9.0-base nvidia-smi
# nvidia-docker run nvidia/cuda:9.0-base nvidia-smi
```

Docker のバージョン確認してみると 19.03 だった。
NVIDIA GPU をサポートする Docker のようである:

```console
# docker version
Client:
 Version:           19.03.1-ce
 API version:       1.40
 Go version:        go1.12.7
 Git commit:        74b1e89e8a
 Built:             Sat Jul 27 21:08:50 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          19.03.1-ce
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.7
  Git commit:       74b1e89e8a
  Built:            Sat Jul 27 21:08:28 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.2.7.m
  GitCommit:        85f6aa58b8a3170aec9824568f7a31832878b603.m
 runc:
  Version:          1.0.0-rc8
  GitCommit:        425e105d5a03fabd737a126ad93d62a9eeede87f
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

そこで nvidia-container-toolkit をインストールしていく。

ただしそのためには、先に libnvidia-container を自前でインストールする必要がある。

[AUR \(en\) \- libnvidia\-container](https://aur.archlinux.org/packages/libnvidia-container/)

これは AUR という方式で提供されており、Pacman をそのまま使うだけではインストールすることができない。AUR というのは Arch User Repository らしい。

[Arch User Repository \- ArchWiki](https://wiki.archlinux.org/index.php/Arch_User_Repository)

PKGBUILD ファイルのあるディレクトリで makepkg コマンドを使うことによりインストールできた:

```console
$ git clone https://aur.archlinux.org/libnvidia-container.git
$ cd libnvidia-container 
$ ls PKGBUILD
$ makepkg -si
==> Making package: libnvidia-container 1.0.2-1 (Sat 10 Aug 2019 01:23:08 AM JST)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Installing missing dependencies...
略
Packages (2) libnvidia-container-1.0.2-1  libnvidia-container-tools-1.0.2-1

Total Installed Size:  0.32 MiB
略
```

なお makepkg コマンドはユーザー権限でないと実行できない。

次に nvidia-container-toolkit をインストールした。

[AUR \(en\) \- nvidia\-container\-toolkit](https://aur.archlinux.org/packages/nvidia-container-toolkit/)

先程と同じ要領で:

```console
$ cd ..
$ git clone https://aur.archlinux.org/nvidia-container-toolkit.git
$ cd nvidia-container-toolkit
$ ls
PKGBUILD
$ makepkg -si 
==> Making package: nvidia-container-toolkit 1.0.1-5 (Sat 10 Aug 2019 01:28:02 AM JST)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Installing missing dependencies...
resolving dependencies...
looking for conflicting packages...

Packages (1) go-2:1.12.7-1

Total Download Size:   122.17 MiB
Total Installed Size:  476.27 MiB
略
Packages (1) nvidia-container-toolkit-1.0.1-5

Total Installed Size:  3.00 MiB
略
```

インストールに成功したら Docker を再起動した:

```console
# systemctl restart docker
# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2019-08-10 01:35:16 JST; 17s ago
     Docs: https://docs.docker.com
略
```

nvidia-smi を実行し GPU に接続:

```console
# docker run --gpus all nvidia/cuda:9.0-base nvidia-smi
```

エラーが出た。
CUDA が使えるデバイスが見付からないと言っている気がする:

```console
docker: Error response from daemon: OCI runtime create failed: container_linux.go:345: starting container process caused "process_linux.go:430: container init caused \"process_linux.go:413: running prestart hook 0 caused \\\"error running hook: exit status 1, stdout: , stderr: exec command: [/usr/bin/nvidia-container-cli --load-kmods configure --ldconfig=@/sbin/ldconfig --device=all --compute --utility --require=cuda>=9.0  --pid=25736 /var/lib/docker/overlay2/167dea498a0640855ce4a4e8b019d9fd3339ef21ff6a50020d5ea1c4930c47b4/merged]\\\\nnvidia-container-cli: initialization error: cuda error: no cuda-capable device is detected\\\\n\\\"\"": unknown.
ERRO[0019] error waiting for container: context canceled 
```

NVIDIA ドライバが反映されていなかったと思われる。
以下のディレクトリを覗くと、nvidia が表示されなかった:

```console
# ls /proc/driver/
rtc
```

NVIDIA のドライバは `pacman -S nvidia` によりインストールされているはず。

そこで PC を再起動したところ、表示されるようになった:

```console
# shutdown -r now
# ls /proc/driver 
nvidia	nvidia-nvlink  nvidia-nvswitch	rtc
```

再度 nvidia-smi を実行したところ、うまく接続できた。
`--gpus` オプションを使って CUDA を有効化している:

```
# docker run --rm --gpus all nvidia/cuda:9.0-base nvidia-smi
Fri Aug  9 17:21:31 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 430.40       Driver Version: 430.40       CUDA Version: 10.1     |
|-------------------------------|----------------------|----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 106...  Off  | 00000000:02:00.0 Off |                  N/A |
| 27%   56C    P5     9W / 120W |      0MiB /  6078MiB |      2%      Default |
+-------------------------------|----------------------|----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

#### Docker Compose

ここで残念なお知らせがある。

Docker の `--gpus` オプションにより CUDA が使えるようになったわけだが、
Docker Compose 側ではこのオプションがまだサポートされていない。

そこで代替手段として、一時的に1世代前 (nvidia-docker2) の `--runtime=nvidia` オプションを使う方法を用いる。

このオプションを利用できるようにするためには、

- nvidia-container-runtime をインストール、
- nvidia ランタイムを Docker のデーモン設定に追加し、
- docker-compose.yml にランタイム設定を追記する。
- さらに Docker Compose のファイルバージョンを 2.3 にして、runtime が指定できるようにする、

という手順を踏むようだ。

- [How to use \-\-gpus option with docker\-compose? \- General Discussions \- Docker Forums](https://forums.docker.com/t/how-to-use-gpus-option-with-docker-compose/78558)
- [Support for NVIDIA GPUs under Docker Compose · Issue \#6691 · docker/compose · GitHub](https://github.com/docker/compose/issues/6691)

nvidia-container-runtime をインストールする:

```console
$ git clone https://aur.archlinux.org/nvidia-container-runtime.git
$ cd nvidia-container-runtime 
$ ls
PKGBUILD
$ makepkg -si
==> Making package: nvidia-container-runtime 3.1.0-2 (Sat 10 Aug 2019 02:39:53 AM JST)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Retrieving sources...
Packages (1) nvidia-container-runtime-3.1.0-2

Total Installed Size:  3.07 MiB
略
```

インストールに成功すると、nvidia-container-runtime というファイルが作られている:

```
$ ls /usr/bin/nvidia-container-runtime
/usr/bin/nvidia-container-runtime
```

Docker の nvidia ランタイムの設定を作成:

```console
# vim /etc/docker/daemon.json
# cat /etc/docker/daemon.json 
{
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

docker-compose.yml を編集する。
編集内容は、

- version を 2.3 にする。
- `runtime: nvidia` を追記する。

最小だと以下のような感じ:

```yaml
version: '2.3'

services:
  app:
    container_name: app
    build: ./app
    runtime: nvidia
    tty: true
```

そして Docker コンテナを起動して動作確認する:

```console
# docker-compose build
# docker-compose up -d
# docker-compose exec app /bin/bash
```

コンテナの中で Python3 を起動し TensorFlow に繋いでみたところ、
GPU デバイスが取得できた。

```
# python3
Python 3.5.2 (default, Nov 23 2017, 16:37:01) 
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> device = tf.test.gpu_device_name()
2019-08-09 17:52:18.289650: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:964] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
2019-08-09 17:52:18.290476: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1432] Found device 0 with properties: 
name: GeForce GTX 1060 6GB major: 6 minor: 1 memoryClockRate(GHz): 1.7845
pciBusID: 0000:02:00.0
totalMemory: 5.94GiB freeMemory: 5.87GiB
2019-08-09 17:52:18.290521: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1511] Adding visible gpu devices: 0
2019-08-09 17:52:28.325800: I tensorflow/core/common_runtime/gpu/gpu_device.cc:982] Device interconnect StreamExecutor with strength 1 edge matrix:
2019-08-09 17:52:28.325851: I tensorflow/core/common_runtime/gpu/gpu_device.cc:988]      0 
2019-08-09 17:52:28.325863: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1001] 0:   N 
2019-08-09 17:52:28.325987: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1115] Created TensorFlow device (/device:GPU:0 with 5643 MB memory) -> physical GPU (device: 0, name: GeForce GTX 1060 6GB, pci bus id: 0000:02:00.0, compute capability: 6.1)
>>> device
'/device:GPU:0'
```

NUMA の INFO が出ているが、私の PC は NUMA アーキテクチャでないと思うので無視することにする。

[CPUを複数持つアーキテクチャNUMAとnumactlの説明 \- Qiita](https://qiita.com/bokotomo/items/5352b8ceb6ccb70390c8)

### 5. まとめ

Docker 19.03 より NVIDIA GPU を公式にサポートするようになり、`--gpus` というオプションが使える。
今回、Arch Linux 上で `--gpus` を利用可能にすることに成功した。

ただ Docker Compose では、現時点 (2019/8/10) でまだ新しいオプションがサポートされていないため、代替手段を用いる必要がある。

サポートがなされたことに気付き次第、次の記事を書く予定である。

### 6. 参考

Pacman:

- [How to clean up pacman download cache on Arch Linux \| Support Tips](https://supporttips.com/how-to-clean-up-pacman-download-cache-on-arch-linux/)

Arch Linux:

- [NVIDIA \- ArchWiki](https://wiki.archlinux.org/index.php/NVIDIA)
- [Docker \- ArchWiki](https://wiki.archlinux.org/index.php/Docker)
- [Arch User Repository \- ArchWiki](https://wiki.archlinux.org/index.php/Arch_User_Repository)
- [AUR \(en\) \- nvidia\-container\-toolkit](https://aur.archlinux.org/packages/nvidia-container-toolkit/)
- [AUR \(en\) \- libnvidia\-container](https://aur.archlinux.org/packages/libnvidia-container/)
- [AUR \(en\) \- nvidia\-container\-runtime](https://aur.archlinux.org/packages/nvidia-container-runtime/)

NVIDIA:

- [CodeNames](https://nouveau.freedesktop.org/wiki/CodeNames/)

Docker Compose:

- [How to use \-\-gpus option with docker\-compose? \- General Discussions \- Docker Forums](https://forums.docker.com/t/how-to-use-gpus-option-with-docker-compose/78558)
- [Support for NVIDIA GPUs under Docker Compose · Issue \#6691 · docker/compose · GitHub](https://github.com/docker/compose/issues/6691)

NUMA:

- [CPUを複数持つアーキテクチャNUMAとnumactlの説明 \- Qiita](https://qiita.com/bokotomo/items/5352b8ceb6ccb70390c8)
