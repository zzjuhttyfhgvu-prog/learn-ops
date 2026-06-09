# 第七阶段学习笔记：Alertmanager + Loki 日志系统监控进阶

> 学习状态：除邮箱告警外，其余内容已完成。  
> 本笔记重点：原理理解、配置字段含义、完整链路、排错方法、面试常见问题。  
> 推荐项目名称：`Prometheus + Alertmanager + Loki + Grafana Observability Lab`

---

## 一、本阶段学习目标

这一阶段的核心目标不是单纯“看图”，而是建立一套接近真实运维工作的可观测性链路。

你需要理解并掌握：

1. Prometheus 如何根据指标判断服务是否异常。
2. Prometheus 告警规则如何从 `Pending` 进入 `Firing`。
3. Alertmanager 如何接收、分组、抑制、路由和发送告警。
4. Webhook 告警如何接收 Alertmanager 推送的数据。
5. Loki 如何保存日志。
6. Alloy / Promtail 这类日志采集器的作用。
7. Grafana 如何同时查看指标和日志。
8. 如何从“服务异常”定位到“指标异常”和“日志异常”。

完整监控链路如下：

```text
服务 / Exporter
    ↓ 暴露指标
Prometheus
    ↓ 根据规则判断异常
Alertmanager
    ↓ 分组 / 抑制 / 路由 / 通知
Webhook / 邮件 / 企业微信 / 钉钉
```

日志链路如下：

```text
Docker 容器日志 / 应用日志
    ↓
Alloy / Promtail
    ↓
Loki
    ↓
Grafana Explore
```

最终排障链路如下：

```text
告警通知发现问题
    ↓
Prometheus 查看指标变化
    ↓
Grafana 查看趋势图
    ↓
Loki 查询相关日志
    ↓
定位问题原因
    ↓
恢复服务并观察告警恢复
```

---

## 二、Prometheus 告警规则原理

### 1. Prometheus 的核心作用

Prometheus 主要负责三件事：

1. 定期抓取目标服务暴露出来的指标。
2. 将指标保存为时间序列数据。
3. 使用 PromQL 查询和判断指标状态。

例如 node-exporter 暴露主机指标，Prometheus 会定期访问：

```text
node-exporter:9100/metrics
```

如果 Prometheus 能成功抓取目标，就会生成：

```promql
up{job="node"} == 1
```

如果抓取失败，就会变成：

```promql
up{job="node"} == 0
```

因此，`up` 是 Prometheus 中最基础、最常用的健康检查指标。

---

### 2. 告警规则的基本结构

典型规则如下：

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
```

字段解释：

| 字段 | 作用 |
|---|---|
| `groups` | 告警规则组 |
| `name` | 规则组名称 |
| `alert` | 告警名称 |
| `expr` | 触发告警的 PromQL 表达式 |
| `for` | 条件持续多久后才真正触发 |
| `labels` | 告警标签，用于分组、路由、过滤 |
| `annotations` | 告警说明，用于通知内容展示 |

---

### 3. Pending 和 Firing 的区别

当 PromQL 表达式满足条件时，告警不会立刻变成 Firing，而是先进入 Pending 状态。

例如：

```yaml
expr: up{job="node"} == 0
for: 30s
```

含义是：

```text
如果 node-exporter 连续 30 秒不可抓取，才真正触发告警。
```

状态变化过程：

```text
服务正常
    ↓
up == 1
    ↓ 停止 node-exporter
up == 0
    ↓ 表达式满足，进入 Pending
持续 30 秒
    ↓
进入 Firing
    ↓
发送给 Alertmanager
```

区别如下：

| 状态 | 含义 |
|---|---|
| Inactive | 告警条件不成立 |
| Pending | 条件已成立，但还没持续到 `for` 时间 |
| Firing | 条件持续满足，正式触发告警 |

`for` 的作用是减少误报。例如服务短暂重启、网络瞬时抖动，不一定需要报警。

---

## 三、Alertmanager 原理

### 1. Alertmanager 的定位

Prometheus 负责判断“是否异常”，Alertmanager 负责处理“异常之后怎么办”。

Prometheus 不直接负责复杂通知逻辑，而是把告警发送给 Alertmanager。Alertmanager 再根据配置决定：

1. 告警应该发给谁。
2. 是否需要分组。
3. 是否需要抑制。
4. 是否需要静默。
5. 多久重复通知一次。
6. 恢复后是否发送 resolved 通知。

完整关系：

```text
Prometheus Rules
    ↓
Alert Firing
    ↓
Alertmanager
    ↓
Routing / Grouping / Inhibition / Silence
    ↓
Receiver
```

---

### 2. Alertmanager 配置结构

典型配置：

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

Alertmanager 配置可以分为四大块：

| 配置块 | 作用 |
|---|---|
| `global` | 全局配置，例如超时时间、SMTP 信息 |
| `route` | 告警路由规则 |
| `receivers` | 告警接收器，例如 Webhook、Email |
| `inhibit_rules` | 告警抑制规则 |

---

### 3. 告警路由 route

`route` 决定告警发给哪个 receiver。

```yaml
route:
  receiver: webhook-demo
```

表示所有没有特殊匹配的告警都发给 `webhook-demo`。

如果有多个团队，可以根据 label 路由：

```yaml
routes:
  - matchers:
      - team="ops"
    receiver: ops-webhook

  - matchers:
      - team="dev"
    receiver: dev-webhook
```

含义：

```text
team=ops 的告警发给运维团队
team=dev 的告警发给开发团队
```

因此，Prometheus 告警规则中的 `labels` 不只是说明信息，它们还会影响 Alertmanager 的路由逻辑。

---

### 4. 告警分组 group_by

配置：

```yaml
group_by:
  - job
  - instance
```

含义：

```text
相同 job 和 instance 的告警合并到一组发送。
```

例如同一台服务器上同时出现：

```text
NodeExporterDown
NodeCPUHigh
NodeMemoryHigh
NodeDiskHigh
```

如果不分组，可能会连续发送多条通知。分组后可以合并成一组通知，减少告警轰炸。

常见分组字段：

```yaml
group_by:
  - alertname
  - job
  - instance
  - severity
```

分组的核心目的：

```text
减少重复通知，让告警更聚合、更可读。
```

---

### 5. group_wait、group_interval、repeat_interval

这三个字段非常容易在面试中被问到。

```yaml
group_wait: 10s
group_interval: 30s
repeat_interval: 3m
```

含义如下：

| 字段 | 含义 |
|---|---|
| `group_wait` | 第一个告警出现后等待多久再发送，目的是等待同组告警一起到达 |
| `group_interval` | 同一组中出现新告警后，距离上次发送至少等待多久 |
| `repeat_interval` | 如果告警一直未恢复，多久重复发送一次 |

举例：

```text
第 0 秒：NodeExporterDown 出现
第 10 秒：Alertmanager 发送第一条通知
第 20 秒：同组又出现 NodeCPUHigh
第 40 秒：超过 group_interval，再次发送更新通知
如果故障一直存在，每 3 分钟重复提醒一次
```

---

### 6. 告警抑制 inhibition

告警抑制用于避免低级别告警重复打扰。

配置示例：

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
如果同一个 instance 上已经存在 critical 告警，
那么 warning 告警就不再单独通知。
```

真实场景：

```text
服务器宕机
    ↓
critical 告警触发
    ↓
CPU、内存、磁盘、端口等 warning 也可能异常
    ↓
这些 warning 可以被抑制，避免告警风暴
```

告警抑制的三个关键概念：

| 字段 | 作用 |
|---|---|
| `source_matchers` | 作为抑制源的告警 |
| `target_matchers` | 被抑制的告警 |
| `equal` | 要求哪些标签相同才抑制 |

---

### 7. 告警静默 silence

Silence 是临时关闭某些告警通知。

常见场景：

```text
计划维护
服务升级
机器重启
故障已知但暂时不希望重复提醒
```

例如你正在维护 node-exporter，可以在 Alertmanager 页面创建 Silence，匹配：

```text
alertname = NodeExporterDown
```

这样告警仍然会存在，但不会通知出去。

注意：

```text
Silence 不是解决问题，只是临时不通知。
```

---

## 四、Webhook 告警原理

### 1. Webhook 的作用

Webhook 本质上就是一个 HTTP 回调地址。

Alertmanager 触发告警后，会向配置中的 URL 发送 POST 请求。

例如：

```yaml
webhook_configs:
  - url: http://alert-webhook:5000/alert
    send_resolved: true
```

含义：

```text
Alertmanager 把告警 JSON POST 到 alert-webhook 容器的 /alert 接口。
```

---

### 2. Webhook 接收到的数据

Alertmanager 通常会发送类似结构：

```json
{
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "NodeExporterDown",
        "severity": "critical",
        "team": "ops"
      },
      "annotations": {
        "summary": "node-exporter is down",
        "description": "Prometheus cannot scrape node-exporter."
      }
    }
  ]
}
```

关键字段：

| 字段 | 含义 |
|---|---|
| `status` | firing 或 resolved |
| `alerts` | 告警列表 |
| `labels` | 告警标签 |
| `annotations` | 告警说明 |
| `startsAt` | 告警开始时间 |
| `endsAt` | 告警结束时间 |

---

### 3. Webhook 和企业微信 / 钉钉的关系

企业微信、钉钉、飞书机器人本质上也是 Webhook。

但是它们通常要求固定格式的数据。例如钉钉机器人可能要求：

```json
{
  "msgtype": "text",
  "text": {
    "content": "告警内容"
  }
}
```

而 Alertmanager 发送的是自己的 JSON 格式。

因此生产环境中常见做法是：

```text
Alertmanager
    ↓
自定义 Webhook Adapter
    ↓
格式转换
    ↓
钉钉 / 企业微信 / 飞书机器人
```

你写的 Flask Webhook 就是一个最小版本的告警适配器。

---

## 五、Loki 日志系统原理

### 1. 为什么需要日志系统

Prometheus 适合回答：

```text
服务是否正常？
CPU 是否过高？
内存是否异常？
接口延迟是否变大？
某个端口是否可访问？
```

但它不适合回答：

```text
服务为什么报错？
请求返回 500 的原因是什么？
容器启动失败时输出了什么？
某个用户请求发生了什么？
```

这些问题需要日志系统。

因此：

```text
Prometheus 负责指标
Loki 负责日志
Grafana 负责统一展示
```

---

### 2. Loki 的核心设计思路

Loki 和 ELK 最大的区别是：

```text
Loki 不会像 Elasticsearch 那样对日志全文建立大量索引，
它主要索引 label，日志正文按时间存储。
```

因此 Loki 更轻量，适合中小规模和云原生日志场景。

Loki 的查询依赖 label，例如：

```logql
{container=~".*prometheus.*"}
```

这表示查询 container 标签匹配 prometheus 的日志流。

---

### 3. Loki 的基本组件

| 组件 | 作用 |
|---|---|
| Loki | 存储和查询日志 |
| Alloy / Promtail | 采集日志并推送到 Loki |
| Grafana | 查询和展示日志 |
| LogQL | Loki 的查询语言 |

日志流向：

```text
容器 stdout / stderr
    ↓
Alloy 读取 Docker 日志
    ↓
添加 label
    ↓
推送到 Loki
    ↓
Grafana 使用 LogQL 查询
```

---

## 六、Alloy / Promtail 原理

### 1. Promtail 的历史作用

Promtail 是 Loki 早期常用的日志采集器。

它的作用是：

```text
发现日志文件或容器日志
    ↓
读取日志
    ↓
给日志加标签
    ↓
发送到 Loki
```

经典路线：

```text
应用日志 / Docker 日志
    ↓
Promtail
    ↓
Loki
    ↓
Grafana
```

---

### 2. 为什么现在更推荐 Alloy

Promtail 已经进入 EOL 阶段，新的 Grafana 生态更推荐使用 Grafana Alloy。

Alloy 的作用更综合，它不仅可以处理日志，还可以处理指标、traces 等可观测性数据。

新的路线：

```text
应用日志 / Docker 日志
    ↓
Grafana Alloy
    ↓
Loki
    ↓
Grafana
```

面试中可以这样回答：

```text
早期 Loki 常用 Promtail 采集日志，但 Promtail 已进入 EOL，
新项目更推荐使用 Grafana Alloy 作为日志采集 Agent。
我在实验中理解了 Promtail 的作用，但实际配置使用 Alloy 采集 Docker 容器日志并发送到 Loki。
```

---

### 3. Alloy 配置原理

示例：

```hcl
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}

discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

discovery.relabel "docker" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }
}

loki.source.docker "containers" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.docker.output
  forward_to = [loki.write.local.receiver]
}
```

这段配置可以拆成四步理解：

```text
1. loki.write：定义日志写到哪里，也就是 Loki 地址
2. discovery.docker：通过 Docker socket 发现容器
3. discovery.relabel：把容器元数据转换成 Loki label
4. loki.source.docker：读取容器日志并转发给 Loki
```

---

## 七、Grafana 中指标和日志的联动

Grafana 在这一阶段的作用是统一展示。

它连接两个数据源：

```text
Prometheus：查询指标
Loki：查询日志
```

在 Grafana Explore 中可以：

1. 用 PromQL 查询指标。
2. 用 LogQL 查询日志。
3. 根据告警时间点查看对应时间范围内的日志。

典型排障流程：

```text
收到 NodeExporterDown 告警
    ↓
Grafana Explore 选择 Prometheus
    ↓
查询 up{job="node"}
    ↓
确认 node-exporter 变成 0
    ↓
切换 Loki
    ↓
查询 {container=~".*node-exporter.*"}
    ↓
查看容器是否停止、报错或重启
```

---

## 八、PromQL 与 LogQL 基础

### 1. PromQL 常用查询

查询所有目标状态：

```promql
up
```

查询 node-exporter 状态：

```promql
up{job="node"}
```

查询抓取失败的目标：

```promql
up == 0
```

查询 Prometheus 自己是否正常：

```promql
up{job="prometheus"}
```

---

### 2. LogQL 常用查询

查询所有容器日志：

```logql
{container=~".*"}
```

查询 Prometheus 容器日志：

```logql
{container=~".*prometheus.*"}
```

查询 Alertmanager 容器日志：

```logql
{container=~".*alertmanager.*"}
```

查询包含 error 的日志：

```logql
{container=~".*"} |= "error"
```

查询包含 failed 的日志：

```logql
{container=~".*"} |= "failed"
```

LogQL 基本结构：

```text
{label selector} 日志过滤条件
```

例如：

```logql
{container=~".*prometheus.*"} |= "error"
```

含义：

```text
先找到 container 标签包含 prometheus 的日志流，
再过滤出日志正文中包含 error 的行。
```

---

## 九、完整故障演练原理

### 1. 停止 node-exporter

命令：

```bash
docker compose stop node-exporter
```

此时 node-exporter 容器停止，Prometheus 无法访问：

```text
node-exporter:9100/metrics
```

---

### 2. Prometheus 发现 up == 0

Prometheus 下一次 scrape 时会发现目标不可达。

于是指标变成：

```promql
up{job="node"} 0
```

告警规则满足：

```promql
up{job="node"} == 0
```

---

### 3. 告警进入 Pending

因为配置了：

```yaml
for: 30s
```

所以 Prometheus 不会立刻触发，而是先进入 Pending。

---

### 4. 告警进入 Firing

如果 30 秒后 node-exporter 仍然不可达，告警进入 Firing。

Prometheus 将告警发送给 Alertmanager。

---

### 5. Alertmanager 处理告警

Alertmanager 执行：

```text
分组
    ↓
判断是否被抑制
    ↓
判断是否被静默
    ↓
选择 receiver
    ↓
发送 Webhook
```

---

### 6. Webhook 收到告警

Webhook 服务收到 POST 请求，打印 JSON。

这说明告警链路已经跑通。

---

### 7. Grafana 查看指标和日志

在 Grafana 中：

```text
Prometheus 数据源：确认 up 从 1 变成 0
Loki 数据源：查看 node-exporter / alertmanager / webhook 日志
```

---

### 8. 恢复 node-exporter

命令：

```bash
docker compose start node-exporter
```

Prometheus 重新抓取成功：

```promql
up{job="node"} 1
```

Alertmanager 收到 resolved 状态，并根据 `send_resolved: true` 发送恢复通知。

---

## 十、常见问题与排错

### 1. Prometheus 看不到 node-exporter

检查容器：

```bash
docker compose ps
```

检查日志：

```bash
docker compose logs node-exporter
```

检查 Prometheus 配置中 target 是否写成：

```yaml
- node-exporter:9100
```

不要写成：

```yaml
- localhost:9100
```

原因：

```text
在 Docker Compose 网络中，Prometheus 容器里的 localhost 指的是 Prometheus 自己，
不是 node-exporter 容器。
```

---

### 2. 告警规则没有加载

检查 Prometheus 日志：

```bash
docker compose logs prometheus
```

常见原因：

1. YAML 缩进错误。
2. `rule_files` 路径错误。
3. 规则文件挂载路径错误。
4. PromQL 表达式写错。

检查配置：

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml
```

检查 Compose 挂载：

```yaml
volumes:
  - ./prometheus/rules:/etc/prometheus/rules:ro
```

---

### 3. Alertmanager 没收到告警

检查 Prometheus 配置：

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

检查 Alertmanager 是否启动：

```bash
docker compose ps alertmanager
```

检查日志：

```bash
docker compose logs alertmanager
```

---

### 4. Webhook 没收到告警

检查 Alertmanager receiver：

```yaml
receivers:
  - name: webhook-demo
    webhook_configs:
      - url: http://alert-webhook:5000/alert
```

不要写成：

```yaml
url: http://localhost:5000/alert
```

原因：

```text
Alertmanager 容器里的 localhost 是 Alertmanager 自己，
不是 alert-webhook 容器。
```

---

### 5. Grafana 看不到 Loki 日志

检查 Loki 是否 ready：

```bash
curl http://localhost:3100/ready
```

检查 Alloy 日志：

```bash
docker compose logs alloy
```

检查 Docker socket 是否挂载：

```yaml
- /var/run/docker.sock:/var/run/docker.sock:ro
```

检查 Alloy 是否配置 Loki 地址：

```hcl
url = "http://loki:3100/loki/api/v1/push"
```

---

## 十一、面试重点问题与参考答案

### 1. Prometheus、Alertmanager、Grafana 分别负责什么？

Prometheus 负责采集和存储指标，并通过 PromQL 进行查询和告警规则判断。Alertmanager 负责接收 Prometheus 发送的告警，并进行分组、抑制、静默、路由和通知。Grafana 负责可视化展示，可以连接 Prometheus 查看指标，也可以连接 Loki 查看日志。

---

### 2. Prometheus 和 Alertmanager 的关系是什么？

Prometheus 负责判断告警是否触发，Alertmanager 负责处理告警触发之后的通知逻辑。Prometheus 根据规则生成告警，当告警进入 Firing 状态后发送给 Alertmanager。Alertmanager 再根据配置决定是否分组、是否抑制、发给哪个接收器。

---

### 3. `up == 0` 表示什么？

`up` 是 Prometheus 自动生成的抓取状态指标。`up == 1` 表示 Prometheus 成功抓取目标，`up == 0` 表示抓取失败。常见原因包括服务停止、端口不通、网络异常、配置 target 错误。

---

### 4. Prometheus 告警中的 `for` 有什么用？

`for` 表示告警条件必须持续满足一段时间后才真正触发。它的作用是减少误报，避免服务短暂抖动或瞬时重启就触发告警。例如 `for: 30s` 表示条件持续 30 秒后告警才会从 Pending 变成 Firing。

---

### 5. Pending 和 Firing 有什么区别？

Pending 表示告警条件已经满足，但还没有持续到 `for` 指定的时间。Firing 表示告警条件持续满足，已经正式触发，并会发送给 Alertmanager。

---

### 6. Alertmanager 的 group_by 有什么作用？

`group_by` 用于对告警进行分组。相同标签的告警会合并到一组通知中，避免一次故障触发大量重复告警。例如同一台机器同时出现 CPU、内存、磁盘异常，可以按 `instance` 合并成一组。

---

### 7. group_wait、group_interval、repeat_interval 分别是什么？

`group_wait` 是第一个告警出现后等待多久再发送，用于等待同组告警一起到达。`group_interval` 是同一告警组有新告警后，距离上次通知至少等待多久再发。`repeat_interval` 是告警一直未恢复时，重复通知的间隔。

---

### 8. Alertmanager 的告警抑制是什么？

告警抑制是指当某个高级别告警存在时，自动抑制相关低级别告警。例如同一台机器已经有 `critical` 级别的宕机告警，那么这台机器上的 `warning` 级别 CPU 或内存告警可以不再单独通知，避免告警风暴。

---

### 9. Alertmanager 的 Silence 是什么？

Silence 是临时静默某些告警通知。它不会让告警消失，只是不再发送通知。常用于计划维护、服务升级、已知故障处理期间。

---

### 10. Webhook 告警的原理是什么？

Webhook 本质是 HTTP 回调。Alertmanager 触发告警后，会把告警内容以 JSON 形式通过 POST 请求发送到配置好的 URL。自定义服务可以接收这个 JSON，然后打印、保存或转换成企业微信、钉钉、飞书等机器人需要的格式。

---

### 11. 为什么 Alertmanager 中不能把 Webhook 地址写成 localhost？

因为在 Docker Compose 网络中，每个容器都有自己的网络命名空间。Alertmanager 容器里的 `localhost` 指的是 Alertmanager 自己，而不是宿主机，也不是其他容器。要访问其他容器，应使用 Compose 服务名，例如：

```text
http://alert-webhook:5000/alert
```

---

### 12. Prometheus 和 Loki 的区别是什么？

Prometheus 主要存储数值型时间序列指标，例如 CPU、内存、请求数、服务状态。Loki 主要存储文本日志，例如应用日志、容器日志、错误日志。Prometheus 使用 PromQL 查询，Loki 使用 LogQL 查询。

---

### 13. Grafana 在监控系统中负责什么？

Grafana 主要负责可视化和统一查询。它本身不是主要的数据采集器或存储系统，而是连接 Prometheus、Loki 等数据源，将指标和日志展示出来，帮助运维人员分析问题。

---

### 14. Loki 和 ELK 有什么区别？

Loki 更轻量，主要索引 label，不会像 Elasticsearch 那样对日志全文建立大量索引，因此资源占用更低。ELK 的全文检索能力更强，但架构更复杂、资源消耗更大。对于学习阶段和中小规模容器日志场景，Loki 更容易上手。

---

### 15. LogQL 是什么？

LogQL 是 Loki 的查询语言，语法风格类似 PromQL，但用于查询日志。它通常先通过 label 选择日志流，再对日志内容进行过滤。例如：

```logql
{container=~".*prometheus.*"} |= "error"
```

表示查询 Prometheus 容器中包含 `error` 的日志。

---

### 16. Promtail 是什么？为什么现在更推荐 Alloy？

Promtail 是 Loki 早期常用的日志采集器，负责读取日志、添加标签并发送给 Loki。但 Promtail 已经进入 EOL 阶段，新项目更推荐使用 Grafana Alloy。Alloy 能采集日志，也能处理指标、traces 等更多可观测性数据。

---

### 17. 如果收到 NodeExporterDown 告警，你会怎么排查？

可以按以下步骤排查：

1. 在 Prometheus 查询 `up{job="node"}`，确认是否为 0。
2. 查看 Prometheus Targets 页面，确认 target 是否 down。
3. 使用 `docker compose ps` 查看 node-exporter 容器是否运行。
4. 使用 `docker compose logs node-exporter` 查看容器日志。
5. 在 Grafana Loki 中查询 node-exporter 相关日志。
6. 检查 Prometheus 配置中的 target 是否正确。
7. 恢复服务后观察 `up` 是否变回 1，并确认 Alertmanager 是否发送 resolved。

---

### 18. 如何避免告警风暴？

可以从以下几个方面处理：

1. 合理设置 `for`，避免瞬时波动触发告警。
2. 使用 `group_by` 合并相似告警。
3. 使用 `inhibit_rules` 抑制低级别重复告警。
4. 使用 `repeat_interval` 控制重复通知频率。
5. 维护期间使用 Silence。
6. 为告警设置合理的 severity 和 team 标签。

---

### 19. 告警恢复通知是怎么实现的？

在 Alertmanager 的 receiver 中设置：

```yaml
send_resolved: true
```

当 Prometheus 中告警条件不再满足时，告警状态会变成 resolved，Alertmanager 会向接收器发送恢复通知。

---

### 20. 这个项目可以怎么写到简历上？

可以写成：

```text
基于 Docker Compose 搭建 Prometheus + Alertmanager + Grafana + Loki + Alloy 可观测性实验环境，实现 node-exporter 监控、Prometheus 告警规则、Alertmanager 告警分组与抑制、Webhook 告警通知、Loki 容器日志采集，并通过 Grafana 完成指标与日志联动排障。
```

如果写成项目经历，可以这样展开：

```text
项目名称：云原生监控告警与日志分析平台实验

项目内容：
- 使用 Docker Compose 部署 Prometheus、Alertmanager、Grafana、Loki、Alloy、node-exporter。
- 编写 Prometheus 告警规则，基于 up 指标检测 node-exporter 异常。
- 配置 Alertmanager 路由、分组、抑制和 Webhook 通知。
- 使用 Flask 编写 Webhook 接收服务，验证 firing 和 resolved 告警流程。
- 使用 Alloy 采集 Docker 容器日志并推送到 Loki。
- 在 Grafana 中接入 Prometheus 和 Loki，实现指标与日志统一查询。
- 模拟 node-exporter 故障，完成从告警触发到日志排查再到服务恢复的完整流程。

技术栈：Docker Compose、Prometheus、Alertmanager、Grafana、Loki、Grafana Alloy、Flask、PromQL、LogQL。
```

---

## 十二、本阶段总结

这一阶段完成后，你已经不只是会看 Grafana 图表，而是掌握了监控系统中非常关键的一条主线：

```text
指标采集 → 规则判断 → 告警触发 → 告警处理 → 通知发送 → 日志排查 → 服务恢复
```

你应该重点记住：

1. Prometheus 负责指标。
2. Alertmanager 负责告警处理。
3. Loki 负责日志。
4. Alloy / Promtail 负责日志采集。
5. Grafana 负责统一展示。
6. `up == 0` 是最基础的服务不可达告警。
7. `for` 用于避免瞬时误报。
8. `group_by` 用于告警分组。
9. `inhibit_rules` 用于告警抑制。
10. Webhook 是告警系统和外部通知系统之间的桥梁。

对运维岗位来说，这一阶段非常重要，因为真实工作中的监控系统不是只看图，而是要做到：

```text
提前发现问题，及时通知人员，快速定位原因，尽快恢复服务。
```

