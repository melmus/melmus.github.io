# Установка AWX из официального репозитория в GitHub на docker-контейнерах
Установка

*Опционально - Пропишите Proxy*
```bash
export http_proxy="http://login:password@proxy:3128"
export https_proxy="https://login:password@proxy:3128"
export ftp_proxy="ftp://login:password@proxy:3128"
```

Добавьте в docker-репозитории и запустите установку всех нужных приложений
```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum -y install epel-release 

## Oracle Linux ##
yum -y install oracle-epel-release-el7.x86_64
/usr/bin/ol_yum_configure.sh
##
  
yum -y install git gcc gcc-c++ lvm2 bzip2 gettext nodejs yum-utils device-mapper-persistent-data ansible python-pip http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-1.el7_6.noarch.rpm docker-ce wget
pip install --upgrade pip
pip uninstall docker docker-py docker-compose
pip install docker-compose
```

Пропишите Proxy в Docker-сервис и запустите его
```bash
mkdir -p /etc/systemd/system/docker.service.d
vim /etc/systemd/system/docker.service.d/https-proxy.conf
[Service]
Environment="HTTP_PROXY=http://login:password@bproxy.nw.mts.ru:3131/"
Environment="HTTPS_PROXY=http://login:password@bproxy.nw.mts.ru:3131/"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,registry.bss.ural.mts.ru"

systemctl daemon-reload && systemctl enable docker && systemctl start docker
```
Скачайте и распакуйте интересующую версию AWX на https://github.com/ansible/awx/releases
```bash
cd /opt
wget https://github.com/ansible/awx/archive/6.1.0.tar.gz
tar xf *0.tar.gz
mv awx-* awx
rm -f *0.tar.gz
cd awx/installer/
```
Поменяйте дефолтные параметры от AWX
```bash
*Задание пароли и генерация секрета*
sed -i 's/admin_password=password/admin_password=*PASSWORD*/g' inventory
sed -i "s/docker_compose_dir=\/tmp\/awxcompose/docker_compose_dir=\/opt\/awx\/awxcompose/g" inventory
sed -i "s/secret_key=awxsecret/secret_key=`openssl rand -base64 30`/g" inventory
sed -i "s/host_port=80/host_port=80/g" inventory
*Место хранения DB*
sed -i "s/postgres_data_dir=\/tmp\/pgdocker/postgres_data_dir=\/data\/awx/g" inventory
mkdir -p /data/awx

*Опционально - задайте прокси*
echo 'http_proxy=http://<LOGIN>:<PASSWORD>@proxy:3128' >> inventory
echo 'https_proxy=http://<LOGIN>:<PASSWORD>@proxy:3128' >> inventory
echo 'no_proxy=localhost' >> inventory
```
Запустите установку
```bash
ansible-playbook -i inventory install.yml
```
По окончании установки откройте в Firewall'е порт 80
```bash
firewall-cmd --add-port=80/tcp
firewall-cmd --add-port=80/tcp --permanent
```

Делаем AWX в виде сервиса
```bash
cat > /etc/systemd/system/awx.service << END
[Unit]
Description=AWX Service
Requires=docker.service
After=docker.service
 
[Service]
Restart=always
WorkingDirectory=opt/awx/awxcompose
ExecStart=/usr/bin/docker-compose -f /opt/awx/awxcompose/docker-compose.yml up
ExecStop=/usr/bin/docker-compose -f /opt/awx/awxcompose/docker-compose.yml down
 
[Install]
WantedBy=multi-user.target
END
```
Включаем сервис
```bash
systemctl daemon-reload
systemctl enable awx
```
