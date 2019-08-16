---
title:      "docker 启动错误排查"
date:       2019-08-14 13:32:50 +0800
categories: 技术
tags:
- docker
- thinpool
---

# 重启 docker
dockerd 有问题时，想要清空 docker 数据重启服务，可以执行以下操作。
```sh
systemctl stop docker     # 停止 docker 服务
dmsetup udevcomplete_all  # 释放未完成的磁盘操作
rm -rf /var/lib/docker/*  # 清空 docker 数据
systemctl start docker    # 重启 docker 服务
```
注：docker 默认的数据目录为 /var/lib/docker（根据实际情况修改）。如有文件删除不了时，可能需要重启服务器。

启动 docker 失败，报错：
> devmapper: unable to take ownership of thin-pool (docker-thinpool) that already has used data blocks

这里应该执行 `ps -ef | grep docker`，查出 docker 相关进程，然后全部 kill 掉，再尝试重启 docker 服务，我直接尝试删除 thin-pool， 走上重建 thinpool 的不归路。

# 重建 thinpool
## 删除 thinpool
执行命令 `lvremove docker/thinpool` 尝试删除 thinpool，报错：
> Logical volume is used by another device

执行以下命令删除被占用的设备，参考[这里](http://blog.roberthallam.org/2017/12/solved-logical-volume-is-used-by-another-device/comment-page-1/)，大概3个步骤：

1) Find Out device-mapper’s Mapping
```sh
> dmsetup info -c | grep docker

vg-lv--old       253   9 L--w    1    2      1 LVM-6O3jLvI6ZR3fg6ZpMgTlkqA...
```

2) Find Out Mapped Device
```sh
> ls -la /sys/dev/block/253\:9/holders

drwxr-xr-x 2 root root 0 Dec 12 01:07 .
drwxr-xr-x 8 root root 0 Dec 12 01:07 ..
lrwxrwxrwx 1 root root 0 Dec 12 01:07 dm-2 -> ../../dm-2
```

3) Remove Device
```sh
dmsetup remove /dev/dm-2
```
再执行命令 `lvremove docker` 删除有问题的 pool

## 创建 thinpool
启动 docker 报错：
> devicemapper: non existing device docker-docker--pool

执行下面的命令重建 docker pool
```sh
#!/bin/sh
lvcreate --wipesignatures y -n thinpool docker -l 95%VG -y
lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG -y
lvconvert -y --zero n -c 512k --thinpool docker/thinpool \
  --poolmetadata docker/thinpoolmeta
\cp docker-thinpool.profile /etc/lvm/profile/docker-thinpool.profile
lvchange --metadataprofile docker-thinpool docker/thinpool
lvs -o+seg_monitor
```

再启动 docker，正常运行了。
