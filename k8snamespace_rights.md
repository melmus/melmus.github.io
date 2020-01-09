# Предоставление доступа на несколько namespace

С помощью данной инструкции вы сможете дать доступ к одному или нескольким Namespace'ам одного Kubernetes-кластера.

Дополнительно мы будем устанавливать лимиты по CPU и RAM на каждый Namespace

В примерах будет использоваться пользователь kube - данный пользователь является админом в кластере и от его имени будут запускаться несколько команд


Для выполнения необходимо повышение прав
```bash
    sudo -u kube kubectl config view
    sudo -u kube kubectl create namespace*
    sudo mkdir /etc/kubernetes/pki/teams/
    sudo openssl genrsa -out /etc/kubernetes/pki/teams/*
    sudo openssl req -new -key /etc/kubernetes/pki/teams/*
    sudo openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key*
    sudo -u kube kubectl config set-credentials*
    sudo -u kube kubectl config set-context*
    sudo -u kube kubectl apply -f*
    sudo chmod 600 /etc/kubernetes/pki/teams/*
    sudo grep certificate-authority-data /etc/kubernetes/admin.conf
```
## Настройка

Все действия выполняем на Мастер-сервере Kubernetes. Для начала настройки необходимо задать несколько параметров в перенные
```bash
TEAM=demo-team # Название пользвателя
NAMESPACE=dev # Название Namespace'а, на который мы будем давать права
IP=`ifconfig ens192 | grep inet | sed 's/\ \{1,\}/ /g' | cut -d' ' -f 3` # IP-адрес Мастер-сервера (убедитесь, что сетевой интерфейс "ens192" существует)
CPU=4 # Лимит на вышеуказанный Namespace в CPU
RAM=8 # Лимит на вышеуказанный Namespace в ГБ RAM
```
Первым делом проверьте, что пользователя не существует
```bash
sudo -u kube kubectl config view
...
contexts:
- context:
    cluster: kubernetes
    namespace: dev
    user: test-team
  name: test-team-dev-admin-context
- context:
    cluster: kubernetes
    namespace: test-func
    user: test-team
  name: test-team-test-fync-admin-context
```
Если Вы увидите, что данному пользователю присвоены некоторые Context'ы переходите сразу к переходите сразу к Добавление новых Namespace к существующему пользователю. Не вздумайте продолжать следующие команды - Вы загубите существующих пользователей.

Если пользователя нет - начнём с его создания

Создаём новые namespace'ы
```bash
sudo -u kube kubectl create namespace ${NAMESPACE}
```
Создаём каталог для хранения сертификатов (если уже существует - ничего страшного
```bash
sudo mkdir /etc/kubernetes/pki/teams/
```
Создаём на Kubernetes Master сертификат и приватный ключ для пользователя, которому требуется предоставить доступ на Namespace(ы)
```bash
sudo openssl genrsa -out /etc/kubernetes/pki/teams/${TEAM}.key 2048
sudo openssl req -new -key /etc/kubernetes/pki/teams/${TEAM}.key  -out /etc/kubernetes/pki/teams/${TEAM}.csr -subj "/CN=${TEAM}/O=namespace-admins"
sudo openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -in /etc/kubernetes/pki/teams/${TEAM}.csr  -out /etc/kubernetes/pki/teams/${TEAM}.crt -days 1024
```
Создаём на Kubernetes пользователя и присваем ему недавно созданные сертификаты и ключи
```bash
sudo -u kube kubectl config set-credentials ${TEAM} --client-certificate=/etc/kubernetes/pki/teams/${TEAM}.crt --client-key=/etc/kubernetes/pki/teams/${TEAM}.key
```
Далее создаём Context - идентификатор для команд kubectr
```bash
sudo -u kube kubectl config set-context ${TEAM}-${NAMESPACE}-admin-context --cluster=kubernetes --namespace=${NAMESPACE} --user=${TEAM}
```
Создайте пары Role и RoleBinding для Namespace'а. Тут же задаём лимиты по ресурсам.
```bash
cat > /tmp/${TEAM}-${NAMESPACE}.yml << END
```
```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: ${NAMESPACE}
  name: ${TEAM}-namespace-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources:
    - componentstatuses
    - configmaps
    - daemonsets
    - deployments
    - deployments/scale
    - events
    - endpoints
    - horizontalpodautoscalers
    - ingress
    - ingresses
    - ingresses.status
    - jobs
    - limitranges
    - namespaces
    - nodes
    - pods
    - pods/exec    ##### Addinal ##### 
    - pods/attach  ##### Permissions #####
    - pods/log
    - pods/portforward
    - persistentvolumes
    - persistentvolumeclaims
    - replicasets
    - replicationcontrollers
    - secrets
    - serviceaccounts
    - services
  verbs: ["*"]
- apiGroups: [""]
  resources:
    - resourcequotas
  verbs: ["get", "watch", "list"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: ${TEAM}-namespace-manager-binding
  namespace: ${NAMESPACE}
subjects:
- kind: User
  name: ${TEAM}
  apiGroup: ""
roleRef:
  kind: Role
  name: ${TEAM}-namespace-manager
  apiGroup: ""
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${TEAM}-quota
  namespace: ${NAMESPACE}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "${CPU}"
    limits.memory: ${RAM}Gi
```
```bash
END
```
Применяем файл в Kubernetes'е
```bash
sudo -u kube kubectl apply -f /tmp/${TEAM}-${NAMESPACE}.yml
```
Создаем папки для YAML-файлов и дисковых разделов.
```bash
sudo -u kube mkdir -m 777 /kuber/yamls/dep-create/${NAMESPACE}
sudo chgrp admsk /kuber/yamls/dep-create/${NAMESPACE}
sudo -u kube mkdir -m 777 /kuber/volumes/${NAMESPACE}
sudo chgrp admsk /kuber/volumes/${NAMESPACE}
```
Формируем ~/.kube/config для пользователя, файл настроек для клиента kubectl
```bash
CAD=`sudo grep certificate-authority-data /etc/kubernetes/admin.conf`
cat > config << END
apiVersion: v1
clusters:
- cluster:
${CAD}
    server: https://${IP}:6443
  name: kubernetes
current-context:
kind: Config
preferences: {}
users:
- name: ${TEAM}
  user:
    client-certificate: ${TEAM}.crt
    client-key: ${TEAM}.key
contexts:
- context:
    cluster: kubernetes
    namespace: ${NAMESPACE}
    user: ${TEAM}
  name: ${TEAM}-${NAMESPACE}-admin-context 
END
```
Прячем сертификаты и удаляем старый yml-файл
```bash
sudo -u kube kubectl apply -f /tmp/${TEAM}-${NAMESPACE}.yml
rm -f /tmp/${TEAM}-${NAMESPACE}.yml
```
Запакуйте сертификаты и конфиг в zip-файл с паролем
```bash
sudo zip -j -e ${TEAM}.zip /etc/kubernetes/pki/teams/${TEAM}.crt /etc/kubernetes/pki/teams/${TEAM}.key config
```
Передайте zip-архив пользователю. Теперь он может распаковать его
```bash
mkdir ~/.kube
unzip *.zip
cp config *.crt *.key ~/.kube
```
и взаимодействовать со своим Namespace'ом через --context
```bash
kubectl --context=demo-team-test-dev-admin-context -n dev get pods
No resources found.
```

Не обязательно сохранять связку Context == Namespace при работе с kubectl

То есть после данного примера успешно работают все перечисленные Namespace'ы с указанием одного из Context'а (но не дальше перечисленных). 

## Пример

kubectl --context=demo-dev-admin-context -n demo-dev get services
No resources found.
kubectl --context=demo-dev-admin-context -n demo-dev-security get services
No resources found. 
kubectl --context=demo-dev-admin-context -n test get services
Error from server (Forbidden): pods is forbidden: User "demo-dev" cannot list resource "services" in API group "" in the namespace "test"

Полезные команды для пользователя для просмотра конфига и переключение контекста:
```bash
kubectl config view
kubectl config use-context
```

### Добавление новых Namespace к существующему пользователю

Если выяснилось, что пользователь под команду уже существует, но к нужно добавить ещё несколько Namespace'ов - выполните следущующие (урезанные) действия.

Снова задайте параметры, изменив название Namespace'ов и лимиты
```bash
TEAM=demo-team # Название пользвателя
NAMESPACE=dev-dmz # Название Namespace'а, на который мы будем давать права
CPU=2 # Лимит на вышеуказанный Namespace в CPU
RAM=4 # Лимит на вышеуказанный Namespace в ГБ RAM
```
Создайте Namespace
```bash
sudo -u kube kubectl create namespace ${NAMESPACE}
```
Создайте Context
```bash
sudo -u kube kubectl config set-context ${TEAM}-${NAMESPACE}-admin-context --cluster=kubernetes --namespace=${NAMESPACE} --user=${TEAM}
```
Создайте пары Role и RoleBinding для Namespace'а, а также лимиты
```bash
cat > /tmp/${TEAM}-${NAMESPACE}.yml << END
```
```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
   namespace: ${NAMESPACE}
   name: ${TEAM}-namespace-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources:
    - componentstatuses
    - configmaps
    - daemonsets
    - deployments
    - events
    - endpoints
    - horizontalpodautoscalers
    - ingress
    - ingresses
    - ingresses.status
    - jobs
    - limitranges
    - namespaces
    - nodes
    - pods
    - pods/log
    - pods/exec
    - pods/attach
    - pods/portforward
    - persistentvolumes
    - persistentvolumeclaims
    - replicasets
    - replicationcontrollers
    - secrets
    - serviceaccounts
    - services
  verbs: ["*"]
- apiGroups: [""]
  resources:
    - resourcequotas
  verbs: ["get", "watch", "list"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
   name: ${TEAM}-namespace-manager-binding
   namespace: ${NAMESPACE}
subjects:
 - kind: User
   name: ${TEAM}
   apiGroup: ""
roleRef:
   kind: Role
   name: ${TEAM}-namespace-manager
   apiGroup: ""
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${TEAM}-quota
  namespace: ${NAMESPACE}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "${CPU}"
    limits.memory: ${RAM}Gi
```
```bash
END
```
Примените и удалите вышесозданный файл
```bash
sudo -u kube kubectl apply -f /tmp/${TEAM}-${NAMESPACE}.yml
rm -f /tmp/${TEAM}-${NAMESPACE}.yml
```
Добавьте новый Namespace в config-файл для пользователя.
```bash
cat >> config << END
- context:
    cluster: kubernetes
    namespace: ${NAMESPACE}
    user: ${TEAM}
  name: ${TEAM}-${NAMESPACE}-admin-context 
END
```
Повторите вышеуказанные действия, если у Вас есть ещё недосозданные Namespace'ы. После продолжайте.

Если старый config-файл не сохранился - передайте ему данные дополнительные настройки и попросите самостоятельно прописать его в конец своего файла ~/.kube/config

Если сохранился - можете передать весь файл цельно.

Сертификаты, ключи передавать не нужно, ровно как и шифровать файлы нет смысла.

Полезные команды для пользователя для просмотра конфига и переключение контекста:
```bash
kubectl config view
kubectl config use-context
```

### Удаление пользователя

Для удаления пользователя выполните:
```bash
kubectl config unset "users.${TEAM}"
```

### Добавление прав

Если пользователи наткнулись на ошибку недостаточности прав, у Вас есть возможность дополнить уже существующий Namespace недостающими правами.

Пример ошибки:
```bash
kubectl attach dev-k8s-6c8db775f7-mfgtb -n dev
Error from server (Forbidden): pods "dev-k8s-6c8db775f7-mfgtb" is forbidden: 
User "demo-team" cannot create resource "pods/attach" in API group "" in the namespace "dev"

kubectl exec -it dev-k8s-6c8db775f7-mfgtb sh -n dev
Error from server (Forbidden): pods "dev-k8s-6c8db775f7-mfgtb" is forbidden: 
User "demo-team" cannot create resource "pods/exec" in API group "" in the namespace "dev"
```

В сообщении ошибки всегда видно, какому пользователю (demo-team), каких прав (pods/attach и pods/exec) на какой Namespace (dev) не хватает.

На основании данного примера мы добавим права.

Копируем yaml-раздел по созданию Role и вносим в него данные из ошибки:
```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
   namespace: dev ##### NAMESPACE #####
   name: demo-team-namespace-manager  ##### USER'S ROLENAME
rules:
- apiGroups: ["", "extensions", "apps"]
  resources:
    - componentstatuses
    - configmaps
    - daemonsets
    - deployments
    - events
    - endpoints
    - horizontalpodautoscalers
    - ingress
    - ingresses
    - ingresses.status
    - jobs
    - limitranges
    - namespaces
    - nodes
    - pods
    - pods/log
    - pods/exec    ##### Addinal ##### 
    - pods/attach  ##### Permissions #####
    - pods/portforward
    - persistentvolumes
    - persistentvolumeclaims
    - replicasets
    - replicationcontrollers
    - secrets
    - serviceaccounts
    - services
  verbs: ["*"]
- apiGroups: [""]
  resources:
    - resourcequotas
  verbs: ["get", "watch", "list"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
```
Применяем данные настройки в Kubernetes
```bash
sudo -u kube kubectl apply -f /tmp/demo-team-add.yml
rm -f /tmp/demo-team-add.yml
```
Готово

### Увеличение квоты

Если пользователь упёрся в лимит - у него перестанут запускаться новые контейнеры (внимание на web)
```bash
kubectl --context demo-dev-dmz-admin-context -n demo-dev get deploy
NAME   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
db     1         1         1            1           3h29m
sec    2         2         1            2           10m
web    2         0         0            0           10m

kubectl --context demo-team-ngenie-dev-dmz-admin-context -n demo-dev get pods
NAME                   READY   STATUS    RESTARTS   AGE
db-8499854f49-54fc4    1/1     Running   0          164m
sec-54f8c97778-45skj   1/1     Running   0          5m16s
sec-5cb8999df8-cdpnv   1/1     Running   0          10m
```
Ошибку можно посмотреть следующим образом
```bash
kubectl --context demo-team-dev-dmz-admin-context -n dev describe deploy web
Name:                   web
Namespace:              dev
CreationTimestamp:      Thu, 04 Apr 2019 16:36:17 +0300
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"web","namespace":"ndev"},"spec":{"replicas":2,"sele...
Selector:               name=web
Replicas:               2 desired | 0 updated | 0 total | 0 available | 3 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  name=web
  Containers:
   web:
    Image:      hub.docker.com/web:3.0.2
    Port:       8081/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     1
      memory:  1Gi
    Requests:
      cpu:        1
      memory:     1Gi
    Environment:  <none>
    Mounts:
      /opt/conf from dev-volume (rw)
  Volumes:
   dev-volume:
    Type:          HostPath (bare host directory volume)
    Path:          /kuber/volumes/dev/conf/
    HostPathType:  
Conditions:
  Type             Status  Reason
  ----             ------  ------
  Available        False   MinimumReplicasUnavailable
  ReplicaFailure   True    FailedCreate
  Progressing      False   ProgressDeadlineExceeded
OldReplicaSets:    web-74f7565dcf (0/2 replicas created)
NewReplicaSet:     web-66c5c6d979 (0/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled up replica set web-74f7565dcf to 2
  Normal  ScalingReplicaSet  5m33s  deployment-controller  Scaled up replica set web-66c5c6d979 to 1

kubectl --context demo-team-dev-dmz-admin-context -n dev describe replicaset web-66c5c6d979
Name:           web-66c5c6d979
Namespace:      dev
Selector:       name=web,pod-template-hash=66c5c6d979
Labels:         name=web
                pod-template-hash=66c5c6d979
Annotations:    deployment.kubernetes.io/desired-replicas: 2
                deployment.kubernetes.io/max-replicas: 3
                deployment.kubernetes.io/revision: 2
Controlled By:  Deployment/web
Replicas:       0 current / 1 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=web
           pod-template-hash=66c5c6d979
  Containers:
   web:
    Image:      hub.docker.com/web:3.0.2
    Port:       8081/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     1
      memory:  1Gi
    Requests:
      cpu:        1
      memory:     1Gi
    Environment:  <none>
    Mounts:
      /opt/conf from dev-volume (rw)
  Volumes:
   dev-volume:
    Type:          HostPath (bare host directory volume)
    Path:          /kuber/volumes/dev/conf/
    HostPathType:  
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason        Age                  From                   Message
  ----     ------        ----                 ----                   -------
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-bclmd" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-4xfvs" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-6xgwb" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-btdj6" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-fr5cm" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-z68jp" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-2btrr" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m46s                replicaset-controller  Error creating: pods "web-66c5c6d979-wkd66" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  5m45s                replicaset-controller  Error creating: pods "web-66c5c6d979-x4tc2" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
  Warning  FailedCreate  19s (x8 over 5m44s)  replicaset-controller  (combined from similar events): Error creating: pods "web-66c5c6d979-rpjq6" is forbidden: exceeded quota: demo-team-quota, requested: requests.cpu=1, used: requests.cpu=4, limited: requests.cpu=4
```
Как видно по сообщению, на demo-team-quota используется requests.cpu=4, хочет использоваться ещё requests.cpu=1, но он уже не лезет в лимит request.cpu=4. Приблизительно такое сообщение может появиться по RAM, а также по hard-ограничению.

Можно попросить пользователя уменьшить потребляемые лимиты контейнеров, а можно - увеличить квоту. Последним мы и займёмся - увеличим request.cpu и hard.cpu до 8
```bash
vim /tmp/demo-team-quota.yml
```
```yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-team-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 8Gi
```
Применяем и удаляём файл
```bash
sudo -u kube kubectl apply -f /tmp/demo-team-quota.yml
rm -f /tmp/demo-team-quota.yml
```

Просим пользователей пересоздать проблемный deployment и готово
```bash
kubectl --context demo-team-dev-dmz-admin-context -n dev get deploy
NAME   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
db     1         1         1            1           3h40m
sec    2         2         1            2           21m
web    2         0         0            0           21m

kubectl --context demo-team-dev-dmz-admin-context -n dev delete deploy web
deployment.extensions "web" deleted

kubectl --context demo-team-ngenie-dev-dmz-admin-context apply -f secweb.yml
deployment.apps/sec unchanged
service/sec unchanged
deployment.apps/web created
service/web unchanged

kubectl --context demo-team-dev-dmz-admin-context -n dev get deploy
NAME   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
db     1         1         1            1           3h40m
sec    2         2         1            2           22m
web    2         2         2            2           13s

kubectl --context demo-team-ngenie-dev-dmz-admin-context -n dev get pods
NAME                   READY   STATUS    RESTARTS   AGE
db-8499854f49-54fc4    1/1     Running   0          176m
sec-54f8c97778-45skj   1/1     Running   0          16m
sec-5cb8999df8-cdpnv   1/1     Running   0          22m
web-66c5c6d979-qbdtf   1/1     Running   0          10s
web-66c5c6d979-skffb   1/1     Running   0          10s
```
