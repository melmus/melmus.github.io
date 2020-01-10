# Отправка логов job's из jenkins в ELK        
В качестве примера будем пересылать логи из jenkins job test-application001-default
Для пересылки логов, на jenkins.testcomain.com установлен filebeat 7.2.0. 
В конфиг файле ```/etc/filebeat/filebeat.yml``` е настройки. 
Включим логирование и укажем путь до файла логов конкретной job-ы
 

```bash
- type: log
  # Change to true to enable this input configuration.
  enabled: true
  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /data/jenkins/jobs/ngcs-solutions-dev/jobs/test-001/jobs/test-application001-default/builds/*/log
```

Укажем путь до logstash-сервера:

```bash
output.logstash:
 # The Logstash hosts
 hosts: ["10.10.10.40:5047"] 
```

С файлбитом пока закончили. Включаем службу filebeat

Теперь переключаемся на logstash

В папке ```/etc/logstash/``` создаём папку для конфигов под конкретную 
ситуацию. Например, для сбора джоб-логов с дженкинса я создал папку 
jenkins. В папке создаём конфиг файл

```bash
input { beats 
        {
         port => 5047    
         }
      }
 
output {
  stdout { codec => rubydebug }
  elasticsearch { hosts => ["localhost"] }
}
```
Cохраняем, перезагружаем службу logstash.

Заходим в kibana.

Идём в Management / Index management

Ищем индекс "logstash". Есть? Отлично! 
