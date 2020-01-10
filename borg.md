# Borg Backup
BorgBackup (сокращение от Borg) - это программа резервного копирования с дедупликацией. Опционально, он поддерживает сжатие и аутентифицированное шифрование.

Основная цель Borg - предоставить эффективный и безопасный способ резервного копирования данных. Используемая техника дедупликации данных делает Borg пригодным для ежедневного резервного копирования, поскольку сохраняются только изменения. Метод аутентифицированного шифрования позволяет сохранять бэкапы даже в недоверенных хранилищах.

Отличительный особенности Borg

*    Дедупликация: файлы в рамках одного Borg repository (т.е. специальном каталоге в специфичном для Borg формате) делятся на блоки по n мегабайт, а повторяющиеся блоки Borg дедуплицирует. Фрагмент считается дублирующим, если его значение id_hash идентично. В качестве id_hash используется непосредственно криптографический хеш или функция MAC, например, (HMAC-) sha256. Дедупликация происходит именно до сжатия. Для дедупликации рассматриваются все фрагменты в одном и том же репозитории, независимо от того, приходят ли они с разных компьютеров, из предыдущих резервных копий, из одной и той же резервной копии или даже из одного и того же файла.
*    Сжатие: после дедупликации данные ещё и сжимаются. Доступны разные алгоритмы сжатия: lz4, lzma, zlib, zstd.
*    Работа по SSH: Borg бэкапит на удалённый сервер по SSH. На стороне сервера нужен просто установленный Borg.  Возможно настроить доступ только по ключам и, разрешение выполнения Borg’ом только одной своей команды при заходе на сервер. Например, так:
```bash    $ cat .ssh/authorized_keys command="/usr/bin/borg serve" ssh-rsa AAAAB3NzaC1yc… ```

*    Поставляется как в PPA так и статичным бинарником. Borg в виде статичного бинарника даёт возможность запускать его практически везде, где есть хоть минимально современный glibc. (Но не везде — например, не удалось запустить на CentOS 5.)
    Гибкая очистка от старых бэкапов. Можно задать как хранение n последних бэкапов, так и 2 бэкапа в час/день/неделю. В последнем случае будет оставлен последний на конец недели бэкап. Условия можно комбинировать, сделав хранение 7 ежедневных бэкапов за последние 7 дней и 4 недельных.

### Установка

Необходимо установить Borg на клиент и на сервер бэкапов
```bash
wget https://github.com/borgbackup/borg/releases/download/1.1.10/borg-linux64  -O /usr/bin/borg
chmod +x /usr/bin/borg
```

На сервере бэкапов создать пользователя borg:
```bash
useradd -m borg
```

Настроить подключение к серверу бэкапов по ssh-ключу (возможно подключение по логину/паролю в статье это не рассмотрено):

На клиенте выполнить:
```bash
cd /root/.ssh
ssh-keygen -t rsa -b 4096
```

На сервере бэкапов:
```bash
mkdir ~borg/.ssh
echo 'command="/usr/bin/borg serve" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDNdaDfqUUf/XmSVWfF7PfjGlbKW00MJ63zal/E/mxm+vJIJRBw7GZofe1PeTpKcEUTiBBEsW9XUmTctnWE6p21gU/JNU0jITLx+vg4IlVP62cac71tkx1VJFMYQN6EulT0alYxagNwEs7s5cBlykeKk/QmteOOclzx684t9d6BhMvFE9w9r+c76aVBIdbEyrkloiYd+vzt79nRkFE4CoxkpvptMgrAgbx563fRmNSPH8H5dEad44/Xb5uARiYhdlIl45QuNSpAdcOadp46ftDeQCGLc4CgjMxessam+9ujYcUCjhFDNOoEa4YxVhXF9Tcv8Ttxolece6y+IQM7fbDR' >> ~borg/.ssh/authorized_keys
chown -R borg:borg ~borg/.ssh
```
Публичный ключ, использовать id_rsa.pub, сгенерированный на предыдущем шаге.

На сервере создать директорию храниения бэкапов:
```bash
mkdir -p /data/borg
chown -R borg:borg /data/borg
```
### Работа с Borg

На клиенте запустить инициализацию хранилища Borg:
```bash
borg init -e none borg@backup_hostname:/data/borg/MyBorgRepo
```
Репозиторий MyBorgRepo Borg создаст автоматически при инициализации.

### Создание бэкапа
```bash
borg create --stats --list borg@backup_hostname:/data/borg/MyBorgRepo::"MyFirstBackup-{now:%Y-%m-%d_%H:%M:%S}" /data/postgresql/pgsql
```
*    --stats и --list дают нам статистику по бэкапу и попавшим в него файлам;
*    borg@backup_hostname:/data/borg/MyBorgRepo —   сервер и каталог.
*    ::"MyFirstBackup-{now:%Y-%m-%d_%H:%M:%S}" —  имя архива внутри репозитория. Имя произвольное. для удобства управления бэкапами применяется наименование  Имя_бэкапа-timestamp (timestamp в формате Python).
    /data/postgresql/pgsql путь до директории которую бэкапим
    
Просмотр списка бэкапов:
```bash
borg list borg@backup_hostname:/data/borg/MyBorgRepo


MyFirstBackup-2019-07-10_17:59:24    Wed, 2019-07-10 17:59:25 [c64ea991b8994552d856e8ac2560651d6ea69f623a12492ca6d0642dc3f17fa3]
MyFirstBackup-2019-07-10_18:27:52    Wed, 2019-07-10 18:27:53 [52368e51224d5f4d1370fc6fb250861f48c7ad1eff304b6dd2cea24b6663bbbf]
MyFirstBackup-2019-07-11_17:12:45    Thu, 2019-07-11 17:12:46 [e8f1f92f62326d3040013d57595815f98cfb344a436f45b0289f7334840c0118]
```
### Восстановление  из бэкапа
```bash
rm -rf /data/postgresql/pgsql 
cd /
borg extract borg@10.10.10.125:/data/borg/pg_backup::MyFirstBackup-2019-07-11_17:12:45
```
Директорию необходимо удалить т.к. Borg если видит что директория не пуста, восстановит лишь недостающие файлы, в случае восстановления БД состав файлов сохраняется и Borg  не восстановит ничего.

Очистка бэкапов:
```bash
borg prune -v --list borg@backup_hostname:/data/borg/MyBorgRepo --keep-daily=7 --keep-weekly=4 --keep-monthly=6
```
Keep 7 daily backups, 4 weekly backups, and 6 monthly ones 
Borg оставляет 7 ежедневных бэкапов, 4 еженедельных, 6 ежемесячных 

## Порядок бэкапирования кластера Patroni

### Снятие бэкапа

Бэкапы снимаются обязательно с ведомой ноды
*    остановить Patroni
*    снять бэкап
*    запустить Patroni

Восстановление из бэкапа

*    остановить Patroni на всех нодах
*    восстановить директорию data из бэкапа
*    запустить Patroni
*    удостовериться что служба поднялась успешно и нода стала master
*    запустить Patroni на остальных нодах, убедиться что изменения корректно отреплицировались
*    рестарт Patroni master (необходимо чтобы он вновь стал secondary)


Хотелось бы все сервисы бекапировать следующим видом (в процессе разработки): 

*    Снимать lvm snapshot
*    Монтировать данный снапшот в файловую систему
*    Снимать бэкап
*    Удалять снапшот
```bash
#!/bin/bash
TIMESTAMP=`date +%Y-%m-%d_%H-%M-%S`
HOST="10.10.10.30" #Your host
lvm snapshot

logpath
mkdir /var/log/borgbackup/


borg init -e none borg@10.10.10.125:/data/borg/Gitlab/
borg create \
  --stats \
  --list \
  borg@10.10.10.125:/data/borg/Gitlab::"$HOST-{now:%Y-%m-%d_%H:%M:%S}" /data/gitlab* \
  2>> /var/log/borgbackup/$TIMESTAMP.log || 
```

### Восстановление

```bash
borg list borg@{{ IP }}:/data/borg/{{ repo }}
cd /
borg extract borg@{{ IP }}:/data/borg/{{ repo }}::{{ BACKUPNAME-2019-12-26T09:20:05 }}
```
