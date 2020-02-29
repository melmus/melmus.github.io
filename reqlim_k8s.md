# Request and Limits Kubernetes

Данная статья предназначена для понимания принципа и приёмов использования лимитов в Kubernetes
## Теория

Лимиты в Kubernetes делятся на 2 вида request и limits. При этом каждый вид можно ограничить по CPU(ядра процессора), RAM(байты оперативной памяти). Отдельно стоит ограничение Storage(байты жесткого диска) - его мы рассмотрим отдельно..
### Requests

**Requests** - задание количество ресурсов, сколько нужно гарантировать контейнеру для работы приложения.

    Полностью управляется Kubernetes
    Работает до запуска контейнера
    Задаётся вручную и не может быть превышено заданным значением в Namespace'е или автоматически расчитанным в кластере
    В Docker-сервисе никак не участвует.
    Не следит за фактическим потребление ресурсов контейнером.

Kubernetes ВСЕГДА имеет некие параметры Request CPU и Request RAM. Их можно задать явно для определённого Namespace'а, но по умолчанию они ровны сумме всех CPU и RAM Worker'ов кластера. Kubernetes не сможет развернуть новые контейнеры, если для них не хватает квоты, однако можно (и нужно) превышать данные установленные лимиты. Лучше указывать здесь маленькие лимиты, чтобы не столкнуться с ситуацией, что запущенные контейнеры не потребляют много CPU и RAM, а новые уже отказываются запусткаться,
### Limits

**Limits** - задание максимального объёма ресурсов, который контейнер может потреблять в штатной работе. Контейнер не сможет превысить заданные тут значения ресурсов: потребление CPU и RAM не подниматься выше заданных параметров.

    Полностью управляется Docker-сервисом (Kubernetes просто передаёт эти параметры Docker-сервису при запуске).
    Фактическое потребление контейнером ресурсов не может быть превышено заданным тут параметрам

Docker умеет ограничивать ресурсы контейнера, при этом стараясь поддерживать жизнь в контейнере. То есть, если контейнер достигнет лимита по CPU, все запущенные процессы начнут делить немногие ядра для своих вычислений, а если достигнет лимита по RAM - запущенные и новые процессы не смогут больше выделить себе ОЗУ (ООМ не будет, но стабильность работы приложения будет нарушено).

Значение Limits может (и должно) быть большим. Возможно даже превышение общего объёма ресурсов во всём кластере, но под свой страх и риск (если ресурсы RAM на кластере кончаться раньше, чем сработает данный лимит - кластер будет медленно умирать от OOM).
## Практика

Имеется кластер Kubernetes 1.12.7 и контейнер со следующими лимитами:
 	Request	Limits
CPU	100m	512Mi
RAM	2	2048Mi

Yaml:
```yaml
---
apiVersion: apps/v1
kind: Deployment # Вид сущности Kubernetes (краткое описание ниже, там же ссылка на полное)
metadata:
  name: sshd  # Название микросервиса - именно по нему будет происходить обращение к данному микросервису от других микросервисов
  namespace: sfo-hightest  # Название Namespace - обязательно определяйте его в каждом yaml-файле
spec:
  replicas: 1 # Количество экземпляров контейнеров в микросервисе (описание ниже)
  selector:
    matchLabels:
      name: sshd # Ярлык - привязка одних видов сущностей Kubernetes к другим. Чтобы не запутаться - пишите его одноимённо с metadata.name  
  template:
    metadata:
      labels:
        name: sshd # Дополнительные к metadata.name имена микросервиса. В нашем примере не используется, поэтому просто укажем такое же название.
    spec:
      containers: # Данные о запускаемом конетейнере
      - name: sshd # Название контейнера в запущенном состоянии - чтобы не путаться, оставляем одноимённо с metadata.name  
        image: rastasheep/ubuntu-sshd:18.04 # Название и тег контейнера - его мы нашли на hub.docker.com
        ports:
        - containerPort: 22 # Рабочий порт контейнера
        resources: # Лимиты памяти и CPU - всегда знайте свои лимиты на Namespace и распределяйте их на микросервисы в этих пределах (см. ниже)
          requests:
            memory: "512Mi"
            cpu: "100m"
          limits:
            memory: "2048Mi"
            cpu: "2"
      hostAliases:
      - ip: "127.0.0.1"
        hostnames:
          - "sshd" # Пропишите возможность обращаться контейнерам сами на себя (localhost)

---
apiVersion: v1
kind: Service
metadata:
  name: sshd # Название сервиса
  labels:
    name: sshd # Ярлык - привязка в Deployment "sshd"
  namespace: sfo-hightest
spec:
  type: NodePort
  ports:
  - port: 22 # Указываем рабочие порты контейнера
    targetPort: 22
  selector:
    name: sshd # Ярлык - привязка в Deployment "sshd"
```
 
### Состояние простоя

В состоянии простоя мы можем увидеть следующие данные:

*Статистика относительная Docker*
```bash
docker stats
CONTAINER ID        NAME                                                                                        CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
bd9af9612d23        k8s_sshd_sshd-5c64f7646f-l44jj_sfo-hightest_a416b450-61f9-11e9-a907-0050562f04b1_0          0.00%               3.371MiB / 2GiB       0.16%               0B / 0B             0B / 0B             1
```
*Статистика относительно Namespace'а
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used   Hard
 --------         ---    ---
 limits.cpu       2      16
 limits.memory    2Gi    64Gi
 requests.cpu     100m   2
 requests.memory  512Mi  4Gi
```
*Статистка относительно Ноды в kubernetes*
```bash
kubectl describe nodes dev-wrk02
  Namespace                  Name                     CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                     ------------  ----------  ---------------  -------------
  sfo-hightest               sshd-5c64f7646f-l44jj    100m (1%)     2 (25%)     512Mi (0%)       2Gi (3%)
```

*Статистика относительно Ноды в ОС*
```bash
ps auxf
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      93195  0.1  0.0 1247552 53868 ?       Ssl  апр08  15:41 /usr/bin/containerd
root       7317  0.0  0.0 451444 11760 ?        Sl   19:48   0:00  \_ containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/bd9af9612d233c6910983a7
root       7334  0.0  0.0  72296  6540 ?        Ss   19:48   0:00      \_ /usr/sbin/sshd -D
```
Как видите, контейнер забрал себе 0.1 CPU и 512Mi RAM request-лимита, но имеет запас лимита в 2 CPU и 2Gi RAM hard-лимита (limits).

При этом видно, что request ограничивает сам Kubernetes, а Limits - сам docker-сервис.
## Тестируем Limits
### Нагрузка по CPU

Попробуем нагрузить все 2 ядра - запустим 4 ресурсоёмких процесса.
```bash
ssh -p 30332 root@k8s.testdomain.com
root@k8s.testdomain.com's password: 
root@sshd-5c64f7646f-l44jj:~# yes > /dev/null & 
[1] 129
root@sshd-5c64f7646f-l44jj:~# yes > /dev/null & 
[2] 130
root@sshd-5c64f7646f-l44jj:~# yes > /dev/null & 
[3] 131
root@sshd-5c64f7646f-l44jj:~# yes > /dev/null & 
[4] 132
root@sshd-5c64f7646f-l44jj:~# 
```

Смотрим статистику

*Статистика относительная Docker*
```bash
docker stats
CONTAINER ID        NAME                                                                                        CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
bd9af9612d23        k8s_sshd_sshd-5c64f7646f-l44jj_sfo-hightest_a416b450-61f9-11e9-a907-0050562f04b1_0          197.68%             8.305MiB / 2GiB       0.41%               0B / 0B 
```

*Статистика относительно Namespace'а
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used   Hard
 --------         ---    ---
 limits.cpu       2      16
 limits.memory    2Gi    64Gi
 requests.cpu     100m   2
 requests.memory  512Mi  4Gi
```

*Статистка относительно Ноды в kubernetes*
```bash
kubectl describe nodes dev-wrk02
  Namespace                  Name                     CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                     ------------  ----------  ---------------  -------------
  sfo-hightest               sshd-5c64f7646f-l44jj    100m (1%)     2 (25%)     512Mi (0%)       2Gi (3%)
```

*Статистика относительно Ноды в ОС*
```bash
ps auxf
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root       7317  0.0  0.0 451444 11792 ?        Sl   19:48   0:00  \_ containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/bd9af9612d233c6910983a7
root       7334  0.0  0.0  72296  6540 ?        Ss   19:48   0:00      \_ /usr/sbin/sshd -D
root      12938  0.0  0.0  74656  6684 ?        Ss   20:10   0:00          \_ sshd: root@pts/0
root      12942  0.0  0.0  18508  3536 pts/0    Ss+  20:10   0:00              \_ -bash
root      14217 53.7  0.0   4532   744 pts/0    R    20:16   0:36                  \_ yes
root      14226 53.7  0.0   4532   828 pts/0    R    20:16   0:34                  \_ yes
root      14234 49.9  0.0   4532   772 pts/0    R    20:16   0:30                  \_ yes
root      14242 50.1  0.0   4532   768 pts/0    R    20:16   0:30                  \_ yes
```
Зафиксировали, отпускаем нагрузку
```bash
root@sshd-5c64f7646f-l44jj:~# kill 132 131 130 129
```
### Нагрузка по RAM

Загружает всё RAM (приложение сразу же сообщает о недостатке ОЗУ)
```bash
root@sshd-5c64f7646f-l44jj:~# :(){ :|:& };:
[1] 243
root@sshd-5c64f7646f-l44jj:~# -bash: fork: Cannot allocate memory
-bash: fork: Cannot allocate memory
-bash: fork: Cannot allocate memory
-bash: fork: Cannot allocate memory
-bash: fork: Cannot allocate memory
```
Смотрим статистику

*Статистика относительная Docker*
```bash
docker stats
CONTAINER ID        NAME                                                                                        CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
bd9af9612d23        k8s_sshd_sshd-5c64f7646f-l44jj_sfo-hightest_a416b450-61f9-11e9-a907-0050562f04b1_0          201.47%             1.999GiB / 2GiB       99.94%              0B / 0B             0B / 2MB            12840
```

*Статистика относительно Namespace'а
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used   Hard
 --------         ---    ---
 limits.cpu       2      16
 limits.memory    2Gi    64Gi
 requests.cpu     100m   2
 requests.memory  512Mi  4Gi
```

*Статистка относительно Ноды в kubernetes*
```bash
kubectl describe nodes dev-wrk02
  Namespace                  Name                     CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                     ------------  ----------  ---------------  -------------
  sfo-hightest               sshd-5c64f7646f-l44jj    100m (1%)     2 (25%)     512Mi (0%)       2Gi (3%)
```
*Статистика относительно Ноды в ОС*
```bash
ps auxf
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root       7317  0.0  0.0 451444 11760 ?        Sl   19:48   0:00  \_ containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/bd9af9612d233c6910983a7
root       7334  0.0  0.0  72296  6540 ?        Rs   19:48   0:00      \_ /usr/sbin/sshd -D
root      12938  0.0  0.0  74656  6684 ?        Ss   20:10   0:00          \_ sshd: root@pts/0
root      12942  0.0  0.0      0     0 pts/0    Rs+  20:10   0:00          |   \_ [bash]
root      15643  0.0  0.0  74656  6744 ?        Ss   20:22   0:00          \_ sshd: root@pts/1
root      15677  0.0  0.0  18508  3464 ?        Ss+  20:22   0:00          |   \_ -bash
root      20390  0.0  0.0  18628  2480 pts/0    R    20:28   0:00          \_ -bash
root      26583  0.0  0.0  18628   512 pts/0    R    20:28   0:00          |   \_ -bash
root      23760  0.0  0.0  18628  1980 pts/0    D    20:28   0:00          \_ -bash
root      24163  0.0  0.0  18628   500 pts/0    R    20:28   0:00          \_ -bash
root      22330  0.0  0.0  18628  1920 pts/0    R    20:28   0:00          \_ -bash
root      22554  0.0  0.0  18628  1788 pts/0    R    20:28   0:00          \_ -bash
root      31242  0.0  0.0  18628   508 pts/0    R    20:28   0:00          |   \_ -bash
root      22592  0.0  0.0  18628  2112 pts/0    R    20:28   0:00          \_ -bash
root      22614  0.0  0.0  18628  1980 pts/0    R    20:28   0:00          \_ -bash
root      22684  0.0  0.0  18628  1792 pts/0    R    20:28   0:00          \_ -bash
root      29300  0.0  0.0  18628   512 pts/0    D    20:28   0:00          |   \_ -bash
root      22703  0.0  0.0  18628  1788 pts/0    R    20:28   0:00          \_ -bash
root      29520  0.0  0.0  18628   508 pts/0    R    20:28   0:00          |   \_ -bash
root      22880  0.0  0.0  18628  1920 pts/0    R    20:28   0:00          \_ -bash
root      25617  0.0  0.0  18628   500 pts/0    R    20:28   0:00          \_ -bash
root      23025  0.0  0.0  18628  2280 pts/0    R    20:28   0:00          \_ -bash
root      30050  0.0  0.0  18628   512 pts/0    R    20:28   0:00          |   \_ -bash
root      30054  0.0  0.0  18628   512 pts/0    R    20:28   0:00          |   \_ -bash
root      23033  0.0  0.0  18628  1788 pts/0    R    20:28   0:00          \_ -bash
root      31935  0.0  0.0  18628   508 pts/0    R    20:28   0:00          |   \_ -bash
```
### Подведение итогов

Как видите, сам Kubernetes просто передаёт все лимиты в Docker и он уже самостоятельно следит, чтобы они не были привышены.

Таким образом, превышение CPU выше установленного невозможно и приложения будут довольствоваться только имеющимся лимитами CPU

В свою очередь, превышение RAM не позволит больше выделять памяти новым или существующим процессам, однако приложения, которые успели отхватить себе кусок ОЗУ работают штатно.
Тестируем Requests

Requests никак не повлияют на уже запущенный контейнер, однако могут очень сильно повлиять на момент запуска контейнера (т.н. планирование).

Приведём пример, у нас есть один Namespace, ограниченный по CPU и RAM.
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used   Hard
 --------         ---    ---
 limits.cpu       2      16
 limits.memory    2Gi    64Gi
 requests.cpu     100m   2
 requests.memory  512Mi  4Gi
```
То есть мы имеет request лимит в 2 CPU и 4Gi RAM

Если вы не используйте лимиты в Namespace'ах, то за Request-лимит будет браться сумма всех CPU и RAM(включая SWAP) кластера. То есть, если вы имеете 3 Worker'а по 4 CPU и 16Gi RAM - Ваш request-лимит будет задан на 12 CPU и 48Gi RAM, даже если Вы сами об этом не знаете.

Попробуем увеличить request-потребление - запустим 3 контейнера с большими request-лимитами
```bash
vim sshd.yml 
---
...
  replicas: 3
...
        resources: 
          requests:
            memory: "1000Mi"
            cpu: "500m"
          limits:
            memory: "12048Mi"
            cpu: "2"
...

```
```bash
kubectl apply -f sshd.yml 
```
Смотрим
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used     Hard
 --------         ---      ---
 limits.cpu       6        16
 limits.memory    36144Mi  64Gi
 requests.cpu     1500m    2
 requests.memory  3000Mi   4Gi

kubectl -n sfo-hightest get pods
NAME                    READY   STATUS    RESTARTS   AGE
sshd-778fb9948d-krz2d   1/1     Running   0          3m50s
sshd-778fb9948d-m7sxq   1/1     Running   0          3m50s
sshd-778fb9948d-vblnb   1/1     Running   0          3m50s
```
3 контейнера запущены. Используемые request-лимиты не привысили заданных лимитов кластере, поэтому всё успешно запустилось. Тут каждый контейнер запросил у Kubernetes по 0.5 CPU и 1Gi RAM

Запустим 4 контейнера
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used     Hard
 --------         ---      ---
 limits.cpu       8        16
 limits.memory    48192Mi  64Gi
 requests.cpu     2        2
 requests.memory  4000Mi   4Gi

kubectl -n sfo-hightest get pods
NAME                    READY   STATUS    RESTARTS   AGE
sshd-778fb9948d-krz2d   1/1     Running   0          5m56s
sshd-778fb9948d-m7sxq   1/1     Running   0          5m56s
sshd-778fb9948d-vblnb   1/1     Running   0          5m56s
sshd-778fb9948d-zbnvz   1/1     Running   0          51s
```
Request достигли лимитов, но не привысили их, а значит 4-ый под доразвернулся.

Пробуем 5 контейнеров
```bash
kubectl describe namespaces sfo-hightest 
Resource Quotas
 Name:            sfoteam-quota
 Resource         Used     Hard
 --------         ---      ---
 limits.cpu       8        16
 limits.memory    48192Mi  64Gi
 requests.cpu     2        2
 requests.memory  4000Mi   4Gi

kubectl -n sfo-hightest get pods
NAME                    READY   STATUS    RESTARTS   AGE
sshd-778fb9948d-krz2d   1/1     Running   0          7m23s
sshd-778fb9948d-m7sxq   1/1     Running   0          7m23s
sshd-778fb9948d-vblnb   1/1     Running   0          7m23s
sshd-778fb9948d-zbnvz   1/1     Running   0          2m18s
```
Ничего не изменилось. Случилось это потому-что для нового контейнера не нашлось свободных ресурсов и Kubernetes ему просто нигде не разворачивает. При этом по факту limits-ресурсов (и тем более самого кластера) ещё достаточно.

При этом нигде явно не сообщается, что "у Вас превышение кворы и поэтому мы Вам 5-ый под не развернём" - просто разворачивает столько контейнеров, сколько получится и всё.
