# Перемещение хранилища Docker

По умолчанию, сервис Docker складирует все свои данные в ```/var/lib/docker```
Поскольку, данных может быть много, а раздел /var небольшим, у Вас может возникнуть необходимость переместить этот раздел в другое место.

### Перенос

На данном примере мы будем переносить ```/var/lib/docker``` в ```/data/docker```
На время переноса docker будет недоступен.

Первым делом, пропишите новый путь в настройки сервиса в systemd:

```bash
# *docker 17.XX*
vim /etc/systemd/system/multi-user.target.wants/docker.service
...
ExecStart=/usr/bin/dockerd -g /data/docker -H fd://
...
# *docker 18.XX*

vim /etc/systemd/system/multi-user.target.wants/docker.service
...
ExecStart=/usr/bin/dockerd --data-root /data/docker -H fd://
...
```

После остановите сервис и примите новые настройки systemd:
```bash
systemctl stop docker
systemctl daemon-reload
```
Далее займёмся переносом и, по завершению, запустим сервис обратно
```bash
mkdir /data/docker
rsync -aqxP /var/lib/docker/ /data/docker && systemctl start docker
```

После успешного копирования сервис запустится сам.
