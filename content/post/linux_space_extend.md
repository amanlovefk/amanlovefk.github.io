---
title: "Ubuntu 22 虚拟机扩容"
date: 2023-01-09T15:43:43+08:00
draft: false
---
首先关闭虚拟机，然后给虚拟机磁盘扩容。

```shell
fdisk /dev/sda
```

进入之后按 d 选择删除 sda 现存的分区，从最大分区开始删，一直删到第一个。

然后新建分区：n, p, 一路回车

之后 reboot

在进入 ：

```shell
resize2fs /dev/sda4
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

resize2fs /dev/sad4 是扩充最大编号的分区


