# Установка Docker на Linux

Удалите уже установленные старые версии docker
```bash
# Ubuntu, Debian
apt-get remove docker docker-engine docker.io
# CentOS, Fedora
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine
```
Настройте прокси-сервер для доступа к репозиториям
В случае наличия в пароле специальных символов (таких, как #,!,$...) необходимо использовать Unicode

https://en.wikipedia.org/wiki/Basic_Latin_(Unicode_block)#Table_of_characters

Например, пароль F@o:o!B#ar$ таким образом превращается в F%40o%3Ao%21B%23ar%24

```bash
# Настройки прокси работают только в текущей сессии и удаляются из истории
export http_proxy="http://login:password@proxy:3128"
export https_proxy="https://login:password@proxy:3128"
export ftp_proxy="ftp://login:password@proxy:3128"
hictory -c
```
Обновите  Вашу систему
```bash
# Ubuntu, Debian
apt -y update && apt -y upgrade
# CentOS, Fedora
yum -y update
```
Добавьте Docker-репозиторий в систему
```bash
# Ubuntu
apt -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
# Debian
apt -y install apt-transport-https ca-certificates curl python-software-properties software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
# CentOS
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager  --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# Fedora
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
```
Установите Docker
```bash
# *Ubuntu, Debian*
apt -y update && apt -y install docker-ce
# CentOS, Fedora
yum -y install docker-ce
```
Запустите Docker-сервис
```bash
# CentOS, Fedora
systemctl restart docker
systemctl enable docker
```
Добавьте Вашего пользователя в группу docker (для возможности взаимодействия с docker не только из-под root)
```bash
usermod -aG docker YOU_USER
```
*Новая группа (вместе с правами) примется к пользователю после перелогирования*

Проверяем работу Docker
```bash
# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
9bb5a5d4561a: Pull complete 
Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
### Проблемы и решения

#### Проблема 1 - docker: error pulling image configuration

При скачивании/запуске образа возникает следующая ошибка:
```bash
docker run hello-world

Unable to find image 'hello-world:latest' locally
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp 52.22.181.254:443: getsockopt: connection refused.
See 'docker run --help'.
# *** или ***
docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
docker: error pulling image configuration: Get https://production.cloudflare.docker.com/registry-v2/docker/registry/v2/blobs/sha256/e3/e38bc07ac18ee64e6d59cf2eafcdddf9cec2364dfe129fe0af75f1b0194e0c96/data?verify=1525343116-YV1QSRNadfQrFpKjZ0JOwBuSqT0%3D: dial tcp 104.18.125.25:443: i/o timeout.
See 'docker run --help'.
```
В этом случае docker не имеет настройки Proxy и пытается соединиться с сервером напрямую.

Их необходимо прописать в виде переменных в сервис docker
```bash
export http_proxy="http://login:password@proxy:3128"
export https_proxy="https://login:password@proxy:3128"
export ftp_proxy="ftp://login:password@proxy:3128"
hictory -c
systemctl restart docker

docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
Поскольку мы запустили сервис docker с уже экспортированными параметрами - соединение с сервером образов (registry) будет работать до первого перезапуска сервиса (включая путём перезагрузки сервера). Это не работает на Debian - он не принимает экспортируемые переменные на сервис и поэтому данные переменные необходимо прописывать при каждой открытие сессии.

Внести данные настройки навсегда можно следующим образом (делайте это, если Вы уверены, что на вашем сервере Вы единственный пользователь - иначе есть риск раскрытия пароля):
```bash
mkdir -p /etc/systemd/system/docker.service.d
vim /etc/systemd/system/docker.service.d/https-proxy.conf

[Service]
Environment="HTTP_PROXY=http://login:password@proxy:3128/"
Environment="HTTPS_PROXY=http://login:password@proxy:3128/" 
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8"
```
```bash
systemctl daemon-reload
systemctl restart docker
```
```bash
docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
9bb5a5d4561a: Already exists 
Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
#### Проблема 2 - x509: certificate signed by unknown authority или http: server gave HTTP response to HTTPS client

При попытке скачать образ с приватного репозитория возникает следующая ошибка:
```bash
docker pull registry.bss.ural.mts.ru/ngenie/ngenie-security:latest 
Error response from daemon: Get https://registry.bss.ural.mts.ru/v2/: x509: certificate signed by unknown authority
# ИЛИ
Error response from daemon: Get https://registry.bss.ural.mts.ru/v2/: http: server gave HTTP response to HTTPS client
```
Данная ошибка связана с проверкой сертификата регистра, что внутри локальной сети не имеет смысла.

Отключить данную проверку (только для этого регистра) можно следующим образом
```bash
vim /etc/docker/daemon.json
{ "insecure-registries": ["hub.docker.com"] }
systemctl restart docker

docker pull hub.docker.com/nginx/nginx:latest 
latest: Pulling from nginx
2595db787577: Pull complete 
0a77c5e9abd1: Pull complete 
e08a6373470f: Pull complete 
775a40b57085: Pull complete 
f434206d4264: Pull complete 
382757a6bbee: Pull complete 
13b677e0f65e: Pull complete 
e9c3be095189: Pull complete 
Digest: sha256:c02e5b889676fec7eca26aa0b24d5b9d020e818650cd1a9425668001f7cbe0b0
Status: Downloaded newer image for hub.docker.com/nginx/nginx:latest 
```
#### Проблема 3 - 407 proxy authentication required

Часто встречается сообщение о ошибке (в разных вариациях).
```407 proxy authentication required```

Данное сообщение говорит о том, что соединение с прокси-сервером имеется, но указаны неверные логин и/или пароль. Проверьте корректность его написания в переменных и запустите программу/сервис заново. Так-же не забывайте о том, что спецсимволы в пароле нужно менять на unicode (см. выше).
