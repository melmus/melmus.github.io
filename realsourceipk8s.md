### Real Source IP в Ингресс контроллере

При установке nginx ingress controller создаётся Service типа ClusterIP с параметром externalIPS, в котором указан физический адрес ноды. При таких настройках происходит натирование и в pod с ингресс контроллером пакеты попадают c source ip внутренней куберовской сети..
Обойти это и получить настоящий ip адрес источника запроса можно используя сервис типа LoadBalancer, но для этого нужен внешний балансировщик с поддержкой протокола прокси, используя который nginx ingress сможет получить source ip клиента(в данной заметке этот способ не рассматривается)

Второй способ - это использовать сервис типа NodePort и ExternalTrafficPolicy: Local
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: ingress-nginx
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local 
  ports:
  - name: web
    nodePort: 32000
    port: 80
    protocol: TCP
    targetPort: 80
  - name: webssl
    nodePort: 32001
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  type: NodePort
```

Но в данном случае есть определённое ограничение внешний трафик должен приходить не на любую ноду кластера, а именно на те где развёрнуты podы c ingress controller.

В данном примере настроим, чтобы pod с ingress развернулся на мастер ноде, для этого нужно добавить следущую секцию в блока spec containers. Эти инструкции, разрешают данной поде запускаться на мастер ноде, и говорят куберу, что данную поду нужно разворачивать на ноде с меткой node-role.kubernetes.io/master
```yaml
nodeSelector:
  node-role.kubernetes.io/master: ""
tolerations:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  operator: Equal
``