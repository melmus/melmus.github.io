# Мониторинг кластера Kubernetes

Для того, чтобы следить через Prometheus за тем, что происходит в кластере, нам поможет данная статья. В ней мы создадим отдельный Namespace "monitoring" и настроим сборку через него полезных метрик кластера и оформим это всё в Prometheus/Grafana.

Для работы нам потребуется админский доступа на стенд и Helm

### Настройка
Первым делом для Helm прописываем настройки прокси. В переменной "no_proxy" обязательно должен быть IP мастер-ноды
```bash
export http_proxy=http://LOGIN:PASSWORD@proxy.example.com:3128
export https_proxy=http://LOGIN:PASSWORD@proxy.example.com:3128
export no_proxy=localhost,127.0.0.1,`ifconfig ens192 | grep inet | sed 's/\ \{1,\}/ /g' | cut -d' ' -f 3`
```
### Запускаем установку с помощью Helm
```bash
helm install stable/prometheus-operator --name prometheus-operator --namespace monitoring --version 6.7.4
```
У нас установился полный пакет мониторинга. Если этого достаточно для Вас - можете вывести grafana'у в Ingress и пользоваться. Но если у Вас уже есть свой сервер Prometheus - можете просто вывести метрики наружу из кластера и настроить сбор метрик сервером Prometheus.

Начнём с начала - удаляем лишнее
```bash
kubectl -n monitoring delete deploy prometheus-operator-operator prometheus-operator-grafana
kubectl -n monitoring delete service alertmanager-operated prometheus-operated prometheus-operator-alertmanager prometheus-operator-grafana prometheus-operator-operator prometheus-operator-prometheus prometheus-operator-prometheus-node-exporter
kubectl -n monitoring delete daemonset prometheus-operator-prometheus-node-exporter
```

Далее, выводим deployment "prometheus-operator-kube-state-metrics" наружу
```bash
vim /tmp/prometheus-operator-kube-state-metrics-nodeport.yml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: prometheus-operator
    app.kubernetes.io/managed-by: Tiller
    app.kubernetes.io/name: kube-state-metrics
    helm.sh/chart: kube-state-metrics-2.0.0
  name: prometheus-operator-kube-state-metrics-nodeport
  namespace: monitoring
spec:
  ports:
  - nodePort: 30112
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/instance: prometheus-operator
    app.kubernetes.io/name: kube-state-metrics
  sessionAffinity: None
  type: NodePort
```
```bash
kubectl apply -f /tmp/prometheus-operator-kube-state-metrics-nodeport.yml
service/prometheus-operator-kube-state-metrics-nodeport create/tmp/sort
```
Данные сервис генерует следующие метрики на уровне кластера

 Список
kube_pod_container_info
kube_pod_container_resource_limits
kube_pod_container_resource_limits_cpu_cores
kube_pod_container_resource_limits_memory_bytes
kube_pod_container_resource_requests
kube_pod_container_resource_requests_cpu_cores
kube_pod_container_resource_requests_memory_bytes
kube_pod_container_status_last_terminated_reason
kube_pod_container_status_ready
kube_pod_container_status_restarts_total
kube_pod_container_status_running
kube_pod_container_status_terminated
kube_pod_container_status_terminated_reason
kube_pod_container_status_waiting
kube_pod_container_status_waiting_reason
kube_pod_created
kube_pod_info
kube_pod_labels
kube_pod_owner
kube_pod_start_time
kube_pod_status_phase
kube_pod_status_ready
kube_pod_status_scheduled
kube_pod_status_scheduled_time
К сожалению, тут находятся не все метрики.

нет метрик Node_exporter т.к. они уже используются в самой ОС. Но если что, можно их вывести по похожему выше пути
нет метрик текущего потребления контейнерами ресурсов (только Kubernetes'ие лимиты Requsts, Limits и подобные). Их мы попробуем вывести через cAdvisor
cAdvisor
Данное приложение от Google позволяет получить текущие метрики работы тех или иных подов в плане потребления ресурсов.

На самом деле данное приложение уже есть в коде helm-релиза "stable/prometheus-operator", но так просто его не вынести наружу.

Создаём файл
```bash
/tmp/cAdvisor.yml Свернуть исходный код
# Source: https://github.com/google/cadvisor/tree/master/deploy/kubernetes/base
```
```yaml 
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-cadvisor
  namespace: monitoring
 
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: prometheus-cadvisor
rules:
  - apiGroups: ['policy']
    resources: ['podsecuritypolicies']
    verbs:     ['use']
    resourceNames:
      - prometheus-cadvisor
 
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: prometheus-cadvisor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-cadvisor
subjects:
- kind: ServiceAccount
  name: prometheus-cadvisor
  namespace: monitoring
 
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: prometheus-cadvisor
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
  allowedHostPaths:
  - pathPrefix: "/"
  - pathPrefix: "/var/run"
  - pathPrefix: "/sys"
  - pathPrefix: "/var/lib/docker"
  - pathPrefix: "/dev/disk"
 
---
apiVersion: v1
kind: ServiceAccount
metadata:container_fs_reads_bytes_total
  name: prometheus-cadvisor
  namespace: monitoring
 
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-cadvisor
  namespace: monitoring
  annotations:
    seccomp.security  scrape_interval: 60s.alpha.kubernetes.io/pod: 'docker/default'
  labels:
    app: prometheus
    prometheus: prometheus-operator-prometheus-cadvisor     
spec:
  selector:
    matchLabels:
      app: prometheus
      prometheus: prometheus-operator-prometheus-cadvisor  
  template:
    metadata:
      labels:
        app: prometheus
        prometheus: prometheus-operator-prometheus-cadvisor 
    spec:
      serviceAccountName: prometheus-cadvisor
      containers:
      - name: prometheus-cadvisor
        image: k8s.gcr.io/cadvisor:v0.30.2
        resources:
          requests:
            memory: 200Mi
            cpu: 150m
          limits:
            memory: 2000Mi
            cpu: 300m
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
          readOnly: true
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: docker
          mountPath: /var/lib/docker
          readOnly: true
        - name: disk
          mountPath: /dev/disk
          readOnly: true
        ports:
          - name: http
            containerPort: 8080
            protocol: TCP
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30
      volumes:
      - name: rootfs
        hostPath:
          path: /
      - name: var-run
        hostPath:
          path: /var/run
      - name: sys  scrape_interval: 60s
 
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /var/lib/docker
      - name: disk
        hostPath:
          path: /dev/disk
 
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus
  name: prom-operator-prometheus-cadvisor
  namespace: monitoring
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 30113
  selector:
    app: prometheus
    prometheus: prometheus-operator-prometheus-cadvisor
  sessionAffinity: None
  type: NodePort
```

Применяем его

```bash
kubectl apply -f /tmp/cAdvisor.yml

serviceaccount/prometheus-cadvisor created
clusterrole.rbac.authorization.k8s.io/prometheus-cadvisor created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-cadvisor created
podsecuritypolicy.policy/prometheus-cadvisor created
serviceaccount/prometheus-cadvisor created
daemonset.ap  scrape_interval: 60s
ps/prometheus-cadvisor created
service/prom-operator-prometheus-cadvisor created
```


Данные сервис генерирует следующие метрики на уровне всего кластера:
<details>
  <summary>metrics</summary>

container_cpu_cfs_periods_total
container_cpu_cfs_throttled_periods_total
container_cpu_cfs_throttled_seconds_total
container_cpu_load_average_10s
container_cpu_schedstat_run_periods_total
container_cpu_schedstat_runqueue_seconds_total
container_cpu_schedstat_run_seconds_total
container_cpu_system_seconds_total
container_cpu_usage_seconds_total
container_cpu_user_seconds_total
container_fs_inodes_free
container_fs_inodes_total
container_fs_io_current
container_fs_io_time_seconds_total
container_fs_io_time_weighted_seconds_total
container_fs_limit_bytes
container_fs_reads_bytes_total
container_fs_read_seconds_total
container_fs_reads_merged_total
container_fs_reads_total
container_fs_sector_reads_total
container_fs_sector_writes_total
container_fs_usage_bytes
container_fs_writes_bytes_total
container_fs_write_seconds_total
container_fs_writes_merged_total
container_fs_writes_total
container_last_seen
container_memory_cache
container_memory_failcnt
container_memory_failures_total
container_memory_max_usage_bytes
container_memory_rss
container_memory_swap
container_memory_usage_bytes
container_memory_working_set_bytes
container_network_receive_bytes_total
container_network_receive_errors_total
container_network_receive_packets_dropped_total
container_network_receive_packets_total
container_network_tcp_usage_total
container_network_transmit_bytes_total
container_network_transmit_errors_total
container_network_transmit_packets_dropped_total
container_network_transmit_packets_total
container_network_udp_usage_total
container_spec_cpu_period
container_spec_cpu_quota
container_spec_cpu_shares
container_spec_memory_limit_bytes
container_spec_memory_reservation_limit_bytes
container_spec_memory_swap_limit_bytes
container_start_time_seconds
container_tasks_state
</details>

### Prometheus
Заносим в Prometheus. Поскольку даже на маленьком Kubernetes-кластере метрики от cAdvisor получается ~1МБ весом - мы сократим время считывания метрик до ежеминутного значения
```bash
vim /opt/prometheus/prometheus.yml
...
```
```yaml
- job_name: 'demo-cluster'
  metrics_path: /metrics
  scrape_interval: 60s
  static_configs:
    - targets: ['demo-cluster.example.com:30112']
      labels:
        port: 30112
    - targets: ['demo-cluster.example.com:30113']
      labels:
        pods: 30113
...
```


```bash
systemctl restart prometheus
```

### Grafana
Доски для Grafana ниже

<details>
  <summary>Kubernetes / Pods Свернуть исходный код</summary>


```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 75,
  "iteration": 1570450800388,
  "links": [],
  "panels": [
    {
      "content": "\n# Kubernetes Pods\n\n##### Select\n\n- Job as Kubernetes Cluster\n- Namespace as Namespace in Kubernetes Cluster\n- Pod as Pod in Kubernetes Namespace\n- Container as Container in Kubernetes Pods (optional)\n \n##### Node\n\nIt's metrics is too heavy, then the load's interval is 60 sec",
      "gridPos": {
        "h": 7,
        "w": 5,
        "x": 0,
        "y": 0
      },
      "id": 7,
      "links": [],
      "mode": "markdown",
      "options": {},
      "timeFrom": null,
      "timeShift": null,
      "title": "Usage Node",
      "type": "text"
    },
    {
      "gridPos": {
        "h": 7,
        "w": 10,
        "x": 5,
        "y": 0
      },
      "id": 9,
      "links": [],
      "options": {
        "displayMode": "lcd",
        "fieldOptions": {
          "calcs": [
            "mean"
          ],
          "defaults": {
            "max": 100,
            "min": 0,
            "title": ""
          },
          "mappings": [],
          "override": {},
          "thresholds": [
            {
              "color": "green",
              "index": 0,
              "value": null
            },
            {
              "color": "red",
              "index": 1,
              "value": 90
            }
          ],
          "values": false
        },
        "orientation": "horizontal"
      },
      "targets": [
        {
          "expr": "sum(kube_deployment_status_replicas{job=\"$job\", namespace=\"$namespace\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Deployments in Namespace",
          "refId": "A"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", phase=\"Running\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Pods is Running in Namespace",
          "refId": "B"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", phase=\"Failed\"})",
          "format": "time_series",
          "instant": false,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "Pods is Failed in Namespace",
          "refId": "C"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", phase=\"Succeeded\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Pods is Succeeded in Namespace",
          "refId": "D"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Short status of selected Namespace",
      "type": "bargauge"
    },
    {
      "gridPos": {
        "h": 7,
        "w": 9,
        "x": 15,
        "y": 0
      },
      "id": 10,
      "links": [],
      "options": {
        "displayMode": "lcd",
        "fieldOptions": {
          "calcs": [
            "mean"
          ],
          "defaults": {
            "max": 100,
            "min": 0
          },
          "mappings": [],
          "override": {},
          "thresholds": [
            {
              "color": "green",
              "index": 0,
              "value": null
            },
            {
              "color": "red",
              "index": 1,
              "value": 80
            }
          ],
          "values": false
        },
        "orientation": "horizontal"
      },
      "targets": [
        {
          "expr": "sum(kube_deployment_status_replicas{job=\"$job\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Deployments in Namespace",
          "refId": "A"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", phase=\"Running\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Pods is Running in Namespace",
          "refId": "B"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", phase=\"Failed\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Pods is Failed in Namespace",
          "refId": "C"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", phase=\"Succeeded\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Pods is Succeeded in Namespace",
          "refId": "D"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Short status of selected Kubernetes Cluster",
      "type": "bargauge"
    },
    {
      "cacheTimeout": null,
      "datasource": "$datasource",
      "gridPos": {
        "h": 4,
        "w": 24,
        "x": 0,
        "y": 7
      },
      "id": 12,
      "interval": "",
      "links": [],
      "options": {
        "fieldOptions": {
          "calcs": [
            "first"
          ],
          "defaults": {
            "max": 1,
            "min": 0
          },
          "mappings": [],
          "override": {},
          "thresholds": [
            {
              "color": "dark-blue",
              "index": 0,
              "value": null
            },
            {
              "color": "dark-green",
              "index": 1,
              "value": 1
            }
          ],
          "values": false
        },
        "orientation": "auto",
        "showThresholdLabels": false,
        "showThresholdMarkers": false
      },
      "pluginVersion": "6.2.2",
      "targets": [
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", pod=~\"$pod\",phase=\"Running\"})",
          "format": "time_series",
          "instant": true,
          "intervalFactor": 1,
          "legendFormat": "Running",
          "refId": "B"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\",namespace=\"$namespace\", pod=~\"$pod\",phase=\"Failed\"})",
          "format": "time_series",
          "instant": true,
          "intervalFactor": 1,
          "legendFormat": "Failed",
          "refId": "A"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", pod=~\"$pod\", phase=\"Pending\"})",
          "format": "time_series",
          "instant": true,
          "intervalFactor": 1,
          "legendFormat": "Pending",
          "refId": "C"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", pod=~\"$pod\", phase=\"Succeeded\"})",
          "format": "time_series",
          "instant": true,
          "intervalFactor": 1,
          "legendFormat": "Succeeded",
          "refId": "D"
        },
        {
          "expr": "sum(kube_pod_status_phase{job=\"$job\", namespace=\"$namespace\", pod=~\"$pod\", phase=\"Unknown\"})",
          "format": "time_series",
          "instant": true,
          "intervalFactor": 1,
          "legendFormat": "Unknown",
          "refId": "E"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Status of Pod (Right Now) ",
      "type": "gauge"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 11
      },
      "id": 2,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {},
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "repeat": null,
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum by(container_label_io_kubernetes_pod_name) (container_memory_usage_bytes{job=\"$job\", container_label_io_kubernetes_pod_namespace=\"$namespace\", container_label_io_kubernetes_pod_name=~\"$pod\", container_label_io_kubernetes_container_name=~\"$container\", container_label_io_kubernetes_container_name!=\"POD\"}) ",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Current: {{ container_label_io_kubernetes_pod_name }}",
          "refId": "A"
        },
        {
          "expr": "sum by(container) (kube_pod_container_resource_requests{job=\"$job\", namespace=\"$namespace\", resource=\"memory\", pod=~\"$pod\", container=~\"$container\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Requested: {{ container }}",
          "refId": "B"
        },
        {
          "expr": "sum by(container) (kube_pod_container_resource_limits{job=\"$job\", cluster=\"$cluster\", namespace=\"$namespace\", resource=\"memory\", pod=~\"$pod\", container=~\"$container\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Limit: {{ container }}",
          "refId": "C"
        },
        {
          "expr": "sum by(container_label_io_kubernetes_pod_name) (container_memory_cache{job=\"$job\", container_label_io_kubernetes_pod_namespace=\"$namespace\", container_label_io_kubernetes_pod_name=~\"$pod\", container_label_io_kubernetes_container_name=~\"$container\", container_label_io_kubernetes_container_name!=\"POD\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Cache: {{ container_label_io_kubernetes_pod_name}}",
          "refId": "D"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Memory Usage",
      "tooltip": {
        "shared": false,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 18
      },
      "id": 3,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {},
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "repeat": null,
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum by (container_label_io_kubernetes_pod_name) (rate(container_cpu_usage_seconds_total{job=\"$job\", cluster=\"$cluster\", container_label_io_kubernetes_pod_namespace=\"$namespace\", image!=\"\", container_label_io_kubernetes_pod_name=~\"$pod\", container_label_io_kubernetes_container_name=~\"$container\", container_label_io_kubernetes_container_name!=\"POD\"}[10m]))",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Current: {{container_label_io_kubernetes_pod_name }}",
          "refId": "A"
        },
        {
          "expr": "sum by(container) (kube_pod_container_resource_requests{job=\"$job\", cluster=\"$cluster\", namespace=\"$namespace\", resource=\"cpu\", pod=~\"$pod\", container=~\"$container\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Requested: {{ container }}",
          "refId": "B"
        },
        {
          "expr": "sum by(container) (kube_pod_container_resource_limits{job=\"$job\", cluster=\"$cluster\", namespace=\"$namespace\", resource=\"cpu\", pod=~\"$pod\", container=~\"$container\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Limit: {{ container }}",
          "refId": "C"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "CPU Usage",
      "tooltip": {
        "shared": false,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 25
      },
      "id": 4,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {},
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "repeat": null,
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum by (container_label_io_kubernetes_pod_name) (container_network_receive_bytes_total{job=\"$job\", container_label_io_kubernetes_pod_namespace=\"$namespace\", container_label_io_kubernetes_pod_name=~\"$pod\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "RX: {{ container_label_io_kubernetes_pod_name }}",
          "refId": "A"
        },
        {
          "expr": "sum by (container_label_io_kubernetes_pod_name) (container_network_transmit_bytes_total{job=\"$job\", container_label_io_kubernetes_pod_namespace=\"$namespace\", container_label_io_kubernetes_pod_name=~\"$pod\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "TX: {{ container_label_io_kubernetes_pod_name }}",
          "refId": "B"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Network I/O",
      "tooltip": {
        "shared": false,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 32
      },
      "id": 5,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {},
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "repeat": null,
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "max by (pod) (kube_pod_container_status_restarts_total{job=\"$job\", cluster=\"$cluster\", namespace=\"$namespace\", pod=~\"$pod\", container=~\"$container\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "Restarts: {{ pod }}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Total Restarts Per Container",
      "tooltip": {
        "shared": false,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": 0,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "refresh": false,
  "schemaVersion": 18,
  "style": "dark",
  "tags": [
    "kubernetes-mixin"
  ],
  "templating": {
    "list": [
      {
        "current": {
          "text": "Promet",
          "value": "Promet"
        },
        "hide": 0,
        "includeAll": false,
        "label": null,
        "multi": false,
        "name": "datasource",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "allValue": null,
        "current": {
          "text": "0700ngenie-geo-kbr-mst01.ngenie.mtsit.com-cluster",
          "value": "0700ngenie-geo-kbr-mst01.ngenie.mtsit.com-cluster"
        },
        "datasource": "$datasource",
        "definition": "label_values(kube_pod_info, job)",
        "hide": 0,
        "includeAll": false,
        "label": "Job",
        "multi": false,
        "name": "job",
        "options": [],
        "query": "label_values(kube_pod_info, job)",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {
          "isNone": true,
          "text": "None",
          "value": ""
        },
        "datasource": "$datasource",
        "definition": "label_values(kube_pod_info, cluster)",
        "hide": 2,
        "includeAll": false,
        "label": "cluster",
        "multi": false,
        "name": "cluster",
        "options": [],
        "query": "label_values(kube_pod_info, cluster)",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {
          "text": "ngenie-gmlc",
          "value": "ngenie-gmlc"
        },
        "datasource": "$datasource",
        "definition": "label_values(kube_pod_info{job=\"$job\"}, namespace)",
        "hide": 0,
        "includeAll": false,
        "label": "Namespace",
        "multi": false,
        "name": "namespace",
        "options": [],
        "query": "label_values(kube_pod_info{job=\"$job\"}, namespace)",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {
          "tags": [],
          "text": "All",
          "value": [
            "$__all"
          ]
        },
        "datasource": "$datasource",
        "definition": "label_values(kube_pod_info{job=\"$job\", namespace=\"$namespace\"}, pod)",
        "hide": 0,
        "includeAll": true,
        "label": "Pod",
        "multi": true,
        "name": "pod",
        "options": [],
        "query": "label_values(kube_pod_info{job=\"$job\", namespace=\"$namespace\"}, pod)",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {
          "text": "All",
          "value": "$__all"
        },
        "datasource": "$datasource",
        "definition": "label_values(kube_pod_container_info{job=\"$job\", namespace=\"$namespace\", pod=~\"$pod\"}, container)",
        "hide": 0,
        "includeAll": true,
        "label": "Container",
        "multi": false,
        "name": "container",
        "options": [],
        "query": "label_values(kube_pod_container_info{job=\"$job\", namespace=\"$namespace\", pod=~\"$pod\"}, container)",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-15m",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "time_options": [
      "5m",
      "15m",
      "1h",
      "6h",
      "12h",
      "24h",
      "2d",
      "7d",
      "30d"
    ]
  },
  "timezone": "",
  "title": "Kubernetes / Pods",
  "uid": "S8yxuKhZzd",
  "version": 18
}
```
</details>
