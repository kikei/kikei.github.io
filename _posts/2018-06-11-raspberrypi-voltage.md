---
layout: post
title:  "RaspberryPi Under-voltage detected!と言われる"
categories: smart-home
---

電源は 5V/1A の USB ACアダプタを使っている。
USB-Micro USB ケーブルを太くて短いやつに変えたら直った。

Raspberry Pi B では入力電圧が 4.7V を下回ると電圧低下と見做すらしい。

直る前は以下のように怒られていた。

```
Jun 10 06:25:55 fg-rasp1 kernel: [ 2961.913557] Voltage normalised (0x00000000)
Jun 10 06:26:02 fg-rasp1 kernel: [ 2968.153543] Under-voltage detected! (0x00050005)
Jun 10 06:26:27 fg-rasp1 kernel: [ 2993.113487] Voltage normalised (0x00000000)
Jun 10 06:26:29 fg-rasp1 kernel: [ 2995.193488] Under-voltage detected! (0x00050005)
Jun 10 06:26:33 fg-rasp1 kernel: [ 2999.353467] Voltage normalised (0x00000000)
Jun 10 06:26:35 fg-rasp1 kernel: [ 3001.433471] Under-voltage detected! (0x00050005)
Jun 10 06:27:33 fg-rasp1 kernel: [ 3059.673366] Voltage normalised (0x00000000)
Jun 10 06:27:35 fg-rasp1 kernel: [ 3061.763327] Under-voltage detected! (0x00050005)
Jun 10 06:27:42 fg-rasp1 kernel: [ 3067.993318] Voltage normalised (0x00000000)
Jun 10 06:27:48 fg-rasp1 kernel: [ 3074.233326] Under-voltage detected! (0x00050005)
```

上記と関連しているか判らないが、`apt-get install` とかやると高確率でハングしたが
これも直った。
