---
layout: post
title:  "Dockerを使ってSonic PIを動かしてみようとした"
categories: software
---
Dockerを使う練習をしてみようとした。
Sonic PIをインストールすることはできたが、
スーパバイザ側の画面にアプリを起動するまでは至らず。

### 参考
- [Installation on Fedora](https://docs.docker.com/engine/installation/linux/fedora/)
- [Sonic PI](http://sonic-pi.net)

### 手順
以下のようなコマンドを実行したと思う。

{% highlight bash %}
# Install Docker
sudo dnf install docker
sudo systemctl start docker

% sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
Trying to pull repository docker.io/library/hello-world ... latest: Pulling from library/hello-world

79112a2b2613: Pull complete 
4c4abd6d4278: Pull complete 
Digest: sha256:4f32210e234b4ad5cac92efacc0a3d602b02476c754f13d517e1ada048e5a8ba
Status: Downloaded newer image for docker.io/hello-world:latest


Hello from Docker.
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker Hub account:
 https://hub.docker.com

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/

Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.

# Running Docker from fujii user
% cat /etc/group | grep docker
% sudo groupadd docker
% sudo usermod -aG docker fujii
% groups fujii
fujii : fujii wheel adb docker

% docker pull ubuntu:latest
% sudo docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
docker.io/ubuntu        latest              17b6a9e179d7        2 weeks ago         120.7 MB
docker.io/hello-world   latest              4c4abd6d4278        3 weeks ago         967 B

% docker run -it ubuntu bash
root@4121305df77e:/# apt-get update
root@4121305df77e:/# apt-get install sonic-pi
root@4121305df77e:/# sonic-pi
QXcbConnection: Could not connect to display 
Aborted (core dumped)

{% endhighlight %}

Fedora上でUbuntuを起動して、`apt-get`を使ったりできたので、とりあえず満足。
謎アプリケーションを入れるときのsandboxとかにも使えそう。
