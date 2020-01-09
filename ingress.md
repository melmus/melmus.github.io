# Ingress Controller Kubernetes


Описание

Данная статья описывает настройку Ingress Controller'а в связки с динамическим FQND для доступа к приложениям в Kubernetes по адресу вида "<project>.<IP>.nip.io"
Настройка 1.17.1

Исходники: https://github.com/kubernetes/ingress-nginx/tree/nginx-0.17.1/deploy


OS: Oracle Linux
Kubernetes: v1.9.1+2.1.8.el7
Docker: 17.03.1-ce
Основной YAML

```yaml
---

apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        image: gcr.io/google_containers/defaultbackend:1.4
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: ingress-nginx
  labels:
    app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app: ingress-nginx
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
        - events
    verbs:
        - create
        - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true'
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.17.1
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
                drop:
                - ALL
                add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
```
Применяем
```bash
kubectl aplly -f mandatory.yaml
```

### Service

Добавляем сервис

Не забудьте поменять "externalIPs" на Ваш(и)!

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  selector:
    app: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  externalIPs:
  - 10.10.10.25
  - 10.10.10.28
```

Для версии 0.23.0 изменили label пода nginx-ingress-controller, поэтому используем следующее

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  selector:
    app.kubernetes.io/name: ingress-nginx 
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  externalIPs:
  - 10.10.10.25
  - 10.10.10.28
```
```bash
kubectl create -f service.yml
service "ingress-nginx" created

kubectl -n ingress-nginx get service
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
default-http-backend   ClusterIP   10.108.40.183    <none>          80/TCP           18m
ingress-nginx          ClusterIP   10.111.208.118   10.10.10.25   80/TCP,443/TCP   5s

```

Общая картина
```bash
kubectl -n ingress-nginx get all
NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/default-http-backend       1         1         1            1           19m
deploy/nginx-ingress-controller   1         1         1            1           9m
NAME                                    DESIRED   CURRENT   READY     AGE
rs/default-http-backend-7bd884976       1         1         1         19m
rs/nginx-ingress-controller-766f45999   1         1         1         9m
NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/default-http-backend       1         1         1            1           19m
deploy/nginx-ingress-controller   1         1         1            1           9m
NAME                                    DESIRED   CURRENT   READY     AGE
rs/default-http-backend-7bd884976       1         1         1         19m
rs/nginx-ingress-controller-766f45999   1         1         1         9m
NAME                                          READY     STATUS    RESTARTS   AGE
po/default-http-backend-7bd884976-4v6xd       1/1       Running   0          19m
po/nginx-ingress-controller-766f45999-rm6ml   1/1       Running   0          9m
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
svc/default-http-backend   ClusterIP   10.108.40.183    <none>          80/TCP           19m
svc/ingress-nginx          ClusterIP   10.111.208.118   10.10.10.25     80/TCP,443/TCP   1m
```

## Настройка 1.12

Исходники: https://github.com/kubernetes/ingress-nginx/tree/master/deploy

OS: Oracle Linux
Kubernetes: v1.12.2
Docker: 18.06.1-ce
Всё в одном

Используйте файл mandatory.yaml
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
```
Применяем
```bash
kubectl create -f mandatory.yaml
```
## Service

### Добавляем сервис

Не забудьте поменять "externalIPs" на Ваш(и) точки входа!

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  selector:
    app: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  externalIPs:
  - 10.10.10.25
  - 10.10.10.28
```
Применяем:
```bash
kubectl create -f service.yml
```

## Применение

Контроллер готов к работе. Для его использования мы возмём один из проектов в отдельно Namespace'е и дадим к нему доступ через Ingress
```bash
kubectl -n kube-system get services
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   68d
kubernetes-dashboard   ClusterIP   10.96.28.159   <none>        443/TCP         60d

kubectl -n kube-system get deploy kubernetes-dashboard
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1         1         1            1           60d
```

Создайте сертификат (есть проходили настройку dashboard по инструкции - это можно пропустить)

Для корректной работы, необходимо сгенерировать сертификаты:
```bash
mkdir /kuber/crt
cd /kuber/crt
openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
openssl req -new -key dashboard.key -out dashboard.csr
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt
```

Затем происходит генерация secrets:
```bash
kubectl create secret generic kubernetes-dashboard-certs --from-file=/kuber/crt --from-file=tls.crt=/kuber/crt/dashboard.crt --from-file=tls.key=/kuber/crt/dashboard.key -n kube-system
```


```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.org/ssl-backends: "kubernetes-dashboard"
    kubernetes.io/ingress.allow-http: "false"
    nginx.ingress.kubernetes.io/secure-backends: "false"
  name: kubernetes-dashboard 
  namespace: kube-system # Ingress создаётся в том Namespace'е, где располагаются конечные узлы.
spec:
  tls:
  - hosts:
    - dashboard.10.10.10.25.nip.io # Виртуальный адрес Вашего сайта (можно заменить на другой, но он обязательно должен резолвиться на наш сервер)
    secretName: kubernetes-dashboard-certs # Сгененированные сертификаты в secret. Они должны быть тут прописаны и имя узла в сертификате должно совпадать с имене в Ingress (иначе https не поднимится)
  rules:
  - host: dashboard.10.10.10.25.nip.io # Аналогично
    http:
      paths:
      - path: / # Веб-каталог
        backend:
          serviceName: kubernetes-dashboard # Имя микросервиса
          servicePort: 443 # Рабочий порт микросервиса
```
```bash
kubectl apply -f ingress.yml 
ingress "kubernetes-dashboard" created
```

## Дополнительные возможности
Увеличение размера загружаемого файла

При работе через Ingress может получится вот такая ошибка
```
HTTP/1.1 413 Request Entity Too Large
```
Связано это с маленьким дефолтным параметром по загрузке файлов в самом Ingress.

Увеличить данный лимит можно прямо в Ingress приложения, в котором обнаружена проблема. Добавляется опция "ingress.kubernetes.io/proxy-body-size" и "nginx.ingress.kubernetes.io/proxy-body-size" в metadata.annotaions
```bash
kubectl -n test-jira edit ingress test-jira-ingress-dns
```
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.allow-http: "true"
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
  creationTimestamp: 2019-06-28T14:25:03Z
```
Таким образом мы изменили лимит до 100МБ

### Basic аутентификация

В примере мы создаём пароль для пользователя goaway на проекте test-jira

Используйте утилиту htpasswd (пакет httpd-tools) для создания пароля
```bash
htpasswd -c ~/auth goaway
*PASSWORD*
```
Добавляем новосозданный файл в качестве секретов
```bash
cd ~
kubectl create secret generic basic-auth --from-file=auth -n test-jira
```
... и прописываем в Ingress настройки Basic-аутентификации в аннотации
```bash
kubectl -n mts-afisha-dmz edit ingress ingress.test-jira.nip.io
...

    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.org/client-max-body-size: 25m
    nginx.org/ssl-backends: stage.afisha.mts.ru
    ###Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - Go Away"
...
```
## Метрики Ingress Controller

Метрики Ingress Controller'а доступен по следующему пути
```bash
kubectl -n ingress-nginx get pods -o wide
NAME                                        READY   STATUS    RESTARTS   AGE    IP           NODE                           NOMINATED NODE
nginx-ingress-controller-56c5c48c4d-2w6nh   1/1     Running   1          110d   10.244.1.3   kbr-wrk04                      <none>
$ curl 10.244.1.3:10254/metrics
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 5.9467e-05
go_gc_duration_seconds{quantile="0.25"} 8.5396e-05
go_gc_duration_seconds{quantile="0.5"} 9.7548e-05
go_gc_duration_seconds{quantile="0.75"} 0.000119025
go_gc_duration_seconds{quantile="1"} 0.001327278
...
```
Здесь задача небольшая - вытащить их во вне, впоимать Prometheus'ом и вывести на доску (делается это один раз на весь каластер)

Вытаскиваем метрики во вне (на порт 30100)
```bash
kubectl -n ingress-nginx edit deploy nginx-ingress-controller
...
        name: nginx-ingress-controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 10254
          name: http-metrics
          protocol: TCP
...
```
```bash
vim ingress-metrics-service.yml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-metrics-service 
  labels:
    app.kubernetes.io/name: ingress-nginx
  namespace: ingress-nginx 
spec:
  type: NodePort
  ports:
  - port: 10254
    targetPort: 10254
    nodePort: 30111
  selector:
    app.kubernetes.io/name: ingress-nginx
```
```bash
kubectl apply -f ingress-metrics-service.yml
service/ingress-metrics-service created
```

Проверяем доступ с внешнего ПК
```bash
http_proxy= curl http://kbr-mst01:30111/metrics | head
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.6545e-05
go_gc_duration_seconds{quantile="0.25"} 6.9208e-05
go_gc_duration_seconds{quantile="0.5"} 7.9327e-05
go_gc_duration_seconds{quantile="0.75"} 9.9948e-05
go_gc_duration_seconds{quantile="1"} 0.003949687
go_gc_duration_seconds_sum 0.012160746
go_gc_duration_seconds_count 51
# HELP go_goroutines Number of goroutines that currently exist.
100 13496    0 13496    0     0  13496      0 --:--:-- --:--:-- --:--:-- 88209
curl: (23) Failed writing body (2888 != 8208)
```

Теперь нужно настроить Prometheus на сбор этих метрик
```bash
vim /opt/prometheus/prometheus.yml
...
 - job_name: '10.10.10.25-ingress'
   metrics_path: /metrics
   static_configs:
     - targets: ['10.10.10.25:30111']
       labels:
         port: 30111

systemctl restart prometheus
```
Далее, подключаемся к Grafana данного сервера мониторинга и устанавливаем новую доску (https://grafana.com/grafana/dashboards/7559 и/или https://github.com/kubernetes/ingress-nginx/tree/master/deploy/grafana/dashboards)

Добавление производительности Ingress Controller

Обычно nginx имеет некоторое органичение в количестве обработки запросов.

Если данное ограничение требуется повысить - самое простое это добавить ещё реплик pod'а контроллера.

По умолчанию, в Ingress Conftroller появляется всего один nginx-под
```bash
kubectl -n ingress-nginx get pods -o wide
NAME                                        READY   STATUS              RESTARTS   AGE   IP            NODE                   NOMINATED NODE
nginx-ingress-controller-64f56b6f7b-5x6f2   1/1     Running             0          91m   10.244.3.11   ngenie-geo-kbr-wrk03   <none>
```

Данное значение можно увеличить прям на горячую через правку deployment'а. Например, увеличим это количество до трёх (по количеству нод)
```bash
kubectl -n ingress-nginx edit deploy nginx-ingress-controller
...
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
...
```

Проверяем
```bash
kubectl -n ingress-nginx get pods -o wide
NAME                                        READY   STATUS              RESTARTS   AGE   IP            NODE                   NOMINATED NODE
nginx-ingress-controller-64f56b6f7b-5x6f2   1/1     Running             0          91m   10.244.3.11   kbr-wrk03 			  <none>
nginx-ingress-controller-64f56b6f7b-rqh2h   0/1     Running             0          8s    10.244.1.10   kbr-wrk02 			  <none>
nginx-ingress-controller-64f56b6f7b-zzj69   0/1     ContainerCreating   0          8s    <none>        kbr-wrk01 			  <none>
```

Прекрасно, теперь у нас три пода и запросы будут равномерно разноситься между ними.
