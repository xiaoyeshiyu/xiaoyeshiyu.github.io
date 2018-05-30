---
title: GPU PassThrough in KVM

description: 近年人工智能、算法、深度学习这些技术非常火，而这些技术在用到计算能力的时候，通常都首选GPU，为了适应多用户共同学习或者处理图形图像的使用环境，基于VM配合GPU就有比较大的优势。本文主要介绍在KVM下，将宿主机上的GPU通过直通的方式提供给客户机。

categories:
 - OpenStack
 - GPU
tags:
 - OpenStack
 - KVM
 - GPU
---

## 实现步骤

### 环境

OS：

``` shell
# cat /etc/redhat-release 
CentOS Linux release 7.3.1611 (Core)
# uname -a
Linux hyhive 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

QEMU版本

```xml
Compiled against library: libvirt 2.0.0
Using library: libvirt 2.0.0
Using API: QEMU 2.0.0
Running hypervisor: QEMU 2.6.0
```

### 操作步骤

#### 1.查找`pci`设备的硬件信息

`pci passthrough`需要`pci`的`vendor_id`和`product_id`以及`iommu`组编号，通常再知道`pci`设备的类型或者厂商名称时，可以使用`lspci`查找，如（这里测试有三块`Quadro M2000`显卡）：

```shell
# lspci -nn | grep -i vga
06:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206GL [Quadro M2000] [10de:1430] (rev a1)
0a:03.0 VGA compatible controller [0300]: Matrox Electronics Systems Ltd. MGA G200eW WPCM450 [102b:0532] (rev 0a)
82:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206GL [Quadro M2000] [10de:1430] (rev a1)
83:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206GL [Quadro M2000] [10de:1430] (rev a1)
```

``` shell
group:06:00.0       82:00.0          83:00.0
venderid:10de       10de             10de
productid:1430      1430             1430
```

#### 2.打开`iommu`

`iommu`功能需要CPU和系统的支持，硬件上要求CPU支持vt-d技术（不是虚拟化）,并在`bios`中打开（通常在北桥芯片设置中）

![配置vt-d](https://cl.ly/1p3w1U1P1I0Z/Screen%20recording%202017-10-18%20at%2005.41.58%20PM.gif)

软件上，需要操作系统的`iommu grub`标志，添加**rd.driver.pre=vfio-pci intel_opmmu=on**

``` shell
# cat /etc/sysconfig/grub 
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet rd.driver.pre=vfio-pci intel_iommu=on"
GRUB_DISABLE_RECOVERY="true"
```

重新生成`grub`：

`EFI`

``` shell
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
```

非`EFI`

```shell
# grub2-mkconfig -o /boot/grub2/grub.cfg 
Generating grub configuration file ...
  WARNING: Not using lvmetad because duplicate PVs were found.
  WARNING: Use multipath or vgimportclone to resolve duplicate PVs?
  WARNING: After duplicates are resolved, run "pvscan --cache" to enable lvmetad.
  WARNING: Not using lvmetad because duplicate PVs were found.
  WARNING: Use multipath or vgimportclone to resolve duplicate PVs?
  WARNING: After duplicates are resolved, run "pvscan --cache" to enable lvmetad.
Found linux image: /boot/vmlinuz-3.10.0-514.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-514.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-43b1cb10c00d48b18061c67e6df40752
Found initrd image: /boot/initramfs-0-rescue-43b1cb10c00d48b18061c67e6df40752.img
  WARNING: Not using lvmetad because duplicate PVs were found.
  WARNING: Use multipath or vgimportclone to resolve duplicate PVs?
  WARNING: After duplicates are resolved, run "pvscan --cache" to enable lvmetad.
  WARNING: Not using lvmetad because duplicate PVs were found.
  WARNING: Use multipath or vgimportclone to resolve duplicate PVs?
  WARNING: After duplicates are resolved, run "pvscan --cache" to enable lvmetad.
done
```

#### 3.配置`vfio`模块

1. 屏蔽宿主机驱动（该步骤验证为非必须步骤，当出现异常时尝试添加）

以nvidia显卡为例：

```shell
# cat /etc/modprobe.d/blacklist.conf
blacklist nouveau
blacklist nvidia
```

2. 添加`vfio`所需内核模块

``` shell
# cat /etc/modules-load.d/vfio.conf 
vfio
vfio_iommu_type1
vfio_pci
```

3. 配置`vfio`

将1.1中查询到的`vendor_id`和`product_id`以`vendor_id:product_id`的形式添加到`vfio option`中，完成`vfio`配置

检查硬件脚本(可以通过以下脚本检查1.1中得到的id是否有误)

```shell
# cat gpu.sh 
#!/bin/bash
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done;
```

通过检查硬件脚本找到显卡对应的显卡设备和声卡设备，将两个设备id都填入vfio.conf文件中

```shell
# ./gpu.sh | grep -i nvidia
IOMMU Group 18 06:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206GL [Quadro M2000] [10de:1430] (rev a1)
IOMMU Group 18 06:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fba] (rev a1)
IOMMU Group 38 82:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206GL [Quadro M2000] [10de:1430] (rev a1)
IOMMU Group 38 82:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fba] (rev a1)
IOMMU Group 39 83:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206GL [Quadro M2000] [10de:1430] (rev a1)
IOMMU Group 39 83:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:0fba] (rev a1)
```

从上面可以查到的显卡设备为`10de:1430`声卡设备为`10de:0fba`

添加到`vfio.conf`文件中

```shell
# cat /etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:1430,10de:0fba
```

4. 重新生成`initramfs`

```shell
# mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
# dracut -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /usr/sbin/dracut -v /boot/initramfs-3.10.0-514.el7.x86_64.img 3.10.0-514.el7.x86_64
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: network ***
*** Including module: ifcfg ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** Constructing GenuineIntel.bin ****
*** Store current command line parameters ***
*** Creating image file ***
*** Creating microcode section ***
*** Created microcode section ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-514.el7.x86_64.img' done ***
```

5. 重启

### 验证

重启之后，验证`vfio`是否生效

```shell
# dmesg | grep -i vfio
[    0.000000] Command line: BOOT_IMAGE=/vmlinuz-3.10.0-514.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet rd.driver.pre=vfio-pci intel_iommu=on
[    0.000000] Kernel command line: BOOT_IMAGE=/vmlinuz-3.10.0-514.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet rd.driver.pre=vfio-pci intel_iommu=on
[    8.362177] VFIO - User Level meta-driver version: 0.3
[    8.367249] vfio_pci: add [10de:1430[ffff:ffff]] class 0x000000/00000000
[    8.367314] vfio_pci: add [10de:0fba[ffff:ffff]] class 0x000000/00000000
[ 2144.638305] vfio-pci 0000:06:00.0: enabling device (0000 -> 0003)
[ 2144.638464] vfio_ecap_init: 0000:06:00.0 hiding ecap 0x1e@0x258
[ 2144.638479] vfio_ecap_init: 0000:06:00.0 hiding ecap 0x19@0x900
```

失败案例：

```shell
# dmesg | grep vfio 
[    0.000000] Command line: BOOT_IMAGE=/vmlinuz-3.10.0-514.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet rd.driver.pre=vfio-pci intel_iommu=on
[    0.000000] Kernel command line: BOOT_IMAGE=/vmlinuz-3.10.0-514.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet rd.driver.pre=vfio-pci intel_iommu=on
[   19.034596] vfio-pci: probe of 0000:06:00.0 failed with error -22
[   19.034636] vfio-pci: probe of 0000:82:00.0 failed with error -22
[   19.034654] vfio-pci: probe of 0000:83:00.0 failed with error -22
[   19.034661] vfio_pci: add [10de:1430[ffff:ffff]] class 0x000000/00000000
```

解决办法：

检查`/etc/modprobe.d/vfio.conf `文件，检查主板北桥`vt-d`设置是否打开

## 调用GPU

创建虚拟机，并添加`pci`设备

 ```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x06' slot='0x00' function='0x1'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </hostdev>
 ```

其中`address`为`1.1`中`06:00:0`

通过`virsh domain ***.xml`孵化出虚拟机即可。 

## ATI部分显卡及某些其他PCI卡reset问题

`ATI`部分显卡和其他某些`cpi`卡存在`reset`问题，具体表现为：

1. 虚拟机重启一次或几次后，`pci passthrough`失效，需要重启宿主机
2. 虚拟机重启时蓝屏，然后恢复正常

该问题由于`pci reset`导致，和`pci`板卡驱动程序相关，对于`kvm`则无根治手段，可以采用的方式：

>  https://curtisshoward.com/post/fixing-amd-gpu-passthrough-reset-issues-in-windows/

`qemu-kvm`代码中关于该问题的解释：

>  https://github.com/qemu/qemu/blob/master/hw/vfio/pci-quirks.c#L1697

## 相关资源

`pci passthrough`当前比较成熟，以下是部分资料：

1. [VGA Passthrough on virtual machines in CentOS 7](https://gist.github.com/cuibonobo/d354440fecdd37c35ecd)
2. [PCI passthrough via OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)
3. [VGAPassthrough](https://wiki.debian.org/VGAPassthrough)
4. [HOW-TO make dual-boot obsolete using kvm VGA passthrough](https://forums.linuxmint.com/viewtopic.php?f=231&t=212692&sid=e446a11cb6bc36c4b5e6d4a04efb8703)
5. [create-gaming-virtual-machine-using-vfio-pci-passthrough-kvm](http://www.firewing1.com/howtos/fedora-20/create-gaming-virtual-machine-using-vfio-pci-passthrough-kvm)

和`pci reset`相关

1. [pci-quirks](https://github.com/qemu/qemu/blob/master/hw/vfio/pci-quirks.c#L1697)
2. [pass-through gpu to guest, but only once?!](https://www.redhat.com/archives/vfio-users/2015-September/msg00467.html)
3. [vfio pass through only works once after reboot](https://www.redhat.com/archives/vfio-users/2015-September/msg00051.html)
4. [Fixing AMD GPU Passthrough Reset Issues in Windows](https://curtisshoward.com/post/fixing-amd-gpu-passthrough-reset-issues-in-windows/)

在线测试结果

[KVM GPU PASSTHROUGH RESULT](https://docs.google.com/spreadsheets/d/1LnGpTrXalwGVNy0PWJDURhyxa3sgqkGXmvNCIvIMenk/edit#gid=0)