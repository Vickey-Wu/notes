#### 磁盘分类

- 固态硬盘SSD(solid state disk)
运行速度快, 但容量小价格贵
- 机械硬盘HDD(hard disk driver)
相对慢, 容量大价格便宜, 但容易坏
- 混合硬盘HHD(hybrid hard disk)
固态硬盘和机械硬盘的结合体

#### 不同接口类型硬盘
`/dev/sdxxx`: scsi(small computer system interface)接口或sata(serial advanced technology attachment)接口类型的硬盘
`/dev/hdxxx`: ide(integrated drive electronics)类型硬盘
`/dev/vdxxx`: 一般是虚拟硬盘

- 验证方法(e.g: vda)
`cat /sys/block/vda/queue/rotational`返回1则是机械硬盘HDD, 返回0则为固态硬盘SSD

#### 阿里云扩容系统盘

- [扩展分区和文件系统_Linux系统盘](https://help.aliyun.com/document_detail/111738.html?spm=a2c4g.11186623.2.30.24177f67AnGYEY#concept-ocb-htw-dhb)

- [在线扩容云盘](https://help.aliyun.com/document_detail/113316.html?spm=a2c4g.11186623.2.18.2f411b25o53A6S#concept-syg-jxz-2hb)

#### [虚拟机扩容系统盘](https://devops.ionos.com/tutorials/increase-the-size-of-a-linux-root-partition-without-rebooting/)

0. 在控制台扩展磁盘大小

![](https://note.youdao.com/yws/api/personal/file/D348F950E13342FCA0A901C86E7D1A18?method=download&shareKey=a45a08cd04dfea388c2ea9f20c4bbde9)

1. 原来的3个分区大小为29G, 1G, 1G
```
root@ubuntu:/home/vickey# fdisk -l
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x99ba741e

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sda1  *        2048 60817407 60815360   29G 83 Linux
/dev/sda2       60819454 62912511  2093058 1022M  5 Extended
/dev/sda5       60819456 62912511  2093056 1022M 82 Linux swap / Solaris

```
2. 关闭swap并删除已有分区
> 注意: 2. 3. 4. 5步骤是连续的, 中间不要退出, 不然配置不生效.

>我这里有3个分区, 按3次d就全删了
```
root@ubuntu:/home/vickey# swapoff -a
root@ubuntu:/home/vickey# fdisk /dev/sda

Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d
Partition number (1,2,5, default 5): 

Partition 5 has been deleted.

Command (m for help): d
Partition number (1,2, default 2): 

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.
```
3. 重新创建分区
>重新创建第1个分区, 依次按n, p, enter, enter, +36G(我分了40G中的36G给分区1)

>重新创建第2个分区, 依次按n, p, enter, enter, enter(总共40G分2个区, 直接enter默认将剩余4G给第二个分区)
```
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-83886079, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-83886079, default 83886079): +36G

Created a new partition 1 of type 'Linux' and of size 36 GiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): 
First sector (75499520-83886079, default 75499520): 
Last sector, +sectors or +size{K,M,G,T,P} (75499520-83886079, default 83886079): 

Created a new partition 2 of type 'Linux' and of size 4 GiB.
```
4. 修改第2个分区的类型
>依次按t, 2, 82

>t(修改类型), 82(按L可以查看所有分区类型,这里82表示改为`Linux swap`类型)
```
Command (m for help): t
Partition number (1,2, default 2): 2
Partition type (type L to list all types): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris        
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden or  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx         
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data    
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility   
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt         
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access     
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O        
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor      
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi ea  Rufus alignment
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         eb  BeOS fs        
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ee  GPT            
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        ef  EFI (FAT-12/16/
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f0  Linux/PA-RISC b
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f1  SpeedStor      
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f4  SpeedStor      
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      f2  DOS secondary  
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fb  VMware VMFS    
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fc  VMware VMKCORE 
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fd  Linux raid auto
1c  Hidden W95 FAT3 75  PC/IX           bc  Acronis FAT32 L fe  LANstep        
1e  Hidden W95 FAT1 80  Old Minix       be  Solaris boot    ff  BBT            
Partition type (type L to list all types): 82

Changed type of partition 'Linux' to 'Linux swap / Solaris'.
```
5. 保存配置
>w(保存配置)
```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Device or resource busy

The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or kpartx(8).
```
6. 使用partprobe告诉内核关于新分区的信息就不必重启服务器了
```
root@ubuntu:/home/vickey# partprobe
```
7. 调整文件系统的大小
```
root@ubuntu:/home/vickey# resize2fs /dev/sda1
resize2fs 1.42.13 (17-May-2015)
Filesystem at /dev/sda1 is mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 3
The filesystem on /dev/sda1 is now 9437184 (4k) blocks long.
```
8. 初始化swap分区
```
root@ubuntu:/home/vickey# mkswap /dev/sda2 
Setting up swapspace version 1, size = 4 GiB (4293914624 bytes)
no label, UUID=ebe0d047-8fae-4e5f-9d9c-636932ab1737
```
9. 设置开机自动挂载分区
>将第8步UUID值`ebe0d047-8fae-4e5f-9d9c-636932ab1737`替换原有的UUID值(类型为swap的那行)
```
root@ubuntu:/home/vickey# vim /etc/fstab
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda1 during installation
UUID=8d88b730-2c24-4717-857a-76a15d79971d /               ext4    errors=remount-ro 0       1
# swap was on /dev/sda5 during installation
UUID=ebe0d047-8fae-4e5f-9d9c-636932ab1737 none            swap    sw              0       0
/dev/fd0        /media/floppy0  auto    rw,user,noauto,exec,utf8 0       0
```
10. 重新打开swap
>`fdisk -l`可以看到重新分区后的磁盘大小已经是36G和4G了
```
root@ubuntu:/home/vickey# swapon -a
root@ubuntu:/home/vickey# fdisk -l
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x99ba741e

Device     Boot    Start      End  Sectors Size Id Type
/dev/sda1           2048 75499519 75497472  36G 83 Linux
/dev/sda2       75499520 83886079  8386560   4G 82 Linux swap / Solaris
```

#### 参考文章
- [1. ncrease The Size Of A Linux Root Partition Without Rebooting](https://devops.ionos.com/tutorials/increase-the-size-of-a-linux-root-partition-without-rebooting/)
- [2. 扩展分区和文件系统_Linux系统盘](https://help.aliyun.com/document_detail/111738.html?spm=a2c4g.11186623.2.30.3aea7f67y2izdC#concept-ocb-htw-dhb)
- [3. Linux扩展磁盘空间到根目录（Vmware和KVM通用）](https://blog.csdn.net/weixin_29115985/article/details/81092179)
