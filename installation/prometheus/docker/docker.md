# install Prometheus with docker

## 1. 安装docker

### 1.1 镜像加速
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
EOF
```

### 1.2 安装docker
```
export DOWNLOAD_URL="http://mirrors.163.com/docker-ce"
curl -fsSL https://get.docker.com/ | sh
```

### 1.3 检查
```
docker -v
或：
systemctl status docker
```

## 2. 安装docker-compose


## 3. 安装Prometheus/Grafana/Alertmanager
```
mkdir /data
git clone https://gitee.com/linge365/docker-prometheus.git
cd docker-prometheus
docker-compose up -d
```
* prometheus: 9090
* grafana: 3000
* alertmanager: 9093
* cadvisor: 8080
* node-exporter: 9100

## 4. 使用Grafana展示Prometheus数据

### 4.1 登录Grafana
* 登录Grafana: http://192.168.50.120:3000
* 默认用户名和密码: admin/password

![alt text](image.png)
* 点击`Add data source`

### 4.2 添加Prometheus数据源
添加Prometheus数据源: http://prometheus:9090
* 这里的`prometheus`是Prometheus的容器名，可根据实际情况修改，比如二进制部署的时候，修改成主机的IP地址

![alt text](8ada9fab-745a-4360-9eaa-f70718225ad4.png)

![alt text](image-1.png)

拉到底部，点击`Save & Test`

![alt text](image-2.png)

## 5. 导入Grafana模板
* 导入Grafana模板: https://grafana.com/grafana/dashboards/

![alt text](image-3.png)

![alt text](image-4.png)

在grafana dashboard官网搜索`node exporter`，找到`Node Exporter Full`的模板ID，这里我们选择`Node Exporter Full`的模板ID为`1860`

![alt text](image-5.png)

![alt text](image-6.png)

* 将拷贝的模板ID粘贴到grafana的模板ID中，点击`Load`

![alt text](image-7.png)

* 选择`Prometheus`数据源，点击`Import`

![alt text](image-8.png)

* 导入后的效果如下

![alt text](image-9.png)

* 这里的`Node Exporter Full`可以通过settings进行修改

![alt text](image-10.png)

![alt text](image-11.png)

