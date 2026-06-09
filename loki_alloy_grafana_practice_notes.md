# Loki + Alloy + Grafana 实操学习笔记

> 主题：Docker Compose 环境下使用 Loki + Alloy + Grafana 采集与查询容器日志  
> 适合阶段：已经完成 Prometheus + Alertmanager + Webhook 告警链路后继续学习日志系统  
> 实验环境：WSL / Linux / Docker Compose / Grafana / Loki / Alloy  
> 目标：让 Docker 容器日志能够被 Alloy 采集、写入 Loki，并在 Grafana Explore 中查询。

---

## 一、学习目标

上一阶段你已经完成了：

```text
Prometheus 发现 node-exporter 异常
        ↓
Prometheus 告警规则进入 Firing
        ↓
Alertmanager 接收告警
        ↓
Alertmanager 调用 alert-webhook
        ↓
Webhook 容器日志中出现 POST /alert 200
```

这一阶段继续学习日志系统，目标是跑通：

```text
Docker 容器产生日志
        ↓
Alloy 通过 Docker socket 发现容器
        ↓
Alloy 读取容器 stdout / stderr 日志
        ↓
Alloy 给日志打上 container、service_name、platform 等标签
        ↓
Alloy 将日志推送到 Loki
        ↓
Grafana 连接 Loki 数据源
        ↓
在 Grafana Explore 中使用 LogQL 查询日志
```

学完后应能理解：

1. Prometheus 负责指标，Loki 负责日志。
2. Alertmanager 负责告警分组、路由、抑制和通知。
3. Alloy 负责采集 Docker 容器日志并发送到 Loki。
4. Grafana 负责统一展示 Prometheus 指标和 Loki 日志。
5. Grafana Explore 中如何使用 LogQL 查询日志。
6. 出现 `No data`、YAML 报错、Webhook 不显示 JSON 时如何排查。

---

## 二、整体架构

本实验最终包含这些服务：

| 服务 | 作用 | 宿主机访问端口 |
|---|---|---|
| Prometheus | 采集指标、计算告警规则 | `9090` |
| Alertmanager | 接收告警、分组、路由、抑制和通知 | `9093` |
| node-exporter | 暴露主机指标，用于模拟故障 | `9100` |
| alert-webhook | 本地 Flask Webhook 接收器 | `5000` |
| Loki | 日志存储与查询系统 | 不暴露或 `3101:3100` |
| Alloy | 日志采集器，读取 Docker 容器日志 | `12345` |
| Grafana | 展示 Prometheus 指标和 Loki 日志 | `3000` |

核心链路：

```text
Docker 容器日志
        ↓
Alloy
        ↓
Loki
        ↓
Grafana Explore
```

结合上一阶段告警链路，完整流程是：

```text
node-exporter 停止
        ↓
Prometheus 检测到 up{job="node"} == 0
        ↓
Alertmanager 发送告警到 alert-webhook
        ↓
alert-webhook 打印告警日志
        ↓
Alloy 采集 alert-webhook 容器日志
        ↓
Loki 存储日志
        ↓
Grafana 查询日志
```

---

## 三、目录结构

进入已有实验目录：

```bash
cd ~/monitor-lab
```

新增 Alloy 和 Grafana 配置目录：

```bash
mkdir -p alloy
mkdir -p grafana/provisioning/datasources
```

推荐目录结构：

```text
monitor-lab/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       └── node_rules.yml
├── alertmanager/
│   └── alertmanager.yml
├── alert-webhook/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
├── alloy/
│   └── config.alloy
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasources.yml
```

---

## 四、Loki、Alloy、Grafana 的关系

### 1. Loki 是什么

Loki 是日志系统，主要负责：

```text
接收日志
存储日志
根据标签查询日志
按照时间范围返回日志内容
```

它和 Prometheus 很像，但关注对象不同：

| 组件 | 处理对象 | 查询语言 | 主要用途 |
|---|---|---|---|
| Prometheus | 指标 | PromQL | CPU、内存、服务存活、告警 |
| Loki | 日志 | LogQL | 容器日志、错误日志、访问日志 |

Prometheus 查询示例：

```promql
up{job="node"}
```

Loki 查询示例：

```logql
{container="alert-webhook"}
```

---

### 2. Alloy 是什么

Alloy 可以理解为采集器，类似以前常见的 Promtail，但现在 Grafana 官方更推荐用 Alloy 作为统一采集组件。

在本实验中 Alloy 做这些事：

```text
连接 Docker socket
        ↓
发现 Docker 容器
        ↓
读取容器日志
        ↓
添加标签
        ↓
转发到 Loki
```

关键点：

```text
Alloy 要读取 Docker 容器日志，就需要挂载 /var/run/docker.sock
```

---

### 3. Grafana 是什么

Grafana 是展示层。

在本实验中 Grafana 同时连接两个数据源：

```text
Prometheus：查看指标
Loki：查看日志
```

也就是说：

```text
Prometheus + Alertmanager 让你知道“出问题了”
Loki + Alloy + Grafana 让你知道“为什么出问题”
```

---

## 五、编写 Alloy 配置

创建文件：

```bash
nano alloy/config.alloy
```

写入：

```hcl
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}

discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

discovery.relabel "docker_logs" {
  targets = []

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "service_name"
  }
}

loki.source.docker "containers" {
  host          = "unix:///var/run/docker.sock"
  targets       = discovery.docker.containers.targets
  labels        = {"platform" = "docker"}
  relabel_rules = discovery.relabel.docker_logs.rules
  forward_to    = [loki.write.local.receiver]
}
```

保存 nano：

```text
Ctrl + O
回车
Ctrl + X
```

---

## 六、Alloy 配置解释

### 1. `loki.write`

```hcl
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

作用：定义日志最终写入的位置。

这里写的是：

```text
http://loki:3100/loki/api/v1/push
```

注意不要写成：

```text
http://localhost:3100/loki/api/v1/push
```

原因：Alloy 和 Loki 都在 Docker Compose 网络中，容器之间通过服务名访问。

正确理解：

```text
alloy 容器访问 loki 容器：loki:3100
```

---

### 2. `discovery.docker`

```hcl
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}
```

作用：通过 Docker socket 发现当前 Docker 中有哪些容器。

Docker socket 可以理解为 Docker 的控制入口。

Alloy 通过它可以知道：

```text
当前有哪些容器
容器叫什么名字
容器是否运行
容器有哪些 Docker 元数据
```

---

### 3. `discovery.relabel`

```hcl
discovery.relabel "docker_logs" {
  targets = []

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "service_name"
  }
}
```

作用：把 Docker 自动发现出的元数据转换成 Loki 查询时好用的标签。

Docker 容器名通常是：

```text
/alert-webhook
/prometheus
/loki
/grafana
```

这段配置会去掉前面的 `/`，生成：

```text
container="alert-webhook"
service_name="alert-webhook"
```

后面在 Grafana 中就可以查询：

```logql
{container="alert-webhook"}
```

或：

```logql
{service_name="alert-webhook"}
```

---

### 4. `loki.source.docker`

```hcl
loki.source.docker "containers" {
  host          = "unix:///var/run/docker.sock"
  targets       = discovery.docker.containers.targets
  labels        = {"platform" = "docker"}
  relabel_rules = discovery.relabel.docker_logs.rules
  forward_to    = [loki.write.local.receiver]
}
```

作用：真正读取 Docker 容器日志。

逐项解释：

| 配置 | 含义 |
|---|---|
| `host` | 连接 Docker daemon |
| `targets` | 要读取哪些 Docker 容器日志 |
| `labels` | 给日志统一添加静态标签 |
| `relabel_rules` | 把 Docker 元数据转换为 Loki 标签 |
| `forward_to` | 把日志转发给 `loki.write.local` |

完整流程：

```text
Docker 容器 stdout / stderr
        ↓
loki.source.docker
        ↓
loki.write.local.receiver
        ↓
Loki API: /loki/api/v1/push
        ↓
Loki 存储日志
```

---

## 七、配置 Grafana 数据源

创建文件：

```bash
nano grafana/provisioning/datasources/datasources.yml
```

写入：

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

保存：

```text
Ctrl + O
回车
Ctrl + X
```

解释：

```yaml
name: Prometheus
type: prometheus
url: http://prometheus:9090
```

表示 Grafana 通过 Docker Compose 服务名访问 Prometheus。

```yaml
name: Loki
type: loki
url: http://loki:3100
```

表示 Grafana 通过 Docker Compose 服务名访问 Loki。

不能写：

```yaml
url: http://localhost:3100
```

因为 Grafana 在容器里面，容器里的 `localhost` 指向 Grafana 自己，不是 Loki。

---

## 八、完整 docker-compose.yml

如果之前 `docker-compose.yml` 出现：

```text
yaml: line 1: did not find expected key
```

说明 YAML 文件格式错误，常见原因是：

```text
1. 第一行不是 services:
2. 把 ```yaml 代码块标记复制进去了
3. 使用了中文冒号 ： 而不是英文冒号 :
4. 使用了 Tab 缩进
5. 缩进层级错误
```

可以直接重写：

```bash
cp docker-compose.yml docker-compose.yml.bak
```

然后：

```bash
cat > docker-compose.yml <<'EOF'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    depends_on:
      - node-exporter
      - alertmanager
      - loki

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"

  alert-webhook:
    build:
      context: ./alert-webhook
    container_name: alert-webhook
    ports:
      - "5000:5000"

  loki:
    image: grafana/loki:latest
    container_name: loki
    command:
      - "-config.file=/etc/loki/local-config.yaml"
    volumes:
      - loki-data:/loki

  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    ports:
      - "12345:12345"
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - /etc/alloy/config.alloy
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
      - loki

volumes:
  grafana-data:
  loki-data:
