# Домашняя работа № 3

+ Установлена ВМ для выполнения работы
+ Ознакомимся с имеющимися томами в ВМ. Выполняем команду lsblk

|NAME                     |MAJ:MIN |RM  |SIZE |RO |TYPE| MOUNTPOINT|
|-------------------------|:-------|:---|:----|:--|:---|:--------|
|sda                      | 8:0    |0  | 40G  |0| disk |          |
|├─sda1                    |8:1    |0   | 1M  |0 |part |          |
|├─sda2                    |8:2    |0    |1G  |0| part |/boot     |
|└─sda3                    |8:3    |0  | 39G  |0 |part |          |
|  ├─VolGroup00-LogVol00 |253:0    |0 |37.5G  |0| lvm  |/         |
|  └─VolGroup00-LogVol01 |253:1    |0  |1.5G  |0 |lvm  |[SWAP]    |
|sdb                      | 8:16   |0   |10G  |0| disk |         |
|sdc                       |8:32   |0    |2G  |0 |disk |          |
|sdd                       |8:48   |0    |1G  |0| disk |          |
|sde                       |8:64  | 0    |1G | 0 |disk |           |

## Уменьшить том под 8Гб
+ Подготовим временный том для / раздела

  ```[vagrant@lvm /]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
  [vagrant@lvm /]$ sudo vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
  [vagrant@lvm /]$ sudo lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
  
 + Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные
 
 ```[vagrant@lvm /]$ sudo mkfs.xfs /dev/vg_root/lv_root 
 meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
    =                       sectsz=512   attr=2, projid32bit=1
    =                       crc=1        finobt=0, sparse=0
 data=                      bsize=4096   blocks=2620416, imaxpct=25
    =                       sunit=0      swidth=0 blks
 naming=version 2           bsize=4096   ascii-ci=0 ftype=1
 log=internal log           bsize=4096   blocks=2560, version=2
    =                       sectsz=512   sunit=0 blks, lazy-count=1
 realtime=none              extsz=4096   blocks=0, rtextents=0
 [vagrant@lvm /]$ sudo mount /dev/vg_root/lv_root /mnt/
 ```
 + командой xfsdump скопируем все даннýе с / раздела в /mnt:
 
 




