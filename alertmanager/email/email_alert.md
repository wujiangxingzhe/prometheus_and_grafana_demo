# email alert

## 1. 开通邮箱的SMTP/POP3
* 获取token
* 可参考Jenkins部分

## 2. 配置alertmanager
```
cd /data/docker-prometheus
```
* 配置alertmanager
```
cat > alertmanager/alertmanager.yml<<"EOF"
global:
  # 邮件服务器
  smtp_smarthost: 'smtp.163.com:465'
  # 发邮件的邮箱
  smtp_from: 'test@163.com'
  # 发邮件的邮箱用户名，也就是你的邮箱
  smtp_auth_username: 'test@163.com'
  # 发邮件的邮箱密码
  smtp_auth_password: 'your-password'
  #tls 验证配置，false 为关闭
  smtp_require_tls: false

route:
  group_by: ['alertname']
  # 当收到告警的时候，等待group_wait配置的时间10s，看是否还有告警，如果有就一起发出去
  group_wait: 10s
  # 如果上次告警信息发送成功，此时又来了一个新的告警数据，则需要等待group_interval配置的时间才可以发送出去
  group_interval: 10s
  # 如果上次告警信息发送成功，且问题没有解决，则等待 repeat_interval配置的时间再次发送告警数据
  repeat_interval: 4h
  # 全局报警组，这个参数是必选的，和下面报警组名要相同
  receiver: 'email'

receivers:
  - name: 'email'
    #收邮件的邮箱
    email_configs:
    - to: 'xxx@qq.com'
      send_resolved: true
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
EOF
```

* 加载alertmanager配置
```
curl -X POST http://192.168.50.120:9093/-/reload
```


## 4. 使用模板

### 4.1 创建模板文件
* 创建目录存放模板
```
mkdir /data/docker-prometheus/alertmanager/template
cd /data/docker-prometheus/alertmanager/template
```

* 创建模板文件
```
cat > alertmanager/template/email.tmpl <<"EOF"
{{ define "email.html" }}
{{- if gt (len .Alerts.Firing) 0 -}}{{ range .Alerts }}
<h2>@告警通知</h2>
告警程序：prometheus_alert <br>
告警级别：{{ .Labels.severity }} 级 <br>
告警类型：{{ .Labels.alertname }} <br>
故障主机：{{ .Labels.instance }} <br>
告警主题：{{ .Annotations.summary }} <br>
告警详情：{{ .Annotations.description }} <br>
触发时间：{{ .StartsAt.Local.Format "2006-01-02 15:04:05" }} <br>
{{ end }}{{ end -}}
{{- if gt (len .Alerts.Resolved) 0 -}}{{ range .Alerts }}
<h2>@告警恢复</h2>
告警程序：prometheus_alert <br>
故障主机：{{ .Labels.instance }}<br>
故障主题：{{ .Annotations.summary }}<br>
告警详情：{{ .Annotations.description }}<br>
告警时间：{{ .StartsAt.Local.Format "2006-01-02 15:04:05"}}<br>
恢复时间：{{ .EndsAt.Local.Format "2006-01-02 15:04:05" }}<br>
{{ end }}{{ end -}}
{{- end }}
EOF
```

* 将模板文件路径信息添加到alertmanager
```
...
templates:
  - '/etc/alertmanager/template/*.tmpl'
...
receivers:
  - name: 'email'
    #收邮件的邮箱
    email_configs:
    - to: 'xxx@qq.com'
      send_resolved: true
      html: '{{ template | "email.html" . }}'
```

* 加载alertmanager配置
```
curl -X POST http://192.168.50.120:9093/-/reload
```