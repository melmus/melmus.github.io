# Установка Vault на VM

Установка состоит из нескольких этапов:

### Установка приложения из бинарника

Скачиваем и устанавливаем vault
```bash
useradd vault -d /etc/vault.d -s /bin/false
sudo mkdir /opt/vault
cd /opt/vault
wget -E https://releases.hashicorp.com/vault/1.3.1/vault_1.3.1_linux_amd64.zip
unzip vault_1.3.1_linux_amd64.zip
sudo chown -R vault:vault /opt/vault
ln -s /opt/vault/vault /usr/bin/vault
vault server -dev
```
Сохраняем информациж для unsealing в надежное место:
```bash
 ==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: (not set)
                   Mlock: supported: false, enabled: false
                 Storage: inmem
                 Version: Vault v1.0.2
             Version Sha: 37a1dcdfdf9c477c1c68c022d2084550f25bfdfdf20c

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: 1+yv+v5mz+aSCK67X6slL3ECxb4UDdf35656L8ujWZU/O=
Root Token: s.XmpNPoi9sRhYdfdfdftdKHaQh55
```

### Сервис systemd vault.service
```bash
sudo vim /etc/systemd/system/vault.service


[Unit]
Description=Vault Agent
Requires=network-online.target
After=network-online.target
[Service]
Restart=on-failure
PermissionsStartOnly=true
ExecStartPre=/sbin/setcap 'cap_ipc_lock=+ep' /opt/vault/vault
ExecStart=/opt/vault/vault server -config /etc/vault.d/config.hcl
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM
User=vault
Group=vault
[Install]
WantedBy=multi-user.target
```
### Подключение postgresql в качестве БД
Необходимо создать БД, для этого подключаемся к кластеру postgresql:
```sql
createuser -U postgres vault
createdb -U postgres vaultdb
```

Создаём таблицу vault_kv_store и INDEX к ней:
```sql
CREATE TABLE vault_kv_store (
  parent_path TEXT COLLATE "C" NOT NULL,
  path        TEXT COLLATE "C",
  key         TEXT COLLATE "C",
  value       BYTEA,
  CONSTRAINT pkey PRIMARY KEY (path, key)
);

CREATE INDEX parent_path_idx ON vault_kv_store (parent_path);
```


Далее подключаемся обратно на сервер, где установлен vault:
```bash
vim /etc/vault.d/config.hcl


storage "postgresql" {
  connection_url = "postgres://vault:PASSWORD@IP:5432/vaultdb?sslmode=disable"
}
ui = true
disable_cache = true
disable_mlock = true
api_addr = "http://vault.testdomain.com:8200"
listener "tcp" {
 address     = "0.0.0.0:8200"
 proxy_protocol_behavior = "use_always"
 tls_disable = 1
}
max_lease_ttl = "1h"
default_lease_ttl = "1h"
```
Далее перезапускаем сервис:
```bash
systemctl daemon-reload
systemctl restart vault.service
systemctl enable vault.service
```

### Nginx проксирование
```bash
upstream vault_server {
    server nginx.testdomain.com:8200 fail_timeout=0;
}
server {
    listen 80;
    server_name   vault.testdomain.com;
    access_log    /var/log/nginx/vault.access.log;
    error_log     /var/log/nginx/vault.error.log;
    ignore_invalid_headers off;
    #Setup standard forwarding headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    #Required for new HTTP-based CLI
    proxy_http_version 1.1;
    proxy_request_buffering off;
    #Required for HTTP-based CLI to work over SSL
    proxy_buffering off;
    location / {
        try_files $uri @vault;
    }
  location @vault {
        #add_header Pragma "no-cache";
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        #proxy timeout
        proxy_connect_timeout      90;
        proxy_read_timeout         90;
        proxy_send_timeout         90;
        proxy_pass  http://vault_server;
    }
}
server {
    listen 443 ssl;
    server_name   vault.testdomain.com;
    access_log    /var/log/nginx/vault.access.log;
    error_log     /var/log/nginx/vault.error.log;
    ssl_certificate    /etc/ssl/certs/testdomain.com.crt;
    ssl_certificate_key    /etc/ssl/certs/testdomain.com_dec.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 SSLv3;
    ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
#    ssl on;
    ssl_prefer_server_ciphers on;
    ignore_invalid_headers off;
    #Setup standard forwarding headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    #Required for new HTTP-based CLI
    proxy_http_version 1.1;
    proxy_request_buffering off;
    #Required for HTTP-based CLI to work over SSL
    proxy_buffering off;
    location / {
        try_files $uri @vault;
    }
    location @vault {
        #add_header Pragma "no-cache";
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        #proxy timeout
        proxy_connect_timeout      90;
        proxy_read_timeout         90;
        proxy_send_timeout         90;
        proxy_pass  http://vault_server;
    }
}
```
