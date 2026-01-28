# Настройка кластера PostgreSQL при помощи patroni etcd, keepalived.
В кластере должно быть минимум 3 ноды. Все компоненты требуется установить и сконфигурировать на всех нодах кластера.

### Установка PostgreSQL
! Тестировалось на версии PostgreSQL 9.6

### Устанавливаем репозиторий PostgreSQL 9.6:
```bash
sudo yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-oraclelinux96-9.6-3.noarch.rpm -y
```
### Устанавливаем пакеты PostgreSQL 9.6:
```bash
sudo yum install postgresql96-server postgresql96-contrib repmgr96 -y
```
Если используете путь хранения базы /data/pgsql/9.6/data, то необходимо заменить его в юнит-файле службы:
```bash
sed -i 's/\/var\/lib\/pgsql/\/data\/pgsql/g' /lib/systemd/system/postgresql-9.6.service
```
Дополнительной настройки PostgreSQL не требуется, так как за конфигурацию PostgreSQL в кластрере отвечает Patroni.

### Установка и настройка Patroni
Patroni — это демон на python, позволяющий автоматически обслуживать кластеры PostgreSQL с различными типами репликации, и автоматическим переключением ролей. Подробная документация доступна по ссылке: https://patroni.readthedocs.io/en/latest/index.html, ссылка на репозиторий кода https://github.com/zalando/patroni

Для поддержания актуальности кластера и выборов мастера используются распределенные DCS хранилища (поддерживаются Zookeeper, etcd, Consul). В нашем примере используется Zookeeper.

### Устанавливаем дополнительные пакеты:
```bash
sudo yum install python-psycopg2 python2-pip python-devel
```
В диструбтивах Oracle довольно старые версии pip и setuptools, последнее придется обновить:
```bash
sudo pip install --upgrade setuptools
```
Для установки patroni c поддержкой Etcd (библиотека python-etcd) выполняем:
```bash
sudo pip install patroni[etcd]
```
Создаём директорию ```/еtс/patroni```:
```bash
sudo mkdir /еtс/patroni
```
После чего копируем в /etc/patroni конфигурационный файл postgres.yml из вложения

В конфигурационном файле меняем на актуальные следующие параметры:
```
name: test-pgsql-01 - Имя ноды, на которой разворачивается patroni
scope: test-cluster - Имя кластера
zookeeper: hosts: ['test-pgsql-01:2181', 'test-pgsql-02:2181', 'test-pgsql-03:2181'] - перечень серверов в кластере zookeeper
restapi: connect_address: 172.16.13.226:8008 - адрес на котором будет поднято rest api
pg_hba: host replication replicator 172.16.13.0/24 md5 - запись в файле pg_hb.conf. На актуальное значение следует поменять сеть из которой разрешены подключения.
postgresql: connect_address: 172.16.13.226:5432 - адрес и порт PostreSQL сервера
max_connection: 300 - увеличиваем лимит подключений со стандартных 100 до 300
```


После настройки конфигурационного файла необходимо в создать в systemd юнит для запуска демона в системе. Для этого копируем в ```/etc/systemd/system/``` файл ```patroni.service```.

После чего выполняем команду:
```bash
sudo systemctl daemon-reload
```
### Установка и настройка Etcd
```bash
sudo yum install etcd
```
Или:
```bash
sudo yum install http://mirror.centos.org/centos/7/extras/x86_64/Packages/etcd-3.2.22-1.el7.x86_64.rpm
```
Настраиваем кластер, приходится использовать IP-адреса, т.к. с DNS-именами данная версия etcd работать не может:
```bash
sudo vim /etc/etcd/etcd.conf

etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.65:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.65:2379,http://127.0.0.1:2379"
ETCD_NAME="test-pgsql-01"
#
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.65:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.65:2379"
ETCD_INITIAL_CLUSTER="test-pgsql-01=http://10.10.10.65:2380,test-pgsql-02=http://10.10.10.66:2380,test-pgsql-03=http://10.10.10.67:2380"
ETCD_INITIAL_CLUSTER_TOKEN="test-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
Перезапускаем службу:

```bash
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/*
sudo systemctl start etcd
```
 

Проверяем состояние кластера etcd:
```bash
sudo etcdctl cluster-health
```
Меняем состояние кластера state="new" на state="existing":
```bash
sudo vim /etc/etcd/etcd.conf
```

### Установка и настройка keepalived
Демон keepalived работает по протоколу vrrp со своими соседями, и в результате выборов одного из демонов как главного (приоритет указан в конфиге), он поднимает у себя виртуальный ip адрес. Демоны keepalived на остальных серверах ждут. Если текущий основной сервер keepalived по какой-то причине умрет либо просигналит соседям аварию, произойдут перевыборы, и следующий по приоритету сервер заберет себе виртуальный ip адрес.
 

Установка Keepalived производится командой:
```bash
sudo yum install keepalived
```


После установки Keepalived следует скопировать в каталог /etc/keepalived/ конфигурационный файл keepalived.conf с содержимым.
```bash
/etc/keepalived/keepalived.conf
vrrp_script chk_patroni {
        script "/usr/local/bin/check_patroni.sh"
        interval 1
        fall 2
        rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 200
    priority 100
    authentication {
        auth_type PASS
        auth_pass *password*
    }
    track_script {
        chk_patroni weight 20
    }
    track_interface {
        ens192
    }
    virtual_ipaddress {
        *VirtualIP*/26
    }
}
```
Вес weight в track_script.chk_patroni должен быть равен или больше количества нод в кластере.



Скрипт опредления является ли postgresql-нода master'ом или же standby:
```bash
/usr/local/bin/check_patroni.sh
#!/bin/bash
 
patroni_status=`/usr/bin/curl -sL -w "%{http_code}\\n" "127.0.0.1:8008" -o /dev/null`
 
if [ $patroni_status -eq 200 ]; then
exit 0
elif [ $patroni_status -eq 503 ]; then
exit 1
fi
```

В конфигурационном файле параметр virtual_router_id должен быть общим для всех нод кластера, но уникален внутри одной сети. Также едиными для всех должны быть и параметры auth_type, auth_pass в секции authentication. 

Параметр priority должен быть уникальным для каждой ноды кластера. Чем выше он, тем выше приоритет у сервера при выборе главного (на котором висит виртуальный IP).

Запуск кластера
После установки и настройки всех вышеупомянутых компонентов можно приступать к запуску кластера.

Запускаем etcd, keepalived. После них запускаем patroni.

Перед первым  запуском Patroni  необходимо очистить директорию /data/pgsql/*

При запуске Patroni будет создана пустая база. Мастером просто станет первый сконфигурированный сервер. При добавлении новых серверов Patroni увидит через Etcd, что мастер кластера уже есть, автоматически скопирует с текущего мастера базу, и подключит к нему slave.

Посмотреть статус кластера можно при помощи команды: 
```bash
patronictl -c /etc/patroni/postgres.yml list test-cluster
```
В редких случаях бывает что slave не может догнаться до состояния мастера (например он упал очень давно и wal журнал успел частично удалиться). В таком случае нужно воспользоваться утилитой patronictl. Команда reinit позволяет безопасно очистить конкретный узел кластера, на мастере она выполняться не будет.

Отладка
Если что-то пошло не так, то лучше попробовать отладить запуск кластера.

Перед запуском желательно предварительно все почистить:
```bash
sudo etcdctl rm -r /service/
sudo rm -rf /data/pgsql/*
```
Оставим службу и запустим Patroni вручную:
```bash
sudo systemctl stop patroni
sudo su - postgres
source /etc/patroni/env.conf
PATRONI_LOGLEVEL=DEBUG /bin/patroni /etc/patroni/postgres.yml
```
После запуска можно смотреть значение в Etcd:
```bash
sudo etcdctl ls -r
sudo etcdctl get /service/test-cluster/initialize
```
