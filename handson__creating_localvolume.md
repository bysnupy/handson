# How to create LocalVolume using wwn(world wide name)

## Summary

This is a note how to create localvolume using wwn at OCPv4.8 on AWS.
In this test, I attached a EBS 1GB volume to the target worker node in advance.

## Procedures

### Partitioning and Format

For getting wwn, You may need to partitioning and format as follows.

```cmd
$ oc debug node/ip-10-0-166-105.ap-northeast-1.compute.internal

sh-4.4# lsblk -o +uuid,name
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT UUID                                 NAME
nvme0n1     259:0    0   50G  0 disk                                                 nvme0n1
|-nvme0n1p1 259:1    0    1M  0 part                                                 |-nvme0n1p1
|-nvme0n1p2 259:2    0  127M  0 part            16F1-BF59                            |-nvme0n1p2
|-nvme0n1p3 259:3    0  384M  0 part /boot      336b92fd-c071-4e50-8c6c-2172324a90d0 |-nvme0n1p3
`-nvme0n1p4 259:4    0 49.5G  0 part /sysroot   ff248ee2-2274-4bc7-bea7-b6895c58cc3f `-nvme0n1p4
nvme1n1     259:5    0    1G  0 disk                                                 nvme1n1

sh-4.4# fdisk /dev/nvme1n1

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

The old xfs signature will be removed by a write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xfc778099.

Command (m for help): p
Disk /dev/nvme1n1: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0xfc778099

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-2097151, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151): 

Created a new partition 1 of type 'Linux' and of size 1023 MiB.

Command (m for help): p
Disk /dev/nvme1n1: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0xfc778099

Device         Boot Start     End Sectors  Size Id Type
/dev/nvme1n1p1       2048 2097151 2095104 1023M 83 Linux

Command (m for help): l

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

Command (m for help): w


sh-4.4# mkfs.xfs /dev/nvme1n1
meta-data=/dev/nvme1n1           isize=512    agcount=8, agsize=32768 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=1      swidth=1 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

sh-4.4# lsblk -o +uuid,name
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT UUID                                 NAME
nvme0n1     259:0    0   50G  0 disk                                                 nvme0n1
|-nvme0n1p1 259:1    0    1M  0 part                                                 |-nvme0n1p1
|-nvme0n1p2 259:2    0  127M  0 part            16F1-BF59                            |-nvme0n1p2
|-nvme0n1p3 259:3    0  384M  0 part /boot      336b92fd-c071-4e50-8c6c-2172324a90d0 |-nvme0n1p3
`-nvme0n1p4 259:4    0 49.5G  0 part /sysroot   ff248ee2-2274-4bc7-bea7-b6895c58cc3f `-nvme0n1p4
nvme1n1     259:5    0    1G  0 disk                                                 nvme1n1
`-nvme1n1p1 259:7    0 1023M  0 part                                                 `-nvme1n1p1
```

You can identify your additional disk wwn as follows after formating.
```cmd
sh-4.4# mkfs.xfs /dev/nvme1n1p1 
meta-data=/dev/nvme1n1p1         isize=512    agcount=8, agsize=32736 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=261888, imaxpct=25
         =                       sunit=1      swidth=1 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=1032, version=2
         =                       sectsz=512   sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

sh-4.4# lsblk -o +uuid,name
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT UUID                                 NAME
nvme0n1     259:0    0   50G  0 disk                                                 nvme0n1
|-nvme0n1p1 259:1    0    1M  0 part                                                 |-nvme0n1p1
|-nvme0n1p2 259:2    0  127M  0 part            16F1-BF59                            |-nvme0n1p2
|-nvme0n1p3 259:3    0  384M  0 part /boot      336b92fd-c071-4e50-8c6c-2172324a90d0 |-nvme0n1p3
`-nvme0n1p4 259:4    0 49.5G  0 part /sysroot   ff248ee2-2274-4bc7-bea7-b6895c58cc3f `-nvme0n1p4
nvme1n1     259:5    0    1G  0 disk                                                 nvme1n1
`-nvme1n1p1 259:7    0 1023M  0 part            1a42144b-3227-493e-ad3b-a1d88e1c8181 `-nvme1n1p1


# ls -l /dev/disk/by-id/ | grep nvme1n1
lrwxrwxrwx. 1 root root 13 Nov 19 03:32 nvme-Amazon_Elastic_Block_Store_vol0591dc197dd5c631e -> ../../nvme1n1
lrwxrwxrwx. 1 root root 15 Nov 19 03:34 nvme-Amazon_Elastic_Block_Store_vol0591dc197dd5c631e-part1 -> ../../nvme1n1p1
lrwxrwxrwx. 1 root root 13 Nov 19 03:32 nvme-nvme.1d0f-766f6c3035393164633139376464356336333165-416d617a6f6e20456c617374696320426c6f636b2053746f7265-00000001 -> ../../nvme1n1
lrwxrwxrwx. 1 root root 15 Nov 19 03:34 nvme-nvme.1d0f-766f6c3035393164633139376464356336333165-416d617a6f6e20456c617374696320426c6f636b2053746f7265-00000001-part1 -> ../../nvme1n1p1
lrwxrwxrwx. 1 root root 15 Nov 19 03:34 wwn-nvme.1d0f-766f6c3035393164633139376464356336333165-416d617a6f6e20456c617374696320426c6f636b2053746f7265-00000001-part1 -> ../../nvme1n1p1
```
### Create LocalVolume

Created localvolume using wwn as follows.
```cmd
$ oc create -f - <<EOF
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-disks"
  namespace: "openshift-local-storage"
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ip-10-0-166-105
  storageClassDevices:
    - storageClassName: "local-sc"
      volumeMode: Filesystem
      fsType: xfs
      devicePaths:
        - /dev/disk/by-id/wwn-nvme.1d0f-766f6c3035393164633139376464356336333165-416d617a6f6e20456c617374696320426c6f636b2053746f7265-00000001-part1
EOF

$ oc get sc
NAME            PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs          Delete          WaitForFirstConsumer   true                   58m
gp2-csi         ebs.csi.aws.com                Delete          WaitForFirstConsumer   true                   58m
local-sc        kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  39s

$ oc get localvolume
NAME          AGE
local-disks   17s

$ oc get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-2ba95e4e   1023Mi     RWO            Delete           Available           local-sc                7s
```

Done.
