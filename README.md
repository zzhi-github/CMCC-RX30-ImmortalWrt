# CMCC-RX30-ImmortalWrt
Tutorial of CCMCC RX30/RAX3000Z增强版 immortalWrt
2025.1.10 闲鱼上收了一台CMCC型号RX30的路由器，外包装上打印的名称为：RAX3000Z增强版，生产日期为20240106，生产企业：无锡盟创网络科技
再网上搜了几天，很多相关的刷机教程，有点乱；最后还是看到github上的一篇教程比较靠谱: [GithHub/lgs2007m](https://github.com/lgs2007m/Actions-OpenWrt/blob/main/Tutorial/RAX3000M-eMMC_XR30-eMMC.md)。

下面是按照教程的刷机过程，记录一下。
Step 1. 开启SSH
这一步比较简单，因为路由器的conf file没有加密，可以直接download conf file，然后修改相应的两个文件，上传新的conf file后就能ssh root登录192.168.10.1.

Step2. 刷uboot
先看了一下原机的分区：
```
root@XR30:~# fdisk -l /dev/mmcblk0
Found valid GPT with protective MBR; using GPT

Disk /dev/mmcblk0: 120832000 sectors, 1656M
Logical sector size: 512
Disk identifier (GUID): 2bd17853-102b-4500-aa1a-8a21d4d7984d
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 120800000

Number  Start (sector)    End (sector)  Size Name
     1            8192            9215  512K u-boot-env
     2            9216           13311 2048K factory
     3           13312           17407 2048K fip
     4           17408           82943 32.0M kernel
     5           82944          214015 64.0M rootfs
     6          214016          279551 32.0M kernel2
     7          279552          410623 64.0M rootfs2
     8          410624          934911  256M rootfs_data
     9          934912         1065983 64.0M plugins
    10         1065984         1098751 16.0M fwk
    11         1098752         1131519 16.0M fwk2
    12         1131520       120800000 57.0G data
```
教程里面用的是RAX3000M，跟上面的内容完全相同。
有两个rootfs，即双分区，大小都是64M。刷机固件要小于64M才行，除非重新分区。
下面是df -h和blkid的结果，跟RAX3000M的也完全一样。
```
root@XR30:~# df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/root                14.0M     14.0M         0 100% /rom
tmpfs                   240.7M     17.8M    222.9M   7% /tmp
/dev/mmcblk0p8          254.0M     85.7M    168.3M  34% /overlay
overlayfs:/overlay      254.0M     85.7M    168.3M  34% /
tmpfs                   512.0K         0    512.0K   0% /dev
/dev/mmcblk0p10           6.8M      6.8M         0 100% /mnt/mmcblk0p11
/dev/mmcblk0p11           6.8M      6.8M         0 100% /mnt/mmcblk0p11
/dev/mmcblk0p12          55.9G     52.0M     53.0G   0% /mnt/mmcblk0p12
/dev/mmcblk0p9           58.0M      1.3M     52.2M   2% /mnt/mmcblk0p9
/dev/mmcblk0p12          55.9G     52.0M     53.0G   0% /extend
/dev/mmcblk0p9           58.0M      1.3M     52.2M   2% /plugin
/dev/loop0                6.8M      6.8M         0 100% /plugin/cmcc/framework
root@XR30:~# blkid
/dev/loop0: TYPE="squashfs"
/dev/mmcblk0p1: PARTLABEL="u-boot-env" PARTUUID="19a4763a-6b19-4a4b-a0c4-8cc34f4c2ab9"
/dev/mmcblk0p2: PARTLABEL="factory" PARTUUID="8142c1b2-1697-41d9-b1bf-a88d76c7213f"
/dev/mmcblk0p3: PARTLABEL="fip" PARTUUID="18de6587-4f17-4e08-a6c9-d9d3d424f4c5"
/dev/mmcblk0p4: PARTLABEL="kernel" PARTUUID="971f7556-ef1a-44cd-8b28-0cf8100b9c7e"
/dev/mmcblk0p5: TYPE="squashfs" PARTLABEL="rootfs" PARTUUID="309a3e76-270b-41b2-b5d5-ed8154e7542b"
/dev/mmcblk0p6: PARTLABEL="kernel2" PARTUUID="9c17fbc2-79aa-4600-80ce-989ef9c95909"
/dev/mmcblk0p7: PARTLABEL="rootfs2" PARTUUID="f19609c8-f7d3-4ac6-b93e-7fd9fad4b4af"
/dev/mmcblk0p8: LABEL="rootfs_data" UUID="52c20710-4bbd-4f8b-b6eb-03ac38e47967" BLOCK_SIZE="4096" TYPE="f2fs" PARTLABEL="rootfs_data" PARTUUID="a4a43b93-f17d-43e2-b7a7-df0bdf610c77"
/dev/mmcblk0p9: LABEL="plugins" UUID="7df77010-50e7-448f-9b3a-68ea3ac83739" BLOCK_SIZE="1024" TYPE="ext4" PARTLABEL="plugins" PARTUUID="518c1031-c234-4d49-8301-02e7ebe31231"
/dev/mmcblk0p10: TYPE="squashfs" PARTLABEL="fwk" PARTUUID="6e2bd585-7b0b-45b5-a8a1-4cf5436b1f73"
/dev/mmcblk0p11: TYPE="squashfs" PARTLABEL="fwk2" PARTUUID="fd8708ae-59c7-4ed5-a467-54bfe357cb48"
/dev/mmcblk0p12: LABEL="extend" UUID="dc03ebd1-f73a-4dda-96a3-6ff534229bc9" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="data" PARTUUID="3c058515-54c3-452f-9b87-7a4f957b5cd1"
```
Step 2.1 备份原厂分区
前面11个分区，即1 - 11的全部备份。
bl2 在/dev/mmcblk0boot0, uboot在fip分区
根据教程，把备份都存入最后一个分区mmcblk0p12,即最大的一个分区。

Step 2.2 开始刷uboot
把fip bin file用scp上传到路由器/tmp目录
用dd命令刷入新的uboot，然后用md5sum验证结果，没问题就可以重启机器进入uboot。

Step 3 刷入immortalWrt固件 （18M）
进入uboot，按住reset，路由器上电，大约10s后，指示灯变色（我的是红色变为白色），松开reset，uboot启动结束。
电脑IP设置为192.168.1.x，netmask：255.255.255.0，gateway：192.168.1.1，DNS：
浏览器打开IP：192.168.1.1 uboot图形界面，选择rx30固件，upload，固件刷入成功后机器自动重启。
再次打开192.168.1.1进入immortWrt图形界面，进行设备配置。

暂时没有需要更改gpt分区，稍后看有需求再动。
