# Мониторинг каждого podа в микросервисе Kubernetes

### Описание
Для того, чтобы получать метрики от каждого Pod'а микросервиса (а не от одного Pod'а, который выбрал service), необходимо выполнить ряд настроек с Kubernetes-кластером и задействовать в микросервис Prometheus в определённом Namespace'е. Итогом будет возможно просматривать метрики с каждого Pod'а в отдельности, а не одного случайно выбранного.

### Настройка
Для того, чтобы Prometheus мог увидеть все pod'ы по отдельности - он сам должен стать pod'ом. 

Разворачиваем следующий Deployment.

<details>
  <summary>prom.yaml</summary>

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- nonResourceURLs:
  - /actuator/prometheus
  verbs:
  - get
 
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoringk8s
 
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoringk8s
 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: monitoringk8s
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s
      evaluation_interval: 15s
  
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
  
    rule_files:
 
    scrape_configs:
      - job_name: kubernetes-pods
        scrape_interval: 15s
        scrape_timeout: 15s
        scheme: http
        metrics_path: '/actuator/prometheus'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          separator: ;
          regex: "true"
          replacement: $1
          action: keep
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          separator: ;
          regex: (.+)
          target_label: __metrics_path__
          replacement: $1
          action: replace
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          separator: ;
          regex: ([^:]+)(?::\d+)?;(\d+)
          target_label: __address__
          replacement: $1:$2
          action: replace
        - separator: ;
          regex: __meta_kubernetes_pod_label_(.+)
          replacement: $1
          action: labelmap
        - source_labels: [__meta_kubernetes_namespace]
          separator: ;
          regex: (.*)
          target_label: kubernetes_namespace
          replacement: $1
          action: replace
        - source_labels: [__meta_kubernetes_pod_name]
          separator: ;
          regex: (.*)
          target_label: kubernetes_pod_name
          replacement: $1
          action: replace
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
  generation: 3
  labels:
    app: prometheus
  name: prometheus
  namespace: monitoringk8s
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: prometheus
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: prometheus
    spec:
      containers:
      - args:
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.retention.size=7GB
        - --storage.tsdb.wal-compression
        - --storage.tsdb.path=/prometheus
        image: prom/prometheus:v2.12.0
        imagePullPolicy: Always
        name: prometheus
        ports:
        - containerPort: 9090
          protocol: TCP
        resources:
          limits:
            cpu: 300m
            memory: 1724Mi
        volumeMounts:
        - name: prometheus
          mountPath: /etc/prometheus/prometheus.yml
          subPath: prometheus.yml
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: prometheus
      serviceAccountName: prometheus
      terminationGracePeriodSeconds: 30
      volumes:
      - name: prometheus
        configMap:
          name: prometheus
 
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  labels:
    app: prometheus
  namespace: monitoringk8s
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
  selector:
    app: prometheus
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  generation: 1
  name: promerheus
  namespace: monitoringk8s
spec:
  rules:
    - host: prometheus.10.10.10.25.nip.io
      http:
        paths:
          - backend:
              serviceName: prometheus
              servicePort: 9090
            path: /
```
</details>

### Провереяем
```bash
kubectl apply -f prom.yml
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
configmap/prometheus created
deployment.apps/prometheus created
service/prometheus created
ingress.extensions/gmlc-adapter-promerheus created

kubectl -n ngenie-gmlc get service
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
gmlc-adapter   LoadBalancer   10.105.101.217   <pending>     8080:32698/TCP,9102:31856/TCP   27h
prometheus     NodePort       10.111.19.128    <none>        9090:31781/TCP                  40s
```


http://gmlc-adapter.10.10.10.25.nip.io:31781/

Таким образом мы вывели Prometheus на Ingress. Теперь настроим общий Prometheus-сервер на сбор метрик с нашего Prometheus из контейнера
```yaml
/opt/prometheus/prometheus.yml
...
- job_name: 'gmlc'
  scrape_interval: 15s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
    - '{job=~".*..*"}'
    - '{namespace=~".*..*"}'
    - '{node=~".*..*"}'
    - '{pod=~".*..*"}'
    - '{__name__=~".*..*"}'
  static_configs:
    - targets:
      - 'gmlc-adapter.10.10.10.25.nip.io:80'
...
```
Перезапускаем
```bash
systemctl restart prometheus
```