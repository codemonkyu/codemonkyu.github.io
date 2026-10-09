---
title: "[Linux] Linux Storage Management"
date: 2024-10-27 17:09:43 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: ["File System", "Linux", "LVM", "NFS"]
tistory_url: https://codemonkyu.tistory.com/entry/Linux-Linux-Storage-Management
---

![출처 : https://www.youtube.com/watch?app=desktop&amp;v=bbmWOjuFmgA](/assets/img/posts/linux-storage-management/01.jpg)
_출처 : https://www.youtube.com/watch?app=desktop&v=bbmWOjuFmgA_

오늘은 Linux 에서 File system에 대한 간단한 설명과 이를 이용하여 실제 물리적 Disk에 대해 File system을 사용하여 마운트하는 방법을 정리하고 LVM과 NFS를 구성하여 정리해 보려고 한다.

---

1) Linux Basic I / O Stack

![그림 1](/assets/img/posts/linux-storage-management/02.png)
_그림 1_

가상 파일 시스템(Virtual File System, VFS):

VFS는 리눅스 커널의 일부로, 다양한 파일 시스템을 통일된 인터페이스로 접근할 수 있게 해줍니다.
VFS는 파일, 디렉토리, 마운트, 인증 등 파일 시스템 관련 작업을 추상화하여 제공합니다.
애플리케이션은 VFS 인터페이스를 통해 파일 시스템과 상호작용하므로, 실제 사용 중인 파일 시스템의 종류를 몰라도 됩니다.
VFS는 파일 시스템 간 투명한 전환을 가능하게 하고, 디스크 입출력 시 공통적인 작업을 처리합니다.

파일 시스템(File System):

파일 시스템은 데이터를 디스크에 저장하고 관리하는 방법을 정의합니다.
리눅스에는 다양한 파일 시스템이 지원됩니다. 예를 들어 ext4, XFS, btrfs 등이 있습니다.
각 파일 시스템은 데이터 저장, 메타데이터 관리, 파일 및 디렉토리 구조 등을 다루는 자체 메커니즘을 가지고 있습니다.
파일 시스템은 장치 드라이버와 직접 상호작용하여 디스크 입출력을 처리합니다.

---

2) FileSystem 생성 및 관리

### Create filesystem
[root@amzn2_lab01p ~]# lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda 202:0 0 8G 0 disk
└─xvda1 202:1 0 8G 0 part /
xvdb 202:16 0 10G 0 disk

[root@ ~]# lsblk -o +SERIAL
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT SERIAL
nvme1n1       259:0    0  10G  0 disk            vol0a6bcacc2f8419027
nvme0n1       259:1    0   8G  0 disk            vol096f219309d55c8a3
├─nvme0n1p1   259:2    0   8G  0 part /
└─nvme0n1p128 259:3    0   1M  0 part
> https://docs.aws.amazon.com/ebs/latest/userguide/identify-nvme-ebs-device.html

[root@amzn2_lab01p ~]# mkfs -t ext4 /dev/xvdb
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
655360 inodes, 2621440 blocks
131072 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2151677952
...(생략)...

### mount
[root@amzn2_lab01p ~]# mount /dev/xvdb /app
[root@amzn2_lab01p ~]# df -Th
Filesystem Type Size Used Avail Use% Mounted on
devtmpfs devtmpfs 467M 0 467M 0% /dev
tmpfs tmpfs 477M 0 477M 0% /dev/shm
tmpfs tmpfs 477M 404K 476M 1% /run
tmpfs tmpfs 477M 0 477M 0% /sys/fs/cgroup
/dev/xvda1 xfs 8.0G 2.8G 5.3G 34% /
tmpfs tmpfs 96M 0 96M 0% /run/user/1000
/dev/xvdb ext4 9.7G 24K 9.2G 1% /app

### check journal info
[root@amzn2_lab01p ~]# dumpe2fs /dev/xvdb | grep -i journal
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem features: has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Journal inode: 8
Journal backup: inode blocks
Journal features: journal_64bit
Journal size: 128M
Journal length: 32768
Journal sequence: 0x00000002
Journal start: 1

### blkid
[root@amzn2_lab01p ~]# blkid
/dev/xvda1: LABEL="/" UUID="584a9aa3-3944-4412-83d4-601901aea52d" TYPE="xfs" PARTLABEL="Linux" PARTUUID="47a8e07d-0c1d-41b6-92af-bcf3b1e2ff88"
/dev/xvdb: UUID="1d02ee7e-5065-4c20-ae93-9aa54bb8f403" TYPE="ext4"

### fstab
[root@amzn2_lab01p ~]# cat /etc/fstab
#
UUID=584a9aa3-3944-4412-83d4-601901aea52d / xfs defaults,noatime 1 1
UUID=1d02ee7e-5065-4c20-ae93-9aa54bb8f403 /app ext4 defaults 0 0
<device name> <mount point> <fs type> <mount options> <dump> <fsck option>

<dump> : 0은 백업하지 않음. 1은 dump로 backup을 할지를 결정
<fsck option> : 0은 fsck로 체크하지 않음. 1은 root file system, 다른 파티션들은 2

---

3) File System 관련 도구

fuser : umount 시 사용중인 프로세스 확인 및 정리 시 주로 사용.

-u: 사용중인 프로세스 & 사용자 확인
-k : 해당 프로세스 kill 진행.
lsof 명령어로도 확인 가능

fsck, e2fsck, xfs_repiar > 파일시스템을 점검하고 손상된 파일시스템 복구 수행.

참고 자료 : Red Hat 문서

du : 서브디렉토리들의 크기를 알기에 유용함, df -h 파일시스템 full 정리시 주로 사용.

참고 자료 : https://tech.chhanz.xyz/linux/2021/05/25/use-du/

[root@amzn2_lab01p ~]# umount /mnt
umount: /mnt: target is busy.

[root@amzn2_lab01p ~]# fuser -u /mnt
/mnt:                 3821c(ec2-user)

[root@amzn2_lab01p ~]# ps -ef | grep 3821
ec2-user  3821  3820  0 07:57 pts/1    00:00:00 -bash  <= bash 로그인쉘.

[root@amzn2_lab01p ~]# fuser -k /mnt
/mnt:                 3821c

### using lsof command.
[root@amzn2_lab01p ~]# lsof +D /mnt
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
bash    3821 root  cwd    DIR 202,16     4096    2 /mnt

$ sudo mount file-system
$ sudo umount file-system

$ sudo xfs_repair -n block-device

$ sudo mount file-system

[root@amzn2_lab01p DATA]# ll
total 4
-rw-rw-r-- 1 ec2-user ec2-user 11 Sep 27 06:44 a
drwxrwxr-x 2 ec2-user ec2-user  6 Sep 27 06:44 bb
drwxr-xr-x 2 root     root     22 Oct 12 08:07 test
drwxr-xr-x 2 root     root     36 Oct 12 08:11 test2_dir

[root@amzn2_lab01p ~]# du -hs /DATA
1.5G    /DATA

[root@amzn2_lab01p DATA]# du -ah . | sort -h
0    ./bb
4.0K    ./a
4.0K    ./test2_dir/a_file
511M    ./test2_dir/temp.log
512M    ./test2_dir
1.0G    ./test
1.0G    ./test/test.log
1.5G    .

---

4) LVM 및 파티션 관리

![](/assets/img/posts/linux-storage-management/03.png)

전체적인 LVM 구성 과정 요약 : PV -> VG -> LV -> filesystem
PV : Physical Volume / PV는 LVM을 통해 관리되는 실제 물리적 저장 공간을 제공
VG : 하나 이상의 PV를 그룹화한 것입니다. VG는 LVM에서 논리 볼륨(LV)을 생성하기 위한 저장 공간의 풀 역할을 합니다.
LV : VG 내에서 할당된 논리적 저장 공간. 파일 시스템이 생성되고 데이터가 저장됨

[root@amzn2_lab01p DATA]# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0    8G  0 disk
└─xvda1 202:1    0    8G  0 part /
xvdc    202:32   0  100G  0 disk
xvdd    202:48   0  100G  0 disk

[root@amzn2_lab01p DATA]# pvcreate /dev/xvdc
  Physical volume "/dev/xvdc" successfully created.
[root@amzn2_lab01p DATA]# pvcreate /dev/xvdd
  Physical volume "/dev/xvdd" successfully created.

[root@amzn2_lab01p DATA]# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sdc
  VG Name               vg_data
  PV Size               100.00 GiB / not usable 4.00 MiB ### PE size * num of PE
  Allocatable           yes
  PE Size               4.00 MiB   ### 기본값은 4MiB 지만 변경 가능, 8MiB 등..
  Total PE              25599  :   ### 총 PE 수
  Free PE               25599
  Allocated PE          0
  PV UUID               dNbTfM-pKVU-o7rb-DRsp-uIT0-W1mG-PYfD8r

  --- Physical volume ---
  PV Name               /dev/sdd
  VG Name               vg_data
  PV Size               100.00 GiB / not usable 4.00 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              25599
  Free PE               25599
  Allocated PE          0
  PV UUID               F5RgfP-LvdJ-9HOd-D7F0-ZLGH-utn7-KU2kIT

[root@amzn2_lab01p DATA]# pvs
  PV         VG      Fmt  Attr PSize    PFree
  /dev/sdc   vg_data lvm2 a--  <100.00g <100.00g
  /dev/sdd   vg_data lvm2 a--  <100.00g <100.00g

[root@amzn2_lab01p DATA]# vgcreate vg_data /dev/xvdc /dev/xvdd
  Volume group "vg_data" successfully created

[root@amzn2_lab01p DATA]# vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  vg_data   2   0   0 wz--n- 199.99g 199.99g

[root@amzn2_lab01p DATA]# lvcreate -n app -L 50G vg_data
  Logical volume "app" created.
[root@amzn2_lab01p DATA]# lvcreate -n applog -L 30G vg_data
  Logical volume "applog" created.

[root@amzn2_lab01p DATA]# lvs
  LV     VG      Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  app    vg_data -wi-a----- 50.00g
  applog vg_data -wi-a----- 30.00g

### filesystem  : /dev/mapper/ 디렉토리에 존재
[root@amzn2_lab01p DATA]# mkfs -t xfs /dev/mapper/vg_data-app
meta-data=/dev/mapper/vg_data-app isize=512    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=0 inobtcount=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

---

VG, LV 관리 : 축소, 확장, 삭제
LV축소의 경우 축소 불가한 파일시스템 타입(xfs)도 있고 파일시스템 축소 작업은 데이터 손실을 초래할 수 있으므로 일반적으로 삭제 후 재생성을 권장.

### VG확장
[root@amzn2_lab01p DATA]# vgextend vg_data /dev/xvde
  Volume group "vg_data" successfully

### LV 확장
[root@amzn2_lab01p DATA]# vgs
  VG      #PV #LV #SN Attr   VSize    VFree
  vg_data   3   2   0 wz--n- <249.99g <169.99g
[root@amzn2_lab01p DATA]# lvextend /dev/vg_data/app -L +20G
[root@amzn2_lab01p DATA]# lvextend /dev/vg_data/app -l +100%FREE / 남은 VG전체 할당.
[root@amzn2_lab01p DATA]# xfs_growfs /dev/mapper/vg_data-app  / 파일시스템 확장

### LV 축소 : !!축소가 불가한 파일시스템 타입(xfs)도 있고 축소 작업은 데이터 손실을 초래할 수
있으므로 일반적으로 삭제 후 재생성 권장.

[root@amzn2_lab01p DATA]# umount /app
[root@amzn2_lab01p DATA]# wipefs -a /dev/mapper/vg_data-app
/dev/mapper/vg_data-app: 4 bytes were erased at offset 0x00000000 (xfs): 58 46 53 42

 > 파일 시스템의 서명을 삭제. 미수행 후 lvcreate시 아래 경고 문구 출력 (큰 문제는 없음)
  WARNING: xfs signature detected on /dev/vg_data/app at offset 0. Wipe it? [y/n]: y

[root@amzn2_lab01p DATA]# lvremove /dev/vg_data/app
Do you really want to remove active logical volume vg_data/app? [y/n]: y

---

- vgchange : VG에 대한 접근 여부 설정 / Active-Standby 구조에서 공유 스토리지 볼륨을 사용할 때, 특정 서버(Active 서버)에서만 볼륨 접근 가능하도록 설정하는 경우, 장애 발생되어 자동으로 Standby로 failover시 HA솔루션 내에 vgchange 가 사용될 수 있음.

[root@amzn2_lab01p DATA]# vgchange -a n vg_data  // 비활성화
  0 logical volume(s) in volume group "vg_data" now active

[root@amzn2_lab01p DATA]# vgchange -a y vg_data  // 활성화
  2 logical volume(s) in volume group "vg_data" now active

---

5) Network File system 구성

NFS Server 작업

### Server 단
[root@amzn2_lab01p ~]# yum install nfs-utils

[root@amzn2_lab01p ~]# vim /etc/exports
/DATA    13.125.214.84(rw,sync,no_root_squash)
<공유할_디렉터리> <클라이언트_IP 또는 호스트명>(옵션)

* rw: 읽기 및 쓰기 권한을 모두 허용.
* ro: 읽기 전용 권한을 허용.
* sync: 클라이언트의 쓰기 요청이 완료될 때까지 서버가 응답하지 않음.
* async: 클라이언트의 쓰기 요청을 비동기적으로 처리.
* no_subtree_check: 서브 디렉터리에 대한 체크를 하지 않음.
* no_root_squash: 클라이언트의 root 사용자에게도 서버의 root 권한을 부여.
* root_squash: 클라이언트의 root 사용자를 일반 사용자로 변환하여 권한을 제한.

[root@amzn2_lab01p ~]# systemctl enable nfs-server.service
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.

[root@amzn2_lab01p ~]# service nfs-server status
Redirecting to /bin/systemctl status nfs-server.service
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: inactive (dead)

[root@amzn2_lab01p ~]# service nfs-server start
Redirecting to /bin/systemctl start nfs-server.service

[root@amzn2_lab01p ~]# exportfs -v
/DATA             3.39.253.128(rw,sync,wdelay,hide,no_subtree_check,sec=sys,secure,no_root_squash,no_all_squash)

[root@amzn2_lab01p ~]# rpcinfo -p | grep nfs
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100227    3   udp   2049  nfs_acl

NFS Client 작업

### Client 단
[root@nfs_client01 ~]# mount -t nfs 43.201.72.5:/DATA /nfs

[root@nfs_client01 ~]# df -h
Filesystem         Size  Used Avail Use% Mounted on
devtmpfs           467M     0  467M   0% /dev
tmpfs              477M     0  477M   0% /dev/shm
tmpfs              477M  412K  476M   1% /run
tmpfs              477M     0  477M   0% /sys/fs/cgroup
/dev/xvda1         8.0G  1.9G  6.2G  23% /
tmpfs               96M     0   96M   0% /run/user/1000
43.201.72.5:/DATA  8.0G  4.3G  3.8G  53% /nfs

/> fstab :
43.201.72.5:/DATA /nfs nfs defaults 0 0
* NFS마운트옵션 :
https://docs.aws.amazon.com/ko_kr/efs/latest/ug/mounting-fs-nfs-mount-settings.html

[root@nfs_client01 /]# mount | grep nfs
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw,relatime)
43.201.72.5:/DATA on /nfs type nfs4 (rw,relatime,vers=4.1,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=172.31.14.146,local_lock=none,addr=43.201.72.5)

---

해당 글을 통해 "Linux 파일 시스템의 개요 및 관리 방법" 그리고 "LVM", "NFS" 에 대한 기본적인 개념 및 구성 방법을 정리하며, 헷갈릴 수 있는 부분과 실제 사용 시 도움이 될 수 있는 내용을 추가하였다.
