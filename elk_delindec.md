# Удаление старых индексов
Для удаления старых индексов можно использовать curator.

Используется два файла, первый delete_indices.yml описывает, что будем делать

```yaml
actions:
  1:
    action: delete_indices
    description: >-
      Delete indices older than 30 days
    options:
      ignore_empty_list: True
      timeout_override: 30000
      continue_if_exception: False
      disable_action: False
    filters:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 30
      exclude:
```

Второй конфигурационный config.yaml, который описывает подключение к эластику
```yaml
---
client:
  hosts:
    - 127.0.0.1
  port: 9200
  url_prefix:
  use_ssl: False
  certificate:
  client_cert:
  client_key:
  ssl_no_validate: False
  http_auth:
  timeout: 30
  master_only: False
 
logging:
  loglevel: INFO
  logfile:
  logformat: default
  blacklist: ['elasticsearch', 'urllib3']
```

Далее запускаем curator 
```bash
curator delete_indices.yml --config config.yml
```
Можно использовать опцию --dry-run при которой удаляться ничего не будет, но покажет список, индексов попадающих под правило
