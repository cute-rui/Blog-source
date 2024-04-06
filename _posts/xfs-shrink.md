---
title: 记一次 XFS 硬盘缩容
date: 2024-04-06 09:43:00
---
由于 CentOS 7 的云模板镜像使用了 XFS，并且默认大小为 8G，实际文件大小只有不到 2G，开二十台测试机的硬盘就要被塞满了... 由于XFS 只提供了扩容，无法直接进行缩容，所以缩容方式大多比较 Dirty

## 思路

把旧模板的内容直接拷贝进新模板，修改 fstab 与 grub 的 UUID，或者直接用旧 UUID 覆盖

## 步骤

先开一台虚拟机，挂载两个模板上去，为了避免某些问题，可以调整挂载/启动顺序来人工干预硬件名

```bash
# 虚拟机启动的硬盘是 SDB , 新模板的硬盘是 SDA 
mount -t xfs -o nouuid /dev/sdc1 /mnt/sdc # 三个分区 UUID 相同，故忽略 UUID
```

使用 fdisk 将 /dev/sda1 缩容到 2G，再次格式化，修改分区 UUID 为 /dev/sdc1 的 UUID

```bash
fdisk /dev/sda # 进入后对 /dev/sda1 进行缩容操作，需要选中 boot flag

···

mkfs.xfs /dev/sda1 # 格式化 /dev/sda1

blkid /dev/sdc1 # 显示 UUID

xfs_admin -U $uuid /dev/sda1 # 把原来的 UUID 塞回去

```

直接挂载 /dev/sda1，把原有镜像内容拷贝到新镜像

```bash
mount -t xfs -o nouuid /dev/sda1 /mnt/sda

cp -rf /mnt/sdc/* /mnt/sda/
```

关机卸载 SDA ，扔进 libguest 缩减镜像大小

```bash
virt-resize --shrink /dev/sda1 $new_image_name $dest_image_name
```

完成