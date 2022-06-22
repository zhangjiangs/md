# 磁盘挂载

## 大于2T的磁盘挂载方法

因为fdisk工具最大只能创建2T的空间的分区，所以我们需要使用parted工具创建分区！

### 查看磁盘详情

```shell
[root@gd-xuelang-manage ~]# fdisk -l
Disk /dev/sda: 549.8 GB, 549755813888 bytes, 1073741824 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000b3723

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1640447      819200   83  Linux
/dev/sda2         1640448  1073741823   536050688   8e  Linux LVM

Disk /dev/mapper/centos-root: 531.7 GB, 531732889600 bytes, 1038540800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 17.2 GB, 17179869184 bytes, 33554432 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdb: 4398.0 GB, 4398046511104 bytes, 8589934592 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
#====================================或者==================================
[root@gd-xuelang-manage ~]# lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0   512G  0 disk 
├─sda1            8:1    0   800M  0 part /boot
└─sda2            8:2    0 511.2G  0 part 
  ├─centos-root 253:0    0 495.2G  0 lvm  /
  └─centos-swap 253:1    0    16G  0 lvm  [SWAP]
sdb               8:16   0     4T  0 disk 
sr0              11:0    1  55.9M  0 rom  
```

### 使用parted命令分区

#### 选择硬盘

```shell
[root@gd-xuelang-manage ~]# parted /dev/sdb
GNU Parted 3.1
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

### 转换gpt分区、手动分区

```shell
(parted) mklabel gpt        #转换gpt分区                                            
(parted) mkpart prinmary 1 -1           #分成一个分区                                  
(parted) print                                       #查看分区情况                     
Model: VMware Virtual disk (scsi)
Disk /dev/sdb: 4398GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name      Flags
 1      1049kB  4398GB  4398GB               prinmary

(parted) q                                                        #退出        
Information: You may need to update /etc/fstab.
```

### 查看4T硬盘分区并格式化磁盘为xfs

```bash
[root@gd-xuelang-manage ~]# lsblk      
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0   512G  0 disk 
├─sda1            8:1    0   800M  0 part /boot
└─sda2            8:2    0 511.2G  0 part 
  ├─centos-root 253:0    0 495.2G  0 lvm  /
  └─centos-swap 253:1    0    16G  0 lvm  [SWAP]
sdb               8:16   0     4T  0 disk 
└─sdb1            8:17   0     4T  0 part 
sr0              11:0    1  55.9M  0 rom  
[root@gd-xuelang-manage ~]# df -TH
Filesystem              Type      Size  Used Avail Use% Mounted on
devtmpfs                devtmpfs   68G     0   68G   0% /dev
tmpfs                   tmpfs      68G     0   68G   0% /dev/shm
tmpfs                   tmpfs      68G  9.4M   68G   1% /run
tmpfs                   tmpfs      68G     0   68G   0% /sys/fs/cgroup
/dev/mapper/centos-root xfs       532G  1.6G  530G   1% /
/dev/sda1               xfs       836M  158M  679M  19% /boot
tmpfs                   tmpfs      14G     0   14G   0% /run/user/0
[root@gd-xuelang-manage ~]# mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=268435328 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1073741312, imaxpct=5
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=521728, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

### 挂载

```bash
mount /dev/sdb1 /usr/local/   #将/dev/sdb1  挂载到/usr/local下
```

### 取消挂载

```bash
umount /usr/local    
```

### 开机自动挂载

```shell
#编辑fstab文件
/dev/sdb1  /usr/local   xfs      defaults  0 0
```

### 删除分区

```bash
进入：#parted /dev/sdb
查看分区： （parted）p
删除分区1：（parted）rm 1
删除分区2：（parted）rm 2
```

