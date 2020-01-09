# Привязвка PerstistentVolume и PersistentVolumeClaim

Для того чтобы сохранять состояние микросервиса необходимо использовать либо StatefulSet либо Deployment + PersistentVolumeClaim + PersistentVolume. В этой инструкции рассматривается второй вариант.

**PersistentVolume** - это болванки дисков, которые заранее подготавливаются администраторами Kubernetes-кластера.

**PersistentVolumeClaim** - это потребители дисков с помощью которых их можно задействовать в Pod или Deployment.
### Автоматическая

По умолчанию, если не указаны параметры описанные в секции Ручная привязка, Kubernetes связывает PersistentVolume и PersistentVolumeClaim на основе совпадения:

    storageClassName совпадают
    labels совпадают
    accessModes совпадают
    Объем дискокого простанства capacity.storage в PersistentVolume больше чем объем запрашиваемого диского пространства requests.storage в PersistentVolumeClaim

### Ручная

Часто автоматическая привязка не всегда желательна и возникает необходимость выполнить ручную привязку.

Для этого со стороны PersistentVolume используется параметр claimRef или со стороны PersistentVolumeClaim параметр volumeName как в примере ниже.

 
Инструкция

Создаем папки для диска:
```bash
sudo -u kube mkdir -p /kuber/volumes/namespace/microservice1
```
Подготавливаем болванку диска:
```yaml
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: microservice1-pv-volume
  labels:
    type: nfs
    app: namespace-microservice1
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/kuber/volumes/namespace/microservice1"
  claimRef:
    name: microservice1-pv-claim
    namespace: namespace
```

```bash
sudo -u kube kubectl apply -f microservice1-pv-volume.yml
```

Обратите внимание что у PersistentVolume нет namespace, потому что они создаются глобально на весь кластер.

 

Подготоваливаем манифест PersistentVolumeClaim, который передадим продуктовой команде запросившей диск:
```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: microservice1-pv-claim
  labels:
    app: namespace-microservice1
  namespace: namespace
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  volumeName: microservice1-pv-volume
  selector:
    matchLabels:
      type: nfs
      app: namespace-microservice1
```
