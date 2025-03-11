# mongo monitor

mongo已在nginx部分通过docker-compose进行了安装


## 1 创建监控用户(限制权限)
登陆mongodb创建监控用户，权限为“readAnyDatabase”，如果是cluster环境，需要有权限“clusterMonitor”

* 登录mongodb（docker安装的mongo）
```
docker exec -it mongo mongo admin
```
=>
```
# docker exec -it mongo mongo admin
MongoDB shell version v4.2.5
connecting to: mongodb://127.0.0.1:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("9db947f9-4179-4b18-bf76-6e826b100538") }
MongoDB server version: 4.2.5
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
> 
```

* 登录mongodb（yum和apt安装的mongo）
```
mongo admin
```

* 创建监控用户
```
> db.auth('root','123456')
1
> db.createUser({ user:'exporter',pwd:'password',roles:[ { role:'readAnyDatabase', db: 'admin'},{ role: "clusterMonitor", db: "admin" }]});
#测试 使用上面创建的用户信息进行连接。
> db.auth('exporter', 'password')
1
#表示成功 
> exit
```

=>

```
> db.auth('root','123456')
1
> db.createUser({ user:'exporter',pwd:'password',roles:[ { role:'readAnyDatabase', db: 'admin'},{ role: "clusterMonitor", db: "admin" }]});
Successfully added user: {
	"user" : "exporter",
	"roles" : [
		{
			"role" : "readAnyDatabase",
			"db" : "admin"
		},
		{
			"role" : "clusterMonitor",
			"db" : "admin"
		}
	]
}
> db.auth('exporter', 'password')
1
> exit
bye
```

## 2. MongoDB exporter

### 2.1 二进制安装


### 2.2 docker安装
```
docker run -d -p 9216:9216 -p 17001:17001 --restart=always --name=mongodb-exporter docker.1ms.run/bitnami/mongodb-exporter:latest --mongodb.uri=mongodb://exporter:password@192.168.50.134:27017/admin?ssl=false
```

* 参数解释
flag	                        含义	                            案例
* -h, --help	            显示上下文相关的帮助	
* --[no-]compatible-mode	启用旧的 mongodb-exporter               兼容指标	
* --[no-]discovering-mode	启用自动发现集合	
* --mongodb.collstats-colls	逗号分隔的databases.collections     列表以获取$collStats	--mongodb.collstats-colls=db1,db2.col2
* --mongodb.indexstats-colls	逗号分隔的 databases.collections 列表以获取 $indexStats	--mongodb.indexstats-colls=db1.col1,db2.col2
* --[no-]mongodb.direct-connect	是否应该进行直接连接。如果指定了多个主机或使用了 SRV URI，则直接连接无效	
* --[no-]mongodb.global-conn-pool	使用全局连接池而不是为每个http请求创建新池	

* --mongodb.uri	MongoDB 连接 URI ($MONGODB_URI)	--mongodb.uri=mongodb://user:pass@127.0.0.1:27017/admin?ssl=true
* --web.listen-address	用于侦听 Web 界面和遥测的地址	--web.listen-address=":9216"
* --web.telemetry-path	指标公开路径	--web.telemetry-path="/metrics/"
* --web.config	具有用于基本身份验证的 Prometheus TLS 配置的文件的路径	--web.config=STRING
* --log.level	仅记录具有给定严重性或更高严重性的消息。有效级别: [调试、信息、警告、错误、致命]	--log.level="error"
* --collector.diagnosticdata	启用从 getDiagnosticData 收集指标	
* --collector.replicasetstatus	启用从 replSetGetStatus 收集指标	

* --collector.dbstats	启用从 dbStats 收集指标
* --collector.topmetrics	启用从 top admin command 收集指标
* --collector.indexstats	启用从 $indexStats 收集指标
* --collector.collstats	启用从 $collStats 收集指标
* --collect-all	启用所有收集器。与指定所有 --collector.<name>相同</name>
* --collector.collstats-limit=0	如果有超过<n>个集合，请禁用 collstats、dbstats、topmetrics 和 indexstats 收集器。0 = 无限制</n>

* --metrics.overridedescendingindex	启用降序索引名称覆盖，以将 -1 替换为 _DESC
* --version	显示版本并退出


## 3. 配置Prometheus
* 配置Prometheus拉取指标
```
cat >> /data/docker-prometheus/prometheus/prometheus.yml << "EOF"
  - job_name: 'mongodb_exporter'
    static_configs:
      - targets: ['192.168.50.134:9216']
        labels:
          instance: worker2
EOF
```

* 加载Prometheus配置
```
curl -X POST http://192.168.50.120:9090/-/reload
```

* 检查
![alt text](image.png)


## 4. 常见的指标
* mongodb_ss_connections {conn_type="available"} 可用的连接总数
![alt text](image-2.png)

* mongodb_ss_mem_virtual
* mongodb_ss_mem_resident

### 4.1 关于 server status
* mongodb_up # 服务器是否在线
* mongodb_ss_ok {cl_id="", cl_role="mongod", rs_state="0"} # 服务器是否正常运行，取值为 1、0。标签中记录了 cluster、ReplicaSet 的信息
* mongodb_ss_uptime # 服务器的运行时长，单位为秒
* mongodb_ss_connections {conn_type="current"} # 客户端连接数


### 4.2 关于主机
* mongodb_sys_cpu_num_cpus # 主机的 CPU 核数

### 4.3 关于collection
* mongodb_collstats_storageStats_count {database="xx", collection="xx"} # collection 全部文档的数量
* mongodb_collstats_storageStats_size # collection 全部文档的体积，单位 bytes
* mongodb_collstats_storageStats_storagesize # collection 全部文档占用的磁盘空间，默认会压缩
* delta (mongodb_collstats_latencyStats_reads_ops [1m]) # collection 读操作的数量（每分钟）
* delta (mongodb_collstats_latencyStats_reads_latency [1m]) # collection 读操作的延迟（每分钟），单位为微秒
* mongodb_collstats_latencyStats_write_ops
* mongodb_collstats_latencyStats_write_latency

### 4.4 关于 index
* mongodb_collstats_storageStats_nindexes # collection 的 index 数量
* mongodb_collstats_storageStats_totalIndexSize # collection 的 index 占用的磁盘空间
* delta (mongodb_indexstats_accesses_ops [1m]) # index 被访问次数

### 4.5 关于操作
* delta (mongodb_ss_opcounters [1m]) # 执行各种操作的数量
* delta (mongodb_ss_opLatencies_latency [1m]) # 执行各种操作的延迟，单位为微秒
* delta (mongodb_ss_metrics_document [1m]) # 各种文档的变化数量
关于锁
* delta (mongodb_ss_locks_acquireCount [lock_mode="w"][1m]) # 新加锁的数量。R 表示共享锁，W 表示独占锁，r 表示意向共享锁，w 表示意向独占锁
mongodb_ss_globalLock_currentQueue {count_type="total"} # 被锁阻塞的操作数


![alt text](image-1.png)


## 5. 添加触发器
* 配置Prometheus告警规则
* 因为MongoDB是单点，所以未配置复制触发器
```
cat > /data/docker-prometheus/prometheus/rules/mongodb-exporter.yml <<"EOF"
groups:
- name: PerconaMongodbExporter
  rules:
  - alert: MongodbDown
    expr: 'mongodb_up == 0'
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "MongoDB Down 容器: $labels.instance"
      description: "MongoDB 容器 is down, 当前值:{{ $value }}"
      
  - alert: MongodbNumberCursorsOpen
    expr: 'mongodb_ss_metrics_cursor_open{csr_type="total"} > 10 * 1000'
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "MongoDB 数字游标打开告警 容器: $labels.instance"
      description: "MongoDB 为客户端打开的游标过多 > 10k, 当前值:{{ $value }}"
      
  - alert: MongodbCursorsTimeouts
    expr: 'increase(mongodb_ss_metrics_cursor_timedOut[1m]) > 100'
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "MongoDB 游标超时 容器: $labels.instance"
      description: "太多游标超时，当前值:{{ $value }}"
  
  - alert: MongodbTooManyConnections
    expr: 'avg by(instance) (rate(mongodb_ss_connections{conn_type="current"}[1m])) / avg by(instance) 
  (sum(mongodb_ss_connections) by (instance)) * 100 > 80'
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "MongoDB 太多连接 容器: $labels.instance"
      description: "MongoDB 连接数 > 80%，当前值:{{ $value }}"
      
  - alert: MongodbVirtualMemoryUsage
    expr: '(sum(mongodb_ss_mem_virtual) BY (instance) / sum(mongodb_ss_mem_resident) BY (instance)) > 3'
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "MongoDB虚拟内存使用告警 容器: $labels.instance"
      description: "虚拟内存使用过高，当前值:{{ $value }}"
EOF
```

* 校验Prometheus配置
```
docker exec -it prometheus promtool check config /etc/prometheus/prometheus.yml
```

* 加载Prometheus配置
```
curl -X POST http://192.168.50.120:9090/-/reload
```

## 5. 添加Grafana面板
* 可以下载json文件，然后导入进去，比如如果grafana官网上没有合适的末班，可以到github上寻找

* id: 2583

![alt text](image-3.png)

