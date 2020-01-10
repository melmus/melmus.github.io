# Blackbox Exporter для мониторинга сайтов



Скачиваем Blackbox Exporter для дальнейшей настройки мониторинга сайтов в Prometheus:

```bash

cd /opt

mkdir /opt/blackbox_exporter

wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.15.1/blackbox_exporter-0.15.1.linux-amd64.tar.gz

tar -C blackbox_exporter --strip 1 -xf blackbox_exporter-*.linux-amd64.tar.gz

rm -f blackbox_exporter-*.linux-amd64.tar.gz

```



Создаем сервис для blackbox и применяем настройки:

```bash

[Unit]

Description=Blackbox Exporter Service

Wants=network-online.target

After=network-online.target



[Service]

User=root

Group=root

Type=Simple

ExecStart=/opt/blackbox_exporter/blackbox_exporter \

    --config.file /opt/blackbox_exporter/blackbox.yml

Restart=always



[Install]

WantedBy=multi-user.target

```

```bash

systemctl daemon-reload && systemctl enable blackbox-exporter.service && systemctl start blackbox-exporter.service

```



Приводим конфигурационный файл blackbox.yml к виду:



```yaml

modules:

  http_2xx:

    prober: http

    timeout: 15s

    http:

      valid_http_versions:

        - "HTTP/1.0"

        - "HTTP/1.1"

        - "HTTP/2"

      method: GET

      valid_status_codes: [200]

      fail_if_ssl: false

      fail_if_not_ssl: false

      tls_config:

