# Helm в определенном namespace'e


Здесь описано работа с helm в пределах одного namespace.

Сначала будет показано создание аккаунта для Tiller (нужно делать с повышенными правами), а после все дейсвия пользователь сможет сделать самостоятельно.

В примере будет испрользоваться:

Команда: demo

Namespace: Demo-team

Server: test-server

Context (для пользователя): demo-team-admin-context


## Создание аккаунта (для Support)

Создайте в namespace'е команды учётную запись tiller
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: demo-team
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: demo-team
```

Применяем конфиг и удаляем файл
```bash
sudo -u kube kubectl apply -f /tmp/tiller-account.yml
rm -f /tmp/tiller-account.yml
```

Всё. Дальше начинает работать пользователь.

Helm должен быть установлен по статье Установка Helm


## Пример работы пользователя

* Опционально прописываем прокси команды *
```bash
export http_proxy=http://USER:PASSWORD@proxy:3128
export https_proxy=http://USER:PASSWORD@proxy:3128
export no_proxy=`hostname -I | cut -d' ' -f1`
```

Инициализирует Helm на Вашем аккаунте. Не забывайте проставлять настоящие контексты и названия namespace'а
```bash
helm init --kube-context demo-team-admin-context --tiller-namespace demo-team --override 'spec.template.spec.containers[0].resources.limits.cpu'="500m" --override 'spec.template.spec.containers[0].resources.limits.memory'="512Mi" --override 'spec.template.spec.containers[0].resources.requests.cpu'="100m" --override 'spec.template.spec.containers[0].resources.requests.memory'="100Mi" --service-account tiller
```
По факту будет создан каталог .helm в домашнем каталоге пользоватлея и запущен один под "tiller-deploy" в рабочем Namespace'е
```bash
kubectl --namespace demo-team --context=17team-demo-team-admin-context get pods
NAME                             READY   STATUS    RESTARTS   AGE
tiller-deploy-6c979f68f6-x6l7r   1/1     Running   0          17m
```
Отлично. Теперь можно работать с helm. Попробуем выполнить поиск

```bash
helm search postgres 
NAME                                   CHART VERSION    APP VERSION    DESCRIPTION                                                 
stable/postgresql                      6.0.0            11.4.0         Chart for PostgreSQL, an object-relational database manag...
stable/prometheus-postgres-exporter    0.7.1            0.5.1          A Helm chart for prometheus postgres-exporter               
stable/stolon                          1.1.2            0.13.0         Stolon - PostgreSQL cloud native High Availability.         
stable/gcloud-sqlproxy                 0.6.1            1.11           DEPRECATED Google Cloud SQL Proxy  
```
В качестве эксперимента попробуем развернуть postgresql версии 11.4.0
```bash
helm install --kube-context demo-team-admin-context --tiller-namespace demo-team --set resources.limits.cpu=500m --set resources.limits.memory=512Mi --set resources.requests.cpu=100m --set resources.requests.memory=100Mi --set persistence.enabled=false stable/postgresql --name mypostgresql --set postgresqlUsername=test --set postgresqlPassword=test --set postgresqlDatabase=ttt
```

Параметрами resources.limits.cpu=500m --set resources.limits.memory=512Mi выбираются лимиты по CPU и памяти соотвественно, name - название heml-релиза и postgresqlUsername=test --set postgresqlPassword=test --set postgresqlDatabase=ttt задаётся логин, пароль и БД соотвественно.
```bash
helm install --kube-context demo-team-admin-context --tiller-namespace demo-team --set resources.limits.cpu=500m --set resources.limits.memory=512Mi --set resources.requests.cpu=100m --set resources.requests.memory=100Mi --set persistence.enabled=false stable/postgresql --name mypostgresql --set postgresqlUsername=test --set postgresqlPassword=test --set postgresqlDatabase=ttt
```

Смотрим, что у нас получилось
```bash
helm list  --kube-context demo-team-admin-context --tiller-namespace demo-team
NAME            REVISION    UPDATED                     STATUS      CHART               APP VERSION    NAMESPACE
mypostgresql    1           Mon Jul 22 21:16:11 2019    DEPLOYED    postgresql-6.0.0    11.4.0         demo-team
```
У нас есть один Helm-релиз и он успешно DEPLOYED. Узнать более подробную информацию о нём можно через команду status
```bash
helm status mypostgresql --kube-context 17team-demo-team-admin-context --tiller-namespace demo-team
LAST DEPLOYED: Tue Jul 23 19:04:06 2019
NAMESPACE: demo-team
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                       READY  STATUS   RESTARTS  AGE
mypostgresql-postgresql-0  1/1    Running  0         41h

==> v1/Secret
NAME          TYPE    DATA  AGE
mypostgresql  Opaque  1     41h

==> v1/Service
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
mypostgresql           ClusterIP  10.109.89.180  <none>       5432/TCP  41h
mypostgresql-headless  ClusterIP  None           <none>       5432/TCP  41h

==> v1beta2/StatefulSet
NAME                     READY  AGE
mypostgresql-postgresql  1/1    41h


NOTES:
** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS name from within your cluster:

    mypostgresql.demo-team.svc.cluster.local - Read/Write connection
To get the password for "test" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace demo-team mypostgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run mypostgresql-client --rm --tty -i --restart='Never' --namespace demo-team --image docker.io/bitnami/postgresql:11.4.0-debian-9-r12 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host mypostgresql -U test -d ttt -p 5432



To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace demo-team svc/mypostgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U test -d ttt -p 5432
```

Кроме статуса ниже ещё предложены возможности подключения через запуск psql в контейнере или через port-forward. К сожалению, использовать psql в контейнере не удобно, а port-forward не работает в закрытой сети ЦОД. Однако есть возможность вывести его через NodePort

Делается это вот так:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mypostgres
  namespace: demo-team
  labels:
    name: mypostgres-svc
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: postgresql
```
```bash
kubectl --context=demo-team-admin-context  apply -f /tmp/service-mypostgresql.yml
kubectl --namespace demo-team --context=demo-team-admin-context get service
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mypostgres              NodePort    10.115.239.175   <none>        5432:30539/TCP   36s
mypostgresql            ClusterIP   10.115.89.165    <none>        5432/TCP         41h
mypostgresql-headless   ClusterIP   None             <none>        5432/TCP         41h
tiller-deploy           ClusterIP   10.102.175.21    <none>        44134/TCP        2d16h
```

В примере рабочий порт 5432 контейнера замапился на наружний порт 30539. Попробуем подключиться к нему со своей машины
```bash
psql -h test-server -p 30539 -U test -d ttt
Password for user test: 
psql (9.6.10, server 11.4)
WARNING: psql major version 9.6, server major version 11.
         Some psql features might not work.
Type "help" for help.
ttt=> 
```
Отлично!
Удаление

Если данный релиз больше не нужен - можете удалить всё следующими командами
```bash
kubectl --namespace demo-team  --context=17team-demo-team-admin-context delete service mypostgres
helm delete --purge mypostgresql --kube-context 17team-demo-team-admin-context --tiller-namespace demo-team
```
