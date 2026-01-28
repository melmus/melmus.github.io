---
layout
title: Создание и увеличение диска LVM
---

# Создание и увеличение диска LVM


Если на сервере закончилось место - у Вас есть возможность добавить ещё несколько десятков/сотен/тысяч ГБ к забитому HDD. Ниже описано как это можно сделать.

Для продолжения у Вас должно выполняться 2 правила:
- Должен быть подключен LVM
- Должно быть именно УВЕЛИЧЕНИЕ объёма. На уменьшение или на перераспределение объёма это руководство не работает.

## Инструкция

В качестве примера возьмём 1 сервер с двумя дисками: OS в 24 ГБ и DATA в 1 ГБ. Мы будем увеличивать данный объём до 250 и 500 ГБ соответственно.

Первоначально необходимо добавить необходимые нам объёмы в сами HDD на аппаратном уровне.

В качестве этого мы в примере будем использовать панель управления vCloud и Hardware-настройки выключенной виртуальной машины (ВМ)

После этого Вы можете включить ВМ и увидеть, что нужные объёмы добавлены к жёстким дискам
```bash
fdisk -l
Disk /dev/sda: 268.4 GB, 268435456000 bytes, 524288000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000d585a
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    50331647    24652800   8e  Linux LVM
Disk /dev/sdb: 536.9 GB, 536870912000 bytes, 1048576000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
Однако, сама ОС использует пока старые объёмы.
```bash
df -h
Filesystem              Size  Used Avail Use% Mounted on
devtmpfs                7.7G     0  7.7G   0% /dev
tmpfs                   7.8G     0  7.8G   0% /dev/shm
tmpfs                   7.8G   12M  7.7G   1% /run
tmpfs                   7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/mapper/INTVG-root  3.9G  2.2G  1.5G  61% /
/dev/sda1               477M  296M  152M  67% /boot
/dev/mapper/INTVG-tmp   2.0G  6.1M  1.8G   1% /tmp
/dev/mapper/INTVG-home  2.0G  6.5M  1.8G   1% /home
/dev/mapper/INTVG-var   5.8G  678M  4.9G  13% /var
/dev/mapper/INTVG-opt   2.0G  6.2M  1.8G   1% /opt
tmpfs                   1.6G     0  1.6G   0% /run/user/1001
```
Это нам и необходимо исправить с помощью LVM

Сначала необходимо сделать новый разделы на новых объёмах с помощью утилиты fdisk
```bash
#*Первый диск*
fdisk /dev/sda
Command (m for help): n
Select: *ENTER*
Partition number): *ENTER*
First sector, +sectors or +size{K,M,G}: *ENTER*
Command (m for help): w
 
#*Второй диск*
fdisk /dev/sdb
Command (m for help): n
Select: *ENTER*
Partition number: *ENTER*
First sector: *ENTER*
Last sector, +sectors or +size{K,M,G}: *ENTER*
Command (m for help):
```
Если после "Command (m for help): w" вы видите сообщение ```"WARNING: Re-reading the partition table failed with error 16: Device or resource busy. The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or kpartx(8)"``` - рекомендуется перезапустить сервер перед продолжением.

Таким образом у нас появились дополнительные разделы
```
    /dev/sda3
    /dev/sdb1
```

Дальше начинаем работать с LVM-утилитами. В примере наша задача - добавить второй раздел (sdb1) полностью в директорию /data и равномерно разбить первый раздел (sda3) между оставшимся директориями (/var, /opt, / и пр.)

Нам необходимо добавить новосозданные разделы в VG.

Первым делом необходимо узнать название VG, куда мы будем добавлять новые объёмы (в примере, INTVG)
```bash
vgdisplay
  --- Volume group ---
  VG Name               INTVG
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  10
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                6
  Open LV               6
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <23.51 GiB
  PE Size               4.00 MiB
  Total PE              6018
  Alloc PE / Size       4608 / 18.00 GiB
  Free  PE / Size       1410 / <5.51 GiB
  VG UUID               G6PyNs-qb92-doxr-ruY2-B3l0-IRdb-6JMbmX
```

Добавляем новые разделы к данному VG
```bash
vgextend INTVG /dev/sda3
vgextend INTVG /dev/sdb1
```
Проверяем
```bash
vgdisplay
  --- Volume group ---
  VG Name               INTVG
  System ID
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  12
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                6
  Open LV               6
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               749.50 GiB
  PE Size               4.00 MiB
  Total PE              191872
  Alloc PE / Size       4608 / 18.00 GiB
  Free  PE / Size       187264 / 731.50 GiB
  VG UUID               G6PyNs-qb92-doxr-ruY2-B3l0-IRdb-6JMbmX
```
Теперь у нас объём равен 749.5 ГиБ, что нам должно хватить. 

### Диск OS и увеличение существующих разделов

Далее переходим на LV и добавляем необходимые объёмы в необходимые нам разделы. Первым делом разберёмся с OS-диском.
```
df -h
Filesystem              Size  Used Avail Use% Mounted on
devtmpfs                7.7G     0  7.7G   0% /dev
tmpfs                   7.8G     0  7.8G   0% /dev/shm
tmpfs                   7.8G   12M  7.7G   1% /run
tmpfs                   7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/mapper/INTVG-root  3.9G  2.2G  1.5G  61% /
/dev/sda1               477M  296M  152M  67% /boot
/dev/mapper/INTVG-tmp   2.0G  6.1M  1.8G   1% /tmp
/dev/mapper/INTVG-home  2.0G  6.5M  1.8G   1% /home
/dev/mapper/INTVG-var   5.8G  679M  4.9G  13% /var
/dev/mapper/INTVG-opt   2.0G  6.2M  1.8G   1% /opt
tmpfs                   1.6G     0  1.6G   0% /run/user/1001
tmpfs                   1.6G     0  1.6G   0% /run/user/0
tmpfs                   1.6G     0  1.6G   0% /run/user/500
```

```bash
lvextend -L47G /dev/mapper/INTVG-var
lvextend -L8G /dev/mapper/INTVG-opt
lvextend -L4G /dev/mapper/INTVG-tmp
lvextend -L5G /dev/mapper/INTVG-home
lvextend -L15G /dev/mapper/INTVG-root
```
Таким образом мы присвоили 47ГБ для var, 8ГБ для opt, 4ГБ для tmp, 5ГБ для home и 15ГБ для root.

Осталось последнее - заставить файловую систему каждого раздела пересчитать и перестроить доступную ёмкость под новые объёмы раздела
```bash
resize2fs /dev/mapper/INTVG-root
resize2fs /dev/mapper/INTVG-tmp
resize2fs /dev/mapper/INTVG-home
resize2fs /dev/mapper/INTVG-var
resize2fs /dev/mapper/INTVG-opt
```
```bash
df -h
Filesystem              Size  Used Avail Use% Mounted on
devtmpfs                7.7G     0  7.7G   0% /dev
tmpfs                   7.8G     0  7.8G   0% /dev/shm
tmpfs                   7.8G   12M  7.7G   1% /run
tmpfs                   7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/mapper/INTVG-root   15G  2.2G   12G  16% /
/dev/sda1               477M  296M  152M  67% /boot
/dev/mapper/INTVG-tmp   3.9G  8.1M  3.7G   1% /tmp
/dev/mapper/INTVG-home  4.9G  8.5M  4.7G   1% /home
/dev/mapper/INTVG-var    47G  689M   44G   2% /var
/dev/mapper/INTVG-opt   7.9G  9.2M  7.5G   1% /opt
tmpfs                   1.6G     0  1.6G   0% /run/user/1001
tmpfs                   1.6G     0  1.6G   0% /run/user/0
```
Диск OS готов. Оставшиеся объёмы оставить пока нетронутыми и добавлять Гигабайты в нужный раздел по мере заполнения, но уже без перезагрузки.

### Диск DATA и добавление нового раздела

Далее продолжим с диском **DATA**. Тут есть отличие - как такового раздела /data в нашем примере не существует и вместо увеличения раздела мы будем создавать и подключать новый раздел /dev/mapper/INTVG-data в /data


Создайте новый LV для data размеров в 500ГБ
```bash
lvcreate -L500G -n data INTVG
```
Создаём файловую систему для нового раздела (пример для файловой системы "ext4")
```bash
mkfs.ext4 /dev/mapper/INTVG-data
```
Раздел готов к подключению.

Добавляем его в fstab и монитруем
```bash
vim /etc/fstab
...
/dev/mapper/INTVG-data   /data                    ext4    defaults        1 2
```
```bash
mkdir /data
mount /data/
df -h
...
/dev/mapper/INTVG-data  493G   73M  467G   1% /data
