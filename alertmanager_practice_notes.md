# Prometheus Alertmanager 实操学习笔记

> 主题：Prometheus 告警规则 + Alertmanager + Webhook 告警接收器  
> 适合阶段：零基础入门监控告警链路  
> 实验环境：Docker Compose / WSL / Linux 终端

---

## 一、学习目标

本阶段的目标不是一开始就配置邮箱、钉钉、企业微信，而是先把最基础、最重要的告警链路跑通：

```text
Prometheus 发现问题
        ↓
Prometheus 告警规则进入 Firing
        ↓
Prometheus 把告警发送给 Alertmanager
        ↓
Alertmanager 分组、路由、抑制
        ↓
Alertmanager 调用 Webhook
        ↓
Flask Webhook 服务打印告警内容
```

学完本阶段后，应能理解：

1. Prometheus 如何通过规则判断服务异常。
2. `up == 0` 的含义。
3. `Pending` 和 `Firing` 的区别。
4. Alertmanager 的分组、路由、重复通知和抑制规则。
5. Webhook 如何接收 Alertmanager 发出的告警。
6. 如何通过停止 `node-exporter` 来模拟真实故障。
7. 如何排查端口占用、容器未启动、告警未发送等问题。

---

## 二、整体架构

本实验使用 5 个服务：

| 服务 | 作用 | 访问端口 |
|---|---|---|
| Prometheus | 采集指标、计算告警规则 | `9090` |
| node-exporter | 暴露主机指标，用于模拟被监控服务 | `9100` |
| Alertmanager | 接收 Prometheus 告警，进行分组、路由、抑制和通知 | `9093` |
| alert-webhook | 用 Flask 写的本地 Webhook 接收器 | `5000` |
| Loki | 日志系统组件，当前阶段不是重点 | `3100` 或不暴露 |

核心链路：

```text
node-exporter 被停止
        ↓
Prometheus 抓取失败
        ↓
up{job="node"} 变成 0
        ↓
告警规则触发
        ↓
Alertmanager 接收告警
        ↓
Webhook 打印告警 JSON
```

---

## 三、目录结构

建议先创建一个独立目录：

```bash
mkdir -p monitor-lab
cd monitor-lab
```

创建子目录：

```bash
mkdir -p prometheus/rules
mkdir -p alertmanager
mkdir -p alert-webhook
```

最终目录结构：

```text
monitor-lab/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       └── node_rules.yml
├── alertmanager/
│   └── alertmanager.yml
└── alert-webhook/
    ├── app.py
    ├── requirements.txt
    └── Dockerfile
```

---

## 四、编写 Prometheus 配置

创建文件：

```bash
nano prometheus/prometheus.yml
```

写入：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090

  - job_name: node
    static_configs:
      - targets:
          - node-exporter:9100

  - job_name: alertmanager
    static_configs:
      - targets:
          - alertmanager:9093

  - job_name: loki
    static_configs:
      - targets:
          - loki:3100
```

保存 nano：

```text
Ctrl + O
回车
Ctrl + X
```

---

## 五、Prometheus 配置解释

### 1. 全局配置

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
```

含义：

```text
scrape_interval: 15s
表示 Prometheus 每 15 秒抓取一次指标。

evaluation_interval: 15s
表示 Prometheus 每 15 秒计算一次规则。
```

也就是说 Prometheus 会每隔 15 秒检查一次：

```text
prometheus 是否正常
node-exporter 是否正常
alertmanager 是否正常
loki 是否正常
```

---

### 2. Alertmanager 地址

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

含义：

```text
Prometheus 一旦发现告警触发，就把告警发给 alertmanager:9093。
```

这里的 `alertmanager` 是 Docker Compose 中的服务名。

在 Docker Compose 网络中，容器之间可以通过服务名互相访问：

```text
prometheus → alertmanager:9093
prometheus → node-exporter:9100
alertmanager → alert-webhook:5000
```

---

### 3. 告警规则文件路径

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml
```

含义：

```text
Prometheus 启动时会加载 /etc/prometheus/rules/ 目录下的所有 .yml 规则文件。
```

注意：

宿主机路径是：

```text
./prometheus/rules/node_rules.yml
```

容器内部路径是：

```text
/etc/prometheus/rules/node_rules.yml
```

这需要通过 Docker Compose 的 `volumes` 做路径映射。

---

### 4. 抓取配置

```yaml
scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - node-exporter:9100
```

含义：

```text
Prometheus 会抓取 node-exporter:9100 这个目标。
```

其中：

```text
job_name: node
```

会变成 Prometheus 指标中的标签：

```text
job="node"
```

所以后面的告警规则可以写：

```promql
up{job="node"} == 0
```

---

## 六、编写 Prometheus 告警规则

创建文件：

```bash
nano prometheus/rules/node_rules.yml
```

写入：

```yaml
groups:
  - name: node-exporter-alerts
    rules:
      - alert: NodeExporterDown
        expr: up{job="node"} == 0
        for: 30s
        labels:
          severity: critical
          team: ops
        annotations:
          summary: "node-exporter is down"
          description: "Prometheus cannot scrape node-exporter for more than 30 seconds."

      - alert: NodeExporterDownWarning
        expr: up{job="node"} == 0
        for: 30s
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "node-exporter warning"
          description: "This is a warning alert used to test Alertmanager inhibition."
```

---

## 七、告警规则解释

### 1. 规则组

```yaml
groups:
  - name: node-exporter-alerts
```

含义：

```text
node-exporter-alerts 这一组专门存放 node-exporter 相关告警。
```

以后可以继续扩展：

```text
nginx-alerts
mysql-alerts
docker-alerts
disk-alerts
memory-alerts
```

---

### 2. 告警名称

```yaml
- alert: NodeExporterDown
```

含义：

```text
这个告警的名字叫 NodeExporterDown。
```

在 Prometheus、Alertmanager、Webhook JSON 中都会看到这个名字。

---

### 3. 告警表达式

```yaml
expr: up{job="node"} == 0
```

`up` 是 Prometheus 自动生成的指标，用来表示某个 target 是否抓取成功。

| `up` 值 | 含义 |
|---|---|
| `up == 1` | 抓取成功，服务正常 |
| `up == 0` | 抓取失败，服务异常 |

所以：

```promql
up{job="node"} == 0
```

表示：

```text
Prometheus 无法抓取 job="node" 的目标。
```

在当前实验中，也就是：

```text
node-exporter 不可达或已停止。
```

---

### 4. for 字段

```yaml
for: 30s
```

含义：

```text
异常条件必须持续 30 秒，才真正触发告警。
```

Prometheus 告警常见状态：

| 状态 | 含义 |
|---|---|
| Inactive | 没有异常 |
| Pending | 条件满足，但还没有持续够 `for` 时间 |
| Firing | 条件持续满足，正式触发告警 |

例如：

```text
node-exporter 刚停止
        ↓
Prometheus 检测到 up == 0
        ↓
进入 Pending
        ↓ 等待 30 秒
进入 Firing
```

`for` 的作用是减少误报，避免服务短暂波动就立即报警。

---

### 5. labels

```yaml
labels:
  severity: critical
  team: ops
```

含义：

```text
labels 是给机器用的标签。
```

Alertmanager 会根据 labels 进行：

```text
路由
分组
抑制
分类
通知选择
```

常见告警等级：

```text
info
warning
critical
```

---

### 6. annotations

```yaml
annotations:
  summary: "node-exporter is down"
  description: "Prometheus cannot scrape node-exporter for more than 30 seconds."
```

含义：

```text
annotations 是给人看的说明文本。
```

区别：

| 字段 | 用途 |
|---|---|
| labels | 给机器判断 |
| annotations | 给人阅读 |

以后发送到邮件、企业微信、钉钉中的告警正文，通常就来自 annotations。

---

## 八、编写 Alertmanager 配置

创建文件：

```bash
nano alertmanager/alertmanager.yml
```

写入：

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: webhook-demo
  group_by:
    - job
    - instance
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 3m

  routes:
    - matchers:
        - severity="critical"
      receiver: webhook-demo

receivers:
  - name: webhook-demo
    webhook_configs:
      - url: http://alert-webhook:5000/alert
        send_resolved: true

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal:
      - instance
```

Alertmanager 配置主要包括三类内容：

```text
1. route：告警怎么分发
2. receivers：告警发给谁
3. inhibit_rules：哪些告警要被抑制
```

---

## 九、Alertmanager 配置解释

### 1. global

```yaml
global:
  resolve_timeout: 5m
```

含义：

```text
和告警恢复判断有关。
```

初学阶段不用重点纠结，先理解它与 resolved 状态有关即可。

---

### 2. 默认路由

```yaml
route:
  receiver: webhook-demo
```

含义：

```text
所有没有被其他子路由匹配的告警，默认都发给 webhook-demo。
```

---

### 3. group_by

```yaml
group_by:
  - job
  - instance
```

含义：

```text
相同 job、相同 instance 的告警会合并为一组发送。
```

例如：

```text
NodeExporterDown
NodeExporterDownWarning
```

如果它们都来自：

```text
job="node"
instance="node-exporter:9100"
```

就会被 Alertmanager 归为同一组。

分组的目的：

```text
减少告警风暴，避免一次故障产生大量重复通知。
```

---

### 4. group_wait

```yaml
group_wait: 10s
```

含义：

```text
第一个告警出现后，先等待 10 秒，看有没有同类告警一起合并发送。
```

---

### 5. group_interval

```yaml
group_interval: 30s
```

含义：

```text
同一组告警已经发过一次后，如果这一组里又出现新的告警，至少等 30 秒再发送下一次。
```

---

### 6. repeat_interval

```yaml
repeat_interval: 3m
```

含义：

```text
如果故障一直没有恢复，每隔 3 分钟重复通知一次。
```

---

### 7. 子路由 routes

```yaml
routes:
  - matchers:
      - severity="critical"
    receiver: webhook-demo
```

含义：

```text
如果告警的 severity 是 critical，就发给 webhook-demo。
```

当前配置中，默认 receiver 也是 `webhook-demo`，所以看起来有些重复。但它的学习意义很大。

以后可以改成：

```yaml
routes:
  - matchers:
      - severity="critical"
    receiver: phone-call

  - matchers:
      - severity="warning"
    receiver: email
```

即：

```text
critical → 电话 / 企业微信
warning  → 邮件
info     → 仅记录
```

---

### 8. receivers

```yaml
receivers:
  - name: webhook-demo
    webhook_configs:
      - url: http://alert-webhook:5000/alert
        send_resolved: true
```

含义：

```text
Alertmanager 把告警 POST 到 http://alert-webhook:5000/alert。
```

注意：

这里应该写容器内部服务名：

```text
alert-webhook:5000
```

不要写：

```text
localhost:5000
```

因为对于 Alertmanager 容器来说，`localhost` 是它自己，不是 Webhook 容器。

---

### 9. send_resolved

```yaml
send_resolved: true
```

含义：

```text
告警恢复时，也向 Webhook 发送 resolved 通知。
```

所以：

```text
停止 node-exporter → 收到 firing
重新启动 node-exporter → 收到 resolved
```

---

### 10. inhibit_rules

```yaml
inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal:
      - instance
```

含义：

```text
如果同一个 instance 已经有 critical 告警，
那么同一个 instance 上的 warning 告警就被抑制。
```

当前实验中有两个告警：

```text
NodeExporterDown          severity=critical
NodeExporterDownWarning   severity=warning
```

它们表达式一样：

```promql
up{job="node"} == 0
```

所以停止 node-exporter 后，这两个规则都会触发。

但是 Alertmanager 会执行抑制：

```text
critical 存在
        ↓
warning 被抑制
        ↓
最终只发送 critical
```

因此会出现：

```text
Prometheus 页面显示 2 个 Firing
Alertmanager 页面只显示 1 个 alert
Webhook 只收到 critical 的通知
```

这是正常现象，说明 inhibition 生效了。

---

## 十、编写本地 Webhook 告警接收器

创建文件：

```bash
nano alert-webhook/app.py
```

写入：

```python
from flask import Flask, request, jsonify
import json
from datetime import datetime

app = Flask(__name__)

@app.route("/alert", methods=["POST"])
def alert():
    data = request.get_json()

    print("\n========== ALERT RECEIVED ==========", flush=True)
    print("time:", datetime.now().isoformat(), flush=True)
    print(json.dumps(data, indent=2, ensure_ascii=False), flush=True)
    print("====================================\n", flush=True)

    return jsonify({"status": "ok"}), 200

@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

这里加了 `flush=True`，是为了避免 Python 输出被缓冲，导致日志中看不到完整 JSON。

---

## 十一、Webhook 代码解释

### 1. 创建 Flask 服务

```python
app = Flask(__name__)
```

含义：

```text
创建一个简单 Web 服务。
```

---

### 2. 接收 Alertmanager 的 POST 请求

```python
@app.route("/alert", methods=["POST"])
def alert():
```

含义：

```text
当 Alertmanager 访问 /alert，并且使用 POST 方法时，执行 alert() 函数。
```

---

### 3. 读取 JSON 数据

```python
data = request.get_json()
```

含义：

```text
读取 Alertmanager 发送过来的 JSON 告警内容。
```

---

### 4. 打印告警内容

```python
print(json.dumps(data, indent=2, ensure_ascii=False), flush=True)
```

含义：

```text
把告警 JSON 格式化后打印到容器日志。
```

查看日志命令：

```bash
docker compose logs -f alert-webhook
```

---

### 5. 健康检查接口

```python
@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok"}), 200
```

含义：

```text
用于检查 Webhook 服务是否正常。
```

测试命令：

```bash
curl http://localhost:5000/health
```

正常返回：

```json
{"status":"ok"}
```

---

## 十二、编写 Webhook 依赖文件

创建文件：

```bash
nano alert-webhook/requirements.txt
```

写入：

```txt
flask==3.0.3
```

---

## 十三、编写 Webhook Dockerfile

创建文件：

```bash
nano alert-webhook/Dockerfile
```

写入：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

解释：

| 行 | 含义 |
|---|---|
| `FROM python:3.12-slim` | 使用轻量 Python 镜像 |
| `WORKDIR /app` | 设置容器内工作目录 |
| `COPY requirements.txt .` | 复制依赖文件 |
| `RUN pip install ...` | 安装 Flask |
| `COPY app.py .` | 复制程序文件 |
| `CMD ["python", "app.py"]` | 容器启动时运行 Flask 程序 |

---

## 十四、编写 docker-compose.yml

在 `monitor-lab` 根目录创建：

```bash
nano docker-compose.yml
```

写入推荐版本：

```yaml
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
```

注意：这里 Loki 没有写：

```yaml
ports:
  - "3100:3100"
```

原因是当前阶段不需要从宿主机直接访问 Loki，Prometheus 在 Docker Compose 内部网络里可以通过：

```text
loki:3100
```

访问它。

---

## 十五、启动服务

在 `monitor-lab` 根目录执行：

```bash
docker compose up -d --build
```

查看容器状态：

```bash
docker compose ps
```

预期应该看到：

```text
prometheus      Up
alertmanager    Up
node-exporter   Up
alert-webhook   Up
loki            Up
```

查看全部日志：

```bash
docker compose logs -f
```

单独查看 Webhook 日志：

```bash
docker compose logs -f alert-webhook
```

---

## 十六、访问页面

### 1. Prometheus

浏览器访问：

```text
http://localhost:9090
```

Targets 页面：

```text
http://localhost:9090/targets
```

Alerts 页面：

```text
http://localhost:9090/alerts
```

---

### 2. Alertmanager

浏览器访问：

```text
http://localhost:9093
```

---

### 3. Webhook 健康检查

命令行执行：

```bash
curl http://localhost:5000/health
```

正常结果：

```json
{"status":"ok"}
```

---

## 十七、测试告警

### 1. 停止 node-exporter

```bash
docker compose stop node-exporter
```

这一步是在模拟故障：

```text
被监控服务挂了。
```

---

### 2. 查看 Prometheus Alerts

访问：

```text
http://localhost:9090/alerts
```

刚开始会看到：

```text
Pending
```

等待超过 30 秒后，会看到：

```text
Firing
```

由于你写了两个规则，所以 Prometheus 会显示：

```text
FIRING (2)
```

包括：

```text
NodeExporterDown
NodeExporterDownWarning
```

这是正常的。

---

### 3. 查看 Alertmanager

访问：

```text
http://localhost:9093
```

会看到类似：

```text
webhook-demo
instance="node-exporter:9100"
job="node"
1 alert
```

如果只看到 1 个 alert，也是正常的。

原因是：

```text
critical 告警存在时，warning 告警被 inhibit_rules 抑制。
```

---

### 4. 查看 Webhook 日志

执行：

```bash
docker compose logs -f alert-webhook
```

应看到类似：

```text
POST /alert HTTP/1.1" 200 -
```

如果 `app.py` 中设置了 `flush=True`，还会看到更完整的 JSON：

```text
========== ALERT RECEIVED ==========
time: 2026-xx-xxTxx:xx:xx
{
  "receiver": "webhook-demo",
  "status": "firing",
  "alerts": [
    ...
  ]
}
====================================
```

这说明：

```text
Alertmanager 已经成功把告警发送给 Webhook。
```

---

## 十八、恢复告警

重新启动 node-exporter：

```bash
docker compose start node-exporter
```

等待 Prometheus 下一轮抓取。

然后检查：

```text
http://localhost:9090/alerts
```

告警应该消失。

再查看 Webhook 日志：

```bash
docker compose logs -f alert-webhook
```

由于配置了：

```yaml
send_resolved: true
```

理论上会收到恢复通知：

```json
"status": "resolved"
```

---

## 十九、实验现象解释

### 现象 1：Prometheus 显示 2 个 Firing

这是正常的。

原因：

你在规则文件中写了两个告警：

```text
NodeExporterDown          severity=critical
NodeExporterDownWarning   severity=warning
```

它们的表达式都是：

```promql
up{job="node"} == 0
```

停止 node-exporter 后，两个规则都会触发。

Prometheus 的职责只是：

```text
计算规则是否触发。
```

所以它会显示两个 Firing。

---

### 现象 2：Alertmanager 只显示 1 个 alert

这也是正常的。

原因是你配置了：

```yaml
inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal:
      - instance
```

表示：

```text
同一个 instance 上，如果 critical 存在，就抑制 warning。
```

所以最终结果是：

```text
critical 保留
warning 被抑制
```

---

### 现象 3：Webhook 日志中出现 POST /alert 200

这是成功现象。

例如：

```text
172.19.0.3 - - [09/Jun/2026 00:40:12] "POST /alert HTTP/1.1" 200 -
```

含义：

```text
Alertmanager 向 Webhook 发送 POST 请求。
Webhook 成功处理并返回 200。
```

这说明整条链路已经跑通。

---

### 现象 4：Flask WARNING 红色提示

日志中可能看到：

```text
WARNING: This is a development server. Do not use it in a production deployment.
```

这不是错误。

含义：

```text
当前使用的是 Flask 自带开发服务器，不建议用于生产环境。
```

本地学习阶段可以忽略。

生产环境才需要使用：

```text
gunicorn
uwsgi
nginx + gunicorn
```

---

### 现象 5：`Command 'dcoker' not found`

这是输入错误。

错误命令：

```bash
dcoker
```

正确命令：

```bash
docker
```

---

## 二十、Loki 3100 端口问题排查

你曾遇到类似错误：

```text
ports are not available: exposing port TCP 0.0.0.0:3100
```

含义：

```text
Docker 想把 Loki 的容器端口 3100 映射到宿主机 3100，
但是宿主机的 3100 端口不可用。
```

可能原因：

```text
1. 3100 端口已经被其他程序占用。
2. Docker Desktop 端口转发异常。
3. 之前的 Loki 或其他服务没有彻底释放端口。
```

---

### 推荐解决方案：不暴露 Loki 端口

当前阶段主要学习 Alertmanager，不需要从浏览器访问 Loki，所以可以删掉：

```yaml
ports:
  - "3100:3100"
```

改为：

```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    command:
      - "-config.file=/etc/loki/local-config.yaml"
```

然后重新启动：

```bash
docker compose down
docker compose up -d --build
```

---

### 为什么不暴露端口也能用？

Prometheus 配置中写的是：

```yaml
- job_name: loki
  static_configs:
    - targets:
        - loki:3100
```

这里的 `loki:3100` 是容器内部访问地址。

在 Docker Compose 网络中：

```text
prometheus 容器 → loki:3100
```

不需要宿主机访问：

```text
localhost:3100
```

所以不暴露 `3100:3100` 也可以让 Prometheus 抓取 Loki。

---

### 备用方案：换宿主机端口

如果确实想从宿主机访问 Loki，可以改成：

```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3101:3100"
    command:
      - "-config.file=/etc/loki/local-config.yaml"
```

含义：

```text
宿主机 3101 → 容器 3100
```

访问地址变成：

```text
http://localhost:3101
```

容器内部仍然是：

```text
loki:3100
```

---

### 检查端口是否被占用

WSL/Linux 中执行：

```bash
ss -tulnp | grep 3100
```

Windows PowerShell 中执行：

```powershell
netstat -ano | findstr :3100
```

如果有输出，说明 3100 被占用。

---

## 二十一、常见错误与排查

### 1. Prometheus 启动失败

查看日志：

```bash
docker compose logs prometheus
```

检查配置：

```bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
```

检查规则：

```bash
docker compose exec prometheus promtool check rules /etc/prometheus/rules/node_rules.yml
```

常见原因：

```text
YAML 缩进错误
rule_files 路径错误
PromQL 表达式错误
规则文件挂载失败
```

---

### 2. Alertmanager 启动失败

查看日志：

```bash
docker compose logs alertmanager
```

常见原因：

```text
alertmanager.yml 缩进错误
matchers 写法错误
receiver 名称不一致
webhook URL 格式错误
```

---

### 3. Prometheus 有 Firing，但 Webhook 没收到

排查顺序：

```bash
docker compose ps
docker compose logs alertmanager
docker compose logs alert-webhook
```

重点检查：

```yaml
url: http://alert-webhook:5000/alert
```

不要写：

```yaml
url: http://localhost:5000/alert
```

因为在容器内部，`localhost` 指向的是当前容器自己。

---

### 4. Webhook 只显示 POST，但没有 JSON

原因可能是 Python 输出缓冲。

解决：

在 `print()` 中加入：

```python
flush=True
```

然后重新构建：

```bash
docker compose down
docker compose up -d --build
```

---

### 5. 改了配置不生效

改 Prometheus 配置或规则后：

```bash
docker compose restart prometheus
```

改 Alertmanager 配置后：

```bash
docker compose restart alertmanager
```

改 Webhook 代码后：

```bash
docker compose down
docker compose up -d --build
```

---

## 二十二、完整测试流程

```bash
# 1. 启动所有服务
docker compose up -d --build

# 2. 查看容器状态
docker compose ps

# 3. 验证 Webhook
curl http://localhost:5000/health

# 4. 打开 Prometheus Targets
# http://localhost:9090/targets

# 5. 停止 node-exporter，模拟故障
docker compose stop node-exporter

# 6. 打开 Prometheus Alerts
# http://localhost:9090/alerts

# 7. 查看 Alertmanager
# http://localhost:9093

# 8. 查看 Webhook 日志
docker compose logs -f alert-webhook

# 9. 恢复 node-exporter
docker compose start node-exporter

# 10. 再次查看 Webhook 日志，观察 resolved
docker compose logs -f alert-webhook
```

---

## 二十三、学习路径建议

建议按这个顺序练习：

```text
第 1 步：理解 Prometheus Targets 页面
目标：知道哪些服务 UP，哪些服务 DOWN。

第 2 步：停止 node-exporter
目标：理解 up{job="node"} == 0。

第 3 步：观察 Pending → Firing
目标：理解 for: 30s。

第 4 步：观察 Prometheus Alerts
目标：理解 Prometheus 负责规则计算。

第 5 步：观察 Alertmanager 页面
目标：理解 Alertmanager 负责告警处理。

第 6 步：观察 Webhook 日志
目标：理解通知链路。

第 7 步：恢复 node-exporter
目标：理解 resolved 通知。

第 8 步：注释 inhibit_rules 再测试
目标：理解告警抑制前后的区别。
```

---

## 二十四、核心概念总结

### 1. Prometheus 是什么？

Prometheus 是监控系统，主要负责：

```text
采集指标
存储指标
查询指标
计算告警规则
把告警发送给 Alertmanager
```

---

### 2. Alertmanager 是什么？

Alertmanager 是告警管理系统，主要负责：

```text
接收 Prometheus 告警
告警分组
告警去重
告警路由
告警抑制
告警静默
发送通知
```

---

### 3. Webhook 是什么？

Webhook 可以理解为：

```text
一个 HTTP 接收接口。
```

Alertmanager 通过 HTTP POST 把告警发送给 Webhook。

---

### 4. `up` 是什么？

`up` 是 Prometheus 判断 target 是否可抓取的基础指标。

```text
up == 1：抓取成功
up == 0：抓取失败
```

---

### 5. `for` 是什么？

`for` 表示异常持续多长时间后才报警。

作用：

```text
减少误报。
```

---

### 6. `group_by` 是什么？

`group_by` 决定哪些告警合并成一组。

作用：

```text
减少告警风暴。
```

---

### 7. `repeat_interval` 是什么？

如果故障一直存在，多久重复通知一次。

---

### 8. `inhibit_rules` 是什么？

告警抑制规则。

例如：

```text
严重告警已经存在时，不再发送较低级别告警。
```

---

## 二十五、面试可能会问的问题

### 1. Prometheus 和 Alertmanager 的区别是什么？

Prometheus 负责采集指标、存储指标、查询指标和计算告警规则。Alertmanager 负责处理 Prometheus 发来的告警，包括分组、去重、路由、抑制、静默和发送通知。

---

### 2. `up == 0` 表示什么？

表示 Prometheus 抓取某个 target 失败。原因可能是服务停止、网络不通、端口错误、容器未启动或配置错误。

---

### 3. Prometheus 告警中的 Pending 和 Firing 有什么区别？

`Pending` 表示告警条件已经满足，但还没有持续够 `for` 指定的时间。  
`Firing` 表示告警条件持续满足，已经正式触发。

---

### 4. 为什么要配置 `for`？

为了避免短暂网络波动或服务瞬时重启导致误报。

---

### 5. Alertmanager 的 `group_by` 有什么作用？

用于把相同标签的告警合并为一组发送，减少重复通知和告警风暴。

---

### 6. Alertmanager 的 `inhibit_rules` 有什么作用？

用于告警抑制。例如同一台机器已经有 critical 告警时，可以抑制 warning 告警，避免重复通知。

---

### 7. 为什么 Alertmanager 的 Webhook URL 不能写 localhost？

因为 Alertmanager 运行在容器里。对容器来说，`localhost` 指向的是容器自己，不是宿主机，也不是其他容器。应该使用 Docker Compose 服务名，例如：

```text
http://alert-webhook:5000/alert
```

---

### 8. 为什么 Prometheus 显示 2 个 Firing，但 Alertmanager 只显示 1 个 alert？

因为 Prometheus 只负责规则计算，所以两个规则都会显示 Firing。  
Alertmanager 会执行抑制规则，因此 critical 存在时，warning 被抑制，最终只显示或发送 critical。

---

### 9. Webhook 返回 200 说明什么？

说明 Alertmanager 已经成功把告警 POST 到 Webhook，并且 Webhook 正常处理了请求。

---

### 10. 如果告警没有发出来，应该如何排查？

可以按以下顺序排查：

```text
1. Prometheus Targets 是否 UP
2. Prometheus Alerts 是否 Firing
3. Prometheus 是否配置了 alertmanagers
4. Alertmanager 是否启动
5. Alertmanager 配置是否正确
6. receiver 是否匹配
7. webhook URL 是否正确
8. Webhook 服务是否正常
9. 容器网络是否可达
10. 查看 docker compose logs
```

---

## 二十六、本阶段掌握标准

完成本阶段后，应能独立完成：

```text
1. 编写 Prometheus 配置。
2. 编写 Prometheus 告警规则。
3. 编写 Alertmanager 配置。
4. 编写本地 Flask Webhook 接收器。
5. 使用 Docker Compose 启动完整监控告警链路。
6. 停止 node-exporter 模拟故障。
7. 在 Prometheus 中观察 Pending 和 Firing。
8. 在 Alertmanager 中观察告警分组和抑制。
9. 在 Webhook 日志中查看告警通知。
10. 处理 Loki 3100 端口占用问题。
```

---

## 二十七、最终记忆图

```text
Prometheus
  |
  | scrape every 15s
  v
node-exporter
  |
  | up{job="node"} == 0
  v
Alert Rule
  |
  | for: 30s
  v
Firing
  |
  | send alert
  v
Alertmanager
  |
  | group / route / inhibit
  v
Webhook Receiver
  |
  | POST /alert
  v
Flask logs
```

---

## 二十八、一句话总结

这次实验成功的标志是：

```text
停止 node-exporter 后，
Prometheus 出现 Firing，
Alertmanager 收到 critical 告警，
warning 被 inhibit_rules 抑制，
alert-webhook 日志中出现 POST /alert 200。
```

你当前看到的：

```text
Prometheus：FIRING (2)
Alertmanager：1 alert
Webhook：POST /alert 200
```

不是错误，而是完全正常的结果，说明 Alertmanager 的抑制规则已经生效，整条告警链路已经跑通。
