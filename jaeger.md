# Установка Jaeger 

Jaeger представляет из себя три/четыре сервиса, которые взаимодействуют между собой

По факту используется схема без кафки, а потому в ходу три сервиса collector - альтернатива logstash-у, query - альтернатива kibana и agent - альтернатива filebeat. В качестве базы данных можно использовать локальный Badger, Cassandra или ElasticSearch. Используем Elasticsearch.

Architecture

Для работы Docker необходимо запустить сервис firewalld
```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-port=9100/tcp --add-port=9200/tcp --add-port=5400-5499/tcp --add-port=80/tcp --add-port=9300/tcp
firewall-cmd --add-port=9100/tcp --add-port=9200/tcp --add-port=5400-5499/tcp --add-port=80/tcp --add-port=9300/tcp --permanent
```

По-умолчанию, jaegger поставляется в виде докер-контейнеров, и похоже предусмотрен для работы в kubernetes. Развернем докер на отдельном сервере и подняли три контейнера.

### Agent:

| Port | Protocol | Function |
| --- |:---:| ---:|
| 6831 | UDP |  accept jaeger.thrift in compact Thrift protocol used by most current Jaeger clients |
| 6832 | UDP |  accept jaeger.thrift in binary Thrift protocol used by Node.js Jaeger client (because thriftrw npm package does not support compact protocol) |
| 5778 | HTTP | serve configs, sampling strategies |
| 5775 | UDP |  accept zipkin.thrift in compact Thrift protocol (deprecated; only used by very old Jaeger clients, circa 2016)|


```bash
docker run -d \
--name=jaeger_agent \
--restart always \
-p14271:14271 \
-p6831:6831/udp \
-p6832:6832/udp \
-p5778:5778/tcp \
-p5775:5775/udp \
jaegertracing/jaeger-agent:1.12 \
--reporter.grpc.host-port=172.17.0.1:14250
```

### Query:

|Port  |	Protocol | Function |
| --- |:---:| ---:|
| 14270 |	HTTP |	Health check at |
| 14271 |	HTTP |	Metrics endpoint |

```bash
docker run -d \
--restart always \
-p 16686:16686 \
-p 16687:16687 \
-e SPAN_STORAGE_TYPE=elasticsearch \
-e ES_SERVER_URLS=http://elk.testdomain.com:9200 \
--name=jaeger_query \
jaegertracing/jaeger-query:1.12
```

### Collector: 

| Port |	Protocol | Function |
| --- |:---:| ---:|
| 14267 |TChannel|used by jaeger-agent to send spans in jaeger.thrift format|
| 14250 |	gRPC | used by jaeger-agent to send spans in model.proto format
| 14268 |	HTTP | can accept spans directly from clients in jaeger.thrift format over binary thrift protocol
|  9411 |	HTTP | can accept Zipkin spans in JSON or Thrift (disabled by default)
| 14269 |	HTTP | Health check at |

```bash
docker run -d \
--name=jaeger_collector \
--restart always \
-p 14267:14267 \
-p 14268:14268 \
-p 14269:14269 \
-p 14250:14250 \
-e SPAN_STORAGE_TYPE=elasticsearch \
-e ES_SERVER_URLS=http://0700ngenie-infr-solut-dbe01.ngenie.mtsit.com:9200 \
jaegertracing/jaeger-collector:1.12 \
--collector.queue-size=10000 --collector.num-workers=100
```
## keepalived (vIP) (include):
Настройка общего IP для нескольких серверов jaeger с использованием keepalived

### Настройка
#### Server #1

```bash
apt install keepalived -y || yum install -y keepalived
vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived.yml
global_defs {
    router_id jaeger-1
}
vrrp_script "jaeger_collector_check" {
        script "/usr/bin/nc -z 10.10.10.30 14250"
        interval 1
        script_user root
        fall 2
        rise 2
}
vrrp_script "jaeger_query_check" {
        script "/usr/bin/nc -z 10.10.10.30 16686"
        interval 1
        script_user root
        fall 2
        rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface "ens33"
    virtual_router_id "21"
    priority 99
    authentication {
        auth_type PASS
        auth_pass "1111"
    }
    track_script {
        "jaeger_collector_check"
        "jaeger_query_check"
    }
    virtual_ipaddress {
        "10.10.10.35"
    }
}
 
echo net.ipv4.ip_nonlocal_bind = 1  >>  /etc/sysctl.conf
sysctl -p
systemctl restart keepalived.service
systemctl enable keepalived.service
```
#### Server #2
```bash
apt install keepalived -y || yum install -y keepalived
vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived.yml
global_defs {
    router_id jaeger-2
}
vrrp_script "jaeger_collector_check" {
        script "/usr/bin/nc -z 10.10.10.31 14250"
        interval 1
        script_user root
        fall 2
        rise 2
}
vrrp_script "jaeger_query_check" {
        script "/usr/bin/nc -z 10.10.10.31 16686"
        interval 1
        script_user root
        fall 2
        rise 2
}
vrrp_instance VI_2 {
    state MASTER
    interface "ens33"
    virtual_router_id "21"
    priority 99
    authentication {
        auth_type PASS
        auth_pass "1111"
    }
    track_script {
        "jaeger_collector_check"
        "jaeger_query_check"
    }
    virtual_ipaddress {
        "10.10.10.35"
    }
}
 
 
echo net.ipv4.ip_nonlocal_bind = 1  >>  /etc/sysctl.conf
sysctl -p
systemctl restart keepalived.service
systemctl enable keepalived.service
```
