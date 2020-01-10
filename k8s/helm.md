# Установка Helm2

Скачиваем скрипт установки helm:
```bash
export http_proxy=http://LOGIN:PASSWORD@HOST:PORT
export https_proxy=${http_proxy}
export no_proxy=`ifconfig ens192 | grep inet | sed 's/\ \{1,\}/ /g' | cut -d' ' -f 3`
curl -o get_helm.sh -LO https://raw.githubusercontent.com/helm/helm/master/scripts/get
export PATH=$PATH:/usr/local/bin
```
Запускаем установщик:
```bash
chmod +x get_helm.sh
sudo -E bash get_helm.sh --version v2.16.1
```

 Делаем костыль, т.к. в Oracle Linux папка /usr/local/bin
```bash
sudo ln -s /usr/local/bin/helm /usr/bin/helm
sudo ln -s /usr/local/bin/tiller /usr/bin/tiller
```

 Создаем ServiceAccount для Tiller и выдаем ему админские права (это не достаток Helm 2, который устранен в Helm 3.0+):
```bash
sudo -u kube kubectl --namespace kube-system create serviceaccount tiller
sudo -u kube kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

Настраиваем Helm для пользователя kube:
```bash
sudo -E -u kube helm init --service-account tiller
```

## Использование

Установка Apache Ignite:
```bash
sudo -u kube helm install --name apache-ignite --namespace perfmgmnt-dev --set persistence.enabled=false --set wal_persistence.enabled=false --set resources.requests.cpu=1 --set resources.requests.memory=1Gi --set resources.limits.cpu=1 --set resources.limits.memory=1Gi stable/ignite
```
