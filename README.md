# ELK
[toc]
# 一. 组件介绍
- Filebeat:采集数据
- Logstash:接收Filebeat采集的数据并分析
- Elasticsearch:存储来自Logstash的数据
- Kibana:将Elasticsearch存储的数据前台展示给用户，用户可以搜索想要的日志内容
# 二. 部署安装与启动
## 2.1 Kibana安装与启动
- 自行找包放进服务器
- 安装
```
tar -zxf kibana-6.6.0-linux-x86_64.tar.gz
mv kibana-6.6.0-linux-x86_64 /apps/software/kibana-6.6.0
```
- 修改Kibana配置/apps/software/kibana-6.6.0/config/kibana.yml

```
server.port: 5601
server.host: "0.0.0.0"
#elasticsearch.hosts: [“http:// 10.156.14.136:9200”]
#elasticsearch.url: "http://localhost:9200"
#elasticsearch.username: "user"
#elasticsearch.password: "pass"

```

- 后台启动

```
nohup /apps/software/kibana-6.6.0/bin/kibana >/apps/software/log/kibana.log 2>/apps/software/log/kibana.log &
```
- 启动成功之后在浏览器访问5601端口即可查看是否启动成功
## 2.2 elasticsearch安装与启动
- 安装

```
tar -zxf elasticsearch-6.6.0.tar.gz
mv elasticsearch-6.6.0 /apps/software/
```
- 修改配置/usr/local/elasticsearch-6.6.0/config/elasticsearch.yml

```
path.data: /usr/local/elasticsearch-6.6.0/data  #保存数据的路径
path.logs: /usr/local/elasticsearch-6.6.0/logs   #保存日志文件的路径
network.host: 127.0.0.1
http.port: 9200
```
- JVM的内存限制更改jvm.options

```
-Xms128M
-Xmx128M

```
- Elasticsearch的启动，得用普通用户启动

```
useradd -s /sbin/nologin elk	#新建一个elk用户
chown –R tj00184:tj00184 /apps/software/elasticsearch-6.6.0/   #把这个目录下所有文件加上elk用户权限
su - elk -s /bin/bash	#切换到elk用户
```
- 启动命令

```
/apps/software/elasticsearch-6.6.0/bin/elasticsearch -d
```
- ES启动三个报错的处理

```
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
[2]: max number of threads [3829] for user [elk] is too low, increase to at least [4096]
[3]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

```
- 最大文件打开数调整/etc/security/limits.conf

```
* - nofile 65536
* - nproc 10240

```
- 最大打开进程数调整/etc/security/limits.d/20-nproc.conf

```
* - nproc 10240
```
- 内核参数调整/etc/sysctl.conf

```
vm.max_map_count = 262144

```
- 保存之后执行系统写入命令

```
sysctl -p
```
- 如果学习，建议监听在127.0.0.1
- 如果是云服务器的话，一定把9200和9300公网入口在安全组限制一下
- 自建机房的话，建议监听在内网网卡，监听在公网会被入侵

## 2.3 logstash安装和启动
- 安装脚本

```
tar -zxf logstash-6.6.0.tar.gz
 mv logstash-6.6.0 /apps/software/
```
- Logstash的JVM配置文件更新/apps/software/logstash-6.6.0/config/jvm.options

```
-Xms200M
-Xmx200M

```
- Logstash最简单配置

```
input{
  stdin{}
}
output{
  stdout{
    codec=>rubydebug
  }
}

```
- 启动命令

```
nohup /apps/software/logstash-6.6.0/bin/logstash -f /apps/software/logstash-6.6.0/config/logstash.conf >/apps/software/log/logstash.log 2>/apps/software/log/logstash.log &
```
## 2.4 Filebeat安装和启动
- 安装

```
tar -zxf filebeat-6.6.0-linux-x86_64.tar.gz
mv filebeat-6.6.0-linux-x86_64 /apps/software/filebeat-6.6.0

```
- Filebeat发送日志到ES配置/apps/software/filebeat-6.6.0/filebeat.yml

```
filebeat.inputs:
- type: log
  tail_files: true
  backoff: "1s"
  paths:
      - /usr/local/nginx/logs/access.log

output:
  elasticsearch:
    hosts: ["192.168.237.50:9200"]

```
- 后台启动

```
nohup /apps/software/filebeat-6.6.0/filebeat  -e -c /apps/software/filebeat-6.6.0/filebeat.yml >/apps/software/log/filebeat.log 2>&1 &
```
# 三. 终极配置
## 3.1 filebeat配置

```
filebeat.inputs:
- type: log
  tail_files: true
  backoff: "1s"
  paths:
      - /apps/soft/newProduct/server/work/log/weixin-dubbo-edor-service.out
  fields:
    type: edor
  fields_under_root: true
- type: log
  tail_files: true
  backoff: "1s"
  paths:
      - /apps/soft/newProduct/server/work/log/weixin-dubbo-tencentface-service.out
  fields:
    type: tencentface
  fields_under_root: true
output:
  logstash:
    hosts: ["10.156.14.136:5044"]

```
## 3.2 logstash配置

```
input {
  beats {
    host => '0.0.0.0'
    port => 5044
  }
}
filter {
    if [type] == "edor" or "tencentface"
    {
        grok {
                 match => {
                        "message" => '(?<requesttime>[0-9]{1,4}-[0-9]{1,2}-[0-9]{1,2} [0-9]{1,2}:[0-9]{1,2}:[0-9]{1,2}.[0-9]{1,4})[ ]+(?<log_level>[^ ]+) (?<thread>[0-9]{1,7} --- \[.+\]) (?<class>[^ ]+)[ ]+: (?<service>.+)'
                 }
                 remove_field => ["message","@version","source","tags","beat","class"]
        }
        date {
        match => ["requesttime","yyyy-MM-dd HH:mm:ss.SSS"]
        target => "@timestamp"
        }
    }
}
output{
    if [type] == "edor"{
        elasticsearch {
                hosts => ["http://10.156.14.136:9200"]
                index => "edor-%{+YYYY.MM.dd}"
        }
                stdout {
                codec => rubydebug
        }
    }
    if [type] == "tencentface"{
        elasticsearch {
                hosts => ["http://10.156.14.136:9200"]
                index => "tencentface-%{+YYYY.MM.dd}"
        }
                stdout {
                codec => rubydebug
        }
    }
}

```

















