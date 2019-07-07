---
layout: post
title:  "Docker+Keras+TensorFlow環境構築その4"
categories: ai
---

いつの間にか手元の TensorFlow が GPU を使わなくなって CPU で超頑張っちゃっていることに気付いたので直した。

改めて設定が完了してみると、GPU (のファン) がコオォォォ…という音を上げており、ちゃんと動いていることがよくわかる。
ちなみに GPU は GeForce GTX 1060 6GB。そろそろ GTX 1080 以上も考えたいところ。

### 1. これまでのいきさつ

TensorFlow は Docker を利用し、公式の [tensorflow/tensorflow](https://hub.docker.com/r/tensorflow/tensorflow/) イメージをベースに Dockerfile で独自のイメージを作って運用している。

さらに Docker イメージとコンテナは Docker Compose で管理している。

昔うまくいっていた設定は以下。このころはスクリプトを一生懸命書いて TensorFlow 環境を起動していたようだ:

- [Keras 環境構築をした その3](https://kikei.github.io/ai/2018/10/14/keras4.html)

その後、本ブログには書いていなかったみたいだが Docker Compose を使うように改良していた。今回はその設定が動かなくなっていたので直した、というお話し。

### 2. docker-compose.yml の修正

TensorFlow が GPU を認識しているか否かは、例えば以下のコードで確認できる。
空のリストが返ってきた場合には GPU が見つけられていない。

```sh
# python3
>>> from tensorflow.python.client import device_lib
>>> device_lib.list_local_devices()
[]
```

GPU が見つけられないということは、CPU で計算を行うことになる。
ニューラルネットのテンソル計算を CPU で行った場合、並列計算やバッチ計算ができない分だけ、処理に時間がかかってしまうと言われている。

さて、うまくいっていないのは docker-compose.yml に問題があるためであると元々察しがついていたので、サクサク調べて次のように直した。
たびたびやり方は変わるようだが、今の docker-compose v3 ではこれでうまくいくようだ。

```yml
version: '3'

volumes:
  nvidia_driver_415.27:
    external: true
services:
  app:
    container_name: app
    build: ./app
    volumes:
      - nvidia_driver_415.27:/usr/local/nvidia:ro
    devices:
      - /dev/nvidiactl
      - /dev/nvidia-uvm
      - /dev/nvidia0
    tty: true
    restart: always
```

これまでは `nvidia-smi` を起動したり、`runtime` を設定したりしていたが、`volume` と`devices` で解決することに成功したようだ。

Docker volume は適切な名前で作成済みである必要があるようだ。
その名前は `nvidia_driver_{VERSION}` の形式になる。
`nvidia-smi` で表示されるドライバのバージョンを指定してうまくいった。
詳しいところは調べていない。

```sh
$ nvidia-smi | head -4
Mon Jul  8 03:29:47 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 415.27       Driver Version: 415.27       CUDA Version: 10.0     |
|-------------------------------|----------------------|----------------------+
```

今回は `415.27` だったので、Volume の名前を `nvidia_driver_415.27` とした。

Volume は次のようにして作成する:

```sh
$ docker volume create --name=nvidia_driver_415.27 -d nvidia-docker
```

こうして作成した Docker volume の名前は先述の docker-compose.yml にも記述してある。

また参考までに、`./app/Dockerfile` は次のようになっている (抜粋)。
まあ特に面白い点は無い。

```sh
$ cat app/Dockerfile
FROM tensorflow/tensorflow:latest-gpu

VOLUME ["/mnt"]
WORKDIR "/mnt"

RUN apt-get update -y --fix-missing

# Python3
RUN apt-get install -y python3 python3-pip
RUN pip3 install --upgrade pip

# Tools
RUN apt-get install -y graphviz

# Start application
COPY entry.sh /root/
RUN chmod u+x /root/entry.sh
ENTRYPOINT ["/root/entry.sh"]
```

entry.sh 内では、/mnt にマウントしたアプリケーションのソースコードに含まれた requirements.txt を見て Python ライブラリをインストールしたり、実際にアプリケーションを起動したりしている。

### 3. TensorFlow のバージョンダウン

ここまでで起動はうまくいくんだけど、実際に TensorFlow を使おうとしてみると次のように表示される:

```
>>> import tensorflow as tf
>>> device = tf.test.gpu_device_name()
2019-07-07 16:16:46.769886: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:1005] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
2019-07-07 16:16:46.771130: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1640] Found device 0 with properties: 
name: GeForce GTX 1060 6GB major: 6 minor: 1 memoryClockRate(GHz): 1.7845
pciBusID: 0000:02:00.0
2019-07-07 16:16:46.771379: I tensorflow/stream_executor/platform/default/dso_loader.cc:53] Could not dlopen library 'libcudart.so.10.0'; dlerror: libcudart.so.10.0: cannot open shared object file: No such file or directory; LD_LIBRARY_PATH: /usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64
2019-07-07 16:16:46.771575: I tensorflow/stream_executor/platform/default/dso_loader.cc:53] Could not dlopen library 'libcublas.so.10.0'; dlerror: libcublas.so.10.0: cannot open shared object file: No such file or directory; LD_LIBRARY_PATH: /usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64
2019-07-07 16:16:46.771761: I tensorflow/stream_executor/platform/default/dso_loader.cc:53] Could not dlopen library 'libcufft.so.10.0'; dlerror: libcufft.so.10.0: cannot open shared object file: No such file or directory; LD_LIBRARY_PATH: /usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64
2019-07-07 16:16:46.771938: I tensorflow/stream_executor/platform/default/dso_loader.cc:53] Could not dlopen library 'libcurand.so.10.0'; dlerror: libcurand.so.10.0: cannot open shared object file: No such file or directory; LD_LIBRARY_PATH: /usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64
2019-07-07 16:16:46.772121: I tensorflow/stream_executor/platform/default/dso_loader.cc:53] Could not dlopen library 'libcusolver.so.10.0'; dlerror: libcusolver.so.10.0: cannot open shared object file: No such file or directory; LD_LIBRARY_PATH: /usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64
2019-07-07 16:16:46.772281: I tensorflow/stream_executor/platform/default/dso_loader.cc:53] Could not dlopen library 'libcusparse.so.10.0'; dlerror: libcusparse.so.10.0: cannot open shared object file: No such file or directory; LD_LIBRARY_PATH: /usr/local/cuda/extras/CUPTI/lib64:/usr/local/nvidia/lib:/usr/local/nvidia/lib64
2019-07-07 16:16:46.772338: I tensorflow/stream_executor/platform/default/dso_loader.cc:42] Successfully opened dynamic library libcudnn.so.7
2019-07-07 16:16:46.772375: W tensorflow/core/common_runtime/gpu/gpu_device.cc:1663] Cannot dlopen some GPU libraries. Skipping registering GPU devices...
2019-07-07 16:16:46.772433: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1181] Device interconnect StreamExecutor with strength 1 edge matrix:
2019-07-07 16:16:46.772465: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1187]      0 
2019-07-07 16:16:46.772499: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1200] 0:   N 
>>> name
''
```

TensorFlowがCUDA 10.0のライブラリを参照しに行ったけれども、無いからエラーになっている。どうもその原因は、インストールしたTensorFlowのバージョンが新しすぎるためのようだ。

調べてみると、CUDA 9.0がインストールされていることがわかった。

ディレクトリ構造から調べる:

```
# ls -l /usr/local/cuda        
lrwxrwxrwx 1 root root 8 Oct 31  2018 /usr/local/cuda -> cuda-9.0
```

`nvcc` コマンドを使って調べる:

```
# nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2017 NVIDIA Corporation
Built on Fri_Sep__1_21:08:03_CDT_2017
Cuda compilation tools, release 9.0, V9.0.176
```

解決方法は CUDA のバージョンアップ、もしくは TensorFlow のバージョンダウンのいずれかが有効そう。今回、CUDA は Docker イメージからインストールしたものであるから、後づけでインストールした TensorFlow の側を合わせることにした。

[Build from source  \|  TensorFlow](https://www.tensorflow.org/install/source#common_installation_problems)

上記ページによると、CUDA 9.0と互換性があるのは "tensorflow_gpu-1.12.0" である。
そこで今持っている TensorFlow をアンインストールし、1.12.0 を再度インストールする。

```sh
# pip3 uninstall tensorflow-gpu
# pip3 install tensorflow-gpu==1.12.0
```

これでもう一度、先程と同じように TensorFlow を使ってみるとうまくいった:

```sh
# python3
Python 3.5.2 (default, Nov 12 2018, 13:43:14) 
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> tf.test.gpu_device_name()
2019-07-07 17:09:18.740512: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:964] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
2019-07-07 17:09:18.741631: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1432] Found device 0 with properties: 
name: GeForce GTX 1060 6GB major: 6 minor: 1 memoryClockRate(GHz): 1.7845
pciBusID: 0000:02:00.0
totalMemory: 5.94GiB freeMemory: 5.86GiB
2019-07-07 17:09:18.741687: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1511] Adding visible gpu devices: 0
2019-07-07 17:09:19.254343: I tensorflow/core/common_runtime/gpu/gpu_device.cc:982] Device interconnect StreamExecutor with strength 1 edge matrix:
2019-07-07 17:09:19.254408: I tensorflow/core/common_runtime/gpu/gpu_device.cc:988]      0 
2019-07-07 17:09:19.254419: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1001] 0:   N 
2019-07-07 17:09:19.254941: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1115] Created TensorFlow device (/device:GPU:0 with 5641 MB memory) -> physical GPU (device: 0, name: GeForce GTX 1060 6GB, pci bus id: 0000:02:00.0, compute capability: 6.1)
'/device:GPU:0'
```

### 3. 参考

- [tensorflow/tensorflow \- Docker Hub](https://hub.docker.com/r/tensorflow/tensorflow/)
- [Docker  \|  TensorFlow Core  \|  TensorFlow](https://www.tensorflow.org/install/docker)
- [nvidia\-dockerをdocker\-composeから使う \- WonderPlanet Tech Blog](http://tech.wonderpla.net/entry/2017/11/14/110000)
- [ubuntu \- How to tell if tensorflow is using gpu acceleration from inside python shell? \- Stack Overflow](https://stackoverflow.com/questions/38009682/how-to-tell-if-tensorflow-is-using-gpu-acceleration-from-inside-python-shell)
- [Build from source  \|  TensorFlow](https://www.tensorflow.org/install/source#common_installation_problems)
- [tensorflow\-gpuとCUDAのバージョン \- 知識のサラダボウル](https://omedstu.jimdo.com/2018/06/28/tensorflow-gpu%E3%81%A8cuda%E3%81%AE%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3/)
