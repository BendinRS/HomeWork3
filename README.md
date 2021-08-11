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
 + командой xfsdump скопируем все данные с / раздела в /mnt

```[root@lvm /]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt  ```

![Рисунок 1](http://images.vfl.ru/ii/1628693416/a85dc0bd/35464909.png "Файлы скопировались успешно")

+ переконфигурируем grub для того, чтобы при старте перейти в новый /
+ Сымитируем текущий root -> сделаем в него chroot и обновим grub
+ Обновим образ initrd
+ в файле /boot/grub2/grub.cfg заменяем rd.lvm.lv=VolGroup00/LogVol01 на rd.lvm.lv=vg_root/lv_root
+ Перезагружаем ВМ. Проверяем, что операция прошла успешно.

![Рисунок 2](http://images.vfl.ru/ii/1628694861/ffc3b457/35465167.png "Директория перенесена успешно")

+ Меняем размер старой VG и возвращаем на него директорию /

## Выделить том под /var в зеркало

+ На свободных дисках sdd и sdd создаем зеркало

  ```[root@lvm /]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
  
  [root@lvm /]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created

  [root@lvm /]#  lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
  ```
+ Создаем на нем ФС и перемещаем туда /var

 ```[root@lvm /]# mkfs.ext4 /dev/vg_var/lv_var
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 
 ```
 
 
  ```[root@lvm /]# mount /dev/vg_var/lv_var /mnt
[root@lvm /]# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
 ```
+ Сохраняем содержимое старого var

```[root@lvm /]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar ```

+ Монтируем новый var в каталог /var

```[root@lvm /]# umount /mnt
   [root@lvm /]# mount /dev/vg_var/lv_var /var
```
+ Правим fstab для автоматического монтирования /var
  
  ```[root@lvm /]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab ```
 
+ Перезагружаем ВМ и удаляем временный Volume Group

## Выделить том под /home

+ Выделяем том под /home по тому же принципу что делали для /var

![Рисунок 3](http://images.vfl.ru/ii/1628699926/86850c5c/35465825.png "Директория перенесена успешно")

## /home - сделать том для снапшотов

+ Сгенерируем файлы в /home/ 

```[root@lvm vagrant]# touch /home/file{1..20} ```
+ Снимаем снапшот
 ```[root@lvm vagrant]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created. 
  ```
+ Восстанавливаемся со снапшота

![Рисунок 4](http://images.vfl.ru/ii/1628700564/a1b9ce5c/35465869.png "Восстановление прошло успешно")

***

### Пробую разобраться с btrfs

+ Диск уже размечен. 
+ Иду по шаблону, но с папкой opt.
+ Файловая система накатывается командой:```  mkfs.btrfs /dev/test/bendin ``` 

Итог:

![Рисунок 4](http://images.vfl.ru/ii/1628704344/29349eea/35466492.png "Восстановление прошло успешно")

### Кэширование

> ВМ переустановлена после неудачных попыток с настройкой кэширования

![Рисунок 5](http://images.vfl.ru/ii/1628712017/4eaab320/35467854.png "Размечены диски")

+ opt/cache - будет пулом кэширования, opt/imen - Кэшируется
``` 
   [root@lvm vagrant]# lvconvert --type cache opt/imen  --cachepool opt/cache
   WARNING: Converting opt/cache to cache pool's data volume with metadata wiping.
   THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
   Do you really want to convert opt/cache? [y/n]: y
   Converted opt/cache to cache pool.
   Logical volume opt/imen is now cached.
  ```
+ Итог
![Рисунок 6](http://images.vfl.ru/ii/1628712240/3a461e80/35467871.png "Процесс кэширования")


























 




