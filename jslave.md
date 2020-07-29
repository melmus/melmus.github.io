*Прописываем пароль в контейнер*
```bash
sudo docker exec -it jenkins-${COM} /bin/sh -c "echo 'jenkins:${JENKINS_PASSWORD}' | chpasswd"
```
Если не используем переменную JENKINS_PASSWORD
```bash
sudo docker exec -it jenkins-${COM} /bin/sh -c 'echo "jenkins:PASSWORD" | chpasswd' memo--cpus=1.0 --memory=2gry=2g
```

Именование

VM_INDEX - индекс сервера
C_INDEX - индекс контейнера
teamName - название группы в LDAP

Docker container name - jenkins-(?P<teamName>[a-z][a-z0-9-]*)


jenkins-slave01.example.com
```bash
jenkins-slave01   2222	--cpus=1.0 --memory=3g
jenkins-slave02   2223	--cpus=1.0 --memory=3g
jenkins-slave03	  2224	--cpus=4.0 --memory=10g
jenkins-slave04   2225	--cpus=1.0 --memory=2g
jenkins-slave05	  2226	--cpus=2.0 --memory=5g
jenkins-slave06   2227	--cpus=1.0 --memory=2g
```

Запуск контейнера

Для создания нового Jenkins-Slave'а необходимо определить следующие параметры:
```
* Выбрать один из физических серверов для разворачивания контейнера - выбирается наименее загруженный сервер по лимитам в таблице выше.
* Определить количество сборщиков Jenkins-слейва - по 1 сборщику на 2 микросервиса
* Определить лимиты контейнера по CPU/RAM - по 1 CPU на 2 сборщика и по 2ГБ на 1 сборщик (используются только целые числа, округлённые в большую сторону)
* Название слейва в латинице в нижнем регистре
* Свободный порт для вывода подключения к контейнеру
* Логин/пароль от сервисной учётной записи слейва
```
Если контейнер для команды уже существует, но необходимо просто изменить параметры - перемещать контейнер на другой сервер не нужно.
Однако процесс установки контейнера не изменяется.

После того, как вся информация будет известна - можно приступать к настройке.

Проект: Demo

Сервер: jenkins-slave01.example.com

Сборщиков: 1

Лимиты: 1 СPU и 2ГБ RAM

Название слейва: demo

Порт: 2225

Подключитесь по SSH к выбранному серверу

ssh user@jenkins-slave01.example.com

Далее задайте значения параметров контейнера

*Название сборщика(слейва) - переменная COM*
*Порт контейнера*
*Лимит по CPU*
*Лимит по RAM*

```bash
COM=demo 
PORT=2225
CPU=1
RAM=2
```

(!! Шаг пропускается, если контейнер уже существует !!) Далее необходимо прописать сервисную учётную запись в файлы (в данном примере, учётная запись service_jenkins)

```bash
mvn --encrypt-master-password
Master password:
```

(!! Шаг пропускается, если контейнер уже существует !!) Занесите полученный мастер-кэш в файл

```bash
sudo vim /data/jenkins/.m2/settings-security.xml
<settingsSecurity>
  <master>*МАСТЕР-КЭШ*</master>
</settingsSecurity>
```
(!! Шаг пропускается, если контейнер уже существует !!) Далее, на основании мастер-кэша, сгеренируйте кэш основного пароля
```bash
sudo -u jenkins sh -c 'mvn --encrypt-password'
Password: *service_jenkins_password*
*Сгененируется кэш*
```

(!! Шаг пропускается, если контейнер уже существует !!) Теперь пропишите все логин и полученный кэш от пароля в замен старых
```bash
sudo vim /data/jenkins/.m2/settings.xml
...
  <proxies>
    <proxy>
    <active>true</active>
    <protocol>http</protocol>
    <host></host>
    <port></port>
    <username>service_jenkins</username>
    <password>*КЭШ*</password>
    <nonProxyHosts></nonProxyHosts>
    </proxy>
  </proxies>
  <servers>
    <server>
      <id>nvgrep1</id>
      <username>service_jenkins</username>
      <password>*КЭШ*</password>
    </server>
    <server>
      <id>nvgproto</id>
      <username>service_jenkins</username>
      <password>*КЭШ*</password>
    </server>
    <server>
      <id>docker-registry</id>
      <username>service_jenkins</username>
      <password>*КЭШ*</password>
    </server>
...

(!! Шаг пропускается, если контейнер уже существует !!) Прописываем логин/пароль (уже в без кэширования) от сервисной учётной записи в docker. В поле auth результат вывода команды base64
```bash
echo 'service_jenkins:*Пароль*' | base64
c2EwNzTgshfzSKL9aPAlY29tMTc6cGFzc3dvcmQ=

sudo vim /data/jenkins/.docker/config.json
...
{
        "auths": {
                "registry.example.com": {
                        "auth": "c2EwNzAwbmdzZXJ2aWNlY29tMTc6cGFzc3dvcmQ="
                }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/18.09.1-ol (linux)"
        },
        "proxies": {
                "default": {
                        "httpProxy": "http://",
                        "httpsProxy": "http://",
                        "noProxy": "localhost,127.0.0.1"
                }
        }
}
```
...

(!! Шаг пропускается, если контейнер уже существует !!) Создаём каталог для команды и переместите туда стандартный набор файлов

```bash
sudo mkdir -p /data/jenkins-${COM}
sudo cp -r /data/jenkins/.m2 /data/jenkins-${COM}
sudo cp -r /data/jenkins/.docker /data/jenkins-${COM}
sudo chown -R 10028:10000 /data/jenkins-${COM}/.
sudo chmod 770 /data/jenkins-${COM}
```

Запустите команду создания/пересоздания контейнера

```bash
sudo docker rm -f jenkins-${COM}; sudo docker pull registry.example.com/jenkins-ssh-slave && sudo docker run --name jenkins-${COM} --restart unless-stopped --cpus=${CPU}.0 --memory=${RAM}g -p ${PORT}:22 --volume /data/jenkins-${COM}/:/data/jenkins/ -d registry.example.com/jenkins-ssh-slave
```

Далее, задайте пароль в контейнере (аналогичный с технологической учётной записью)

Внимание! В пароле все знаки "$" должны экранировать с помощь знака "\" перед символом

Например, пароль 1:21!!3Ase$wFFda будет выглядить как 1:21!!3Ase\$wFFda

```bash
sudo docker exec -it jenkins-${COM} /bin/sh -c 'echo "jenkins:*PASSWORD*" | chpasswd'
```

Вместо PASSWORD указываете пароль от service_jenkins без * звёздочек

Подключаем контейнеры к Jenkins
   
Подключение Jenkins-slave'а к Мастеру

Аутентифицируйтесь в Jenkins и перейдите в http://jenkins.example.com/computer/
    Нажмите на ссылку "Новый узел"
    
    Введите название узла (полное название команды с большой буквы) и выберите "Permanent Agent" на переключателе. Нажмите кнопку "ОК"
    На появившейся странице заполните следующие данные:
      *  Имя: Название слейва
      *  Описание: Адрес машины, где располагается docker-конейнер и его порт через пробел
      *  Количество процессов-исполнителей: *количество сборщиков*
      *  Корень удалённой ФС: /data/jenkins/
      *  Метки: Название слейва
      *  Способ запуска "Launch slave agent via SSH"
      *  Host: <адрес машины, где располагается наш docker-контейнер>
      *  Credentials: *создадим новый, см. ниже*
      *  Host Key Verification Strategy: Non verifying Verification Strategy
      *  Post (скрыто за кнопкой "Расширенные..."): Рабочий порт Вашего контейнера и JavaPath "/usr/bin/java"
      *  В разделе "Node properties" установите и заполните 2 следующие галочки
      *  В "Environment variables" добавьте и заполните переменную "DOCKER_HOST" значением "tcp://registry.example.com:23760"
      *  В "Restrict jobs execution at node" в выпадающем списке выберите "Regular Expression (Job Name)" и заполните Expression в формате "<название слейва>-.*"

Отдельно занесите новый Credentials кнопкой Add:
       * Username "jenkins"
       * Password "*service_jenkins_password*"
       * Description "Node<#слейва>"

По окончанию нажмите кнопку "Add", выберите новосозданный Credential и "Save"
После этого подождите, пока Slave подключится и станет готов принимать задание
