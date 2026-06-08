# Nginx Advanced: Reverse Proxy with Flask

> 第四阶段学习笔记：Nginx 进阶 —— 反向代理、日志、负载均衡与 502 排错

---

## 目录

1. [阶段目标](#1-阶段目标)
2. [本阶段要掌握的核心知识](#2-本阶段要掌握的核心知识)
3. [整体运行流程](#3-整体运行流程)
4. [Nginx 基础配置结构](#4-nginx-基础配置结构)
5. [server 块详解](#5-server-块详解)
6. [location 匹配规则详解](#6-location-匹配规则详解)
7. [root 与 alias 的区别](#7-root-与-alias-的区别)
8. [proxy_pass 与反向代理原理](#8-proxy_pass-与反向代理原理)
9. [实战项目：Nginx 反向代理 Flask](#9-实战项目nginx-反向代理-flask)
10. [查看 access.log 与 error.log](#10-查看-accesslog-与-errorlog)
11. [模拟后端故障并观察 502](#11-模拟后端故障并观察-502)
12. [负载均衡扩展练习](#12-负载均衡扩展练习)
13. [HTTPS 证书学习路线](#13-https-证书学习路线)
14. [常见问题与排错方法](#14-常见问题与排错方法)
15. [常用命令总结](#15-常用命令总结)
16. [面试常见问题与参考答案](#16-面试常见问题与参考答案)
17. [本阶段完成标准](#17-本阶段完成标准)
18. [后续学习建议](#18-后续学习建议)

---

# 1. 阶段目标

在前面的学习中，你已经接触了 Nginx 的静态页面和简单的 `location` 配置。第四阶段的目标是学习 Nginx 在真实运维中的核心用途：**反向代理**。

完成本阶段后，你应该能够独立完成以下任务：

1. 编写基本的 Nginx `server` 配置。
2. 使用多个 `location` 匹配不同路径。
3. 理解 `root` 和 `alias` 的区别。
4. 使用 `proxy_pass` 将请求转发给后端服务。
5. 用 Nginx 代理 Flask / Node.js / Java 等后端服务。
6. 查看并理解 `access.log` 和 `error.log`。
7. 判断 `502 Bad Gateway` 是由 Nginx 引起的，还是后端服务引起的。
8. 使用 `upstream` 配置简单负载均衡。
9. 理解 HTTPS 证书在 Nginx 中的作用。

---

# 2. 本阶段要掌握的核心知识

本阶段重点学习内容如下：

| 知识点 | 说明 |
|---|---|
| `server` 块 | 定义一个虚拟主机，也可以理解为一个站点配置 |
| `location` 匹配 | 根据 URL 路径执行不同处理逻辑 |
| `root` | 用于配置静态资源目录，会将 URL 路径拼接到指定目录后面 |
| `alias` | 用于配置静态资源目录，会用指定目录替换 location 匹配路径 |
| `proxy_pass` | 将请求转发给后端服务，是反向代理的核心指令 |
| 反向代理 | 客户端访问 Nginx，Nginx 再转发给后端真实服务 |
| 负载均衡 | Nginx 将请求分发给多个后端服务实例 |
| HTTPS 证书 | 让 Nginx 支持 HTTPS 访问 |
| `access.log` | 记录客户端访问日志 |
| `error.log` | 记录 Nginx 运行错误和代理错误 |

---

# 3. 整体运行流程

本阶段实战项目的核心流程如下：

```text
浏览器 / curl
    ↓
Nginx : 80
    ↓ proxy_pass
Flask : 8000
```

用户不直接访问 Flask 的 `8000` 端口，而是访问 Nginx 的 `80` 端口。

Nginx 根据用户请求的路径进行判断：

```text
访问 /          → Nginx 自己返回 nginx is running
访问 /health    → Nginx 自己返回 ok
访问 /api/      → Nginx 转发给 Flask
访问 /api/hello → Nginx 转发给 Flask 的 /hello
```

这就是反向代理的基本思想：

```text
客户端并不知道真正的后端服务在哪里。
客户端只访问 Nginx。
Nginx 再把请求转发给后端应用。
```

---

# 4. Nginx 基础配置结构

一个最简单的 Nginx 配置如下：

```nginx
server {
    listen 80;
    server_name _;

    location / {
        return 200 "hello nginx\n";
    }
}
```

含义如下：

```text
server      定义一个站点配置
listen      指定监听端口
server_name 指定匹配的域名
location    指定匹配的 URL 路径
return      直接返回响应内容
```

测试方式：

```bash
curl http://localhost/
```

返回：

```text
hello nginx
```

---

# 5. server 块详解

## 5.1 server 块是什么？

`server` 块可以理解为 Nginx 中的一个“网站配置”。

例如：

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        return 200 "site A\n";
    }
}
```

这个配置表示：

```text
监听 80 端口
处理 example.com 的请求
访问 / 时返回 site A
```

## 5.2 listen 的作用

```nginx
listen 80;
```

表示 Nginx 监听本机的 `80` 端口。

常见端口：

| 端口 | 作用 |
|---|---|
| 80 | HTTP 默认端口 |
| 443 | HTTPS 默认端口 |
| 8000 | 常见后端开发服务端口 |
| 3000 | Node.js / React 常用端口 |
| 5000 | Flask 常用开发端口 |

## 5.3 server_name 的作用

```nginx
server_name _;
```

这里的 `_` 通常表示兜底匹配，也就是没有明确指定域名时也可以处理请求。

生产环境中可能会写成：

```nginx
server_name example.com www.example.com;
```

表示这个 `server` 块专门处理 `example.com` 和 `www.example.com` 的请求。

---

# 6. location 匹配规则详解

`location` 用于根据 URL 路径执行不同处理逻辑。

例如：

```nginx
location / {
    return 200 "home\n";
}

location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}

location = /health {
    return 200 "ok\n";
}
```

含义如下：

| 请求路径 | 匹配结果 | 处理方式 |
|---|---|---|
| `/` | `location /` | 返回 home |
| `/abc` | `location /` | 返回 home |
| `/api/` | `location /api/` | 转发给 Flask |
| `/api/hello` | `location /api/` | 转发给 Flask |
| `/health` | `location = /health` | 返回 ok |

## 6.1 普通前缀匹配

```nginx
location / {
    return 200 "nginx is running\n";
}
```

`location /` 是最宽泛的匹配，几乎所有路径都能匹配到它。

## 6.2 指定前缀匹配

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}
```

这个配置会匹配所有以 `/api/` 开头的路径。

例如：

```text
/api/
/api/hello
/api/user/list
```

## 6.3 精确匹配

```nginx
location = /health {
    return 200 "ok\n";
}
```

`=` 表示精确匹配。

它只匹配：

```text
/health
```

不会匹配：

```text
/health/
/health/check
```

## 6.4 初学者需要记住的匹配优先级

初学阶段先记住：

```text
精确匹配 = /health 优先级最高
/api/ 这种更具体的路径优先于 /
/ 是兜底匹配
```

也就是说：

```nginx
location / {
    ...
}

location /api/ {
    ...
}

location = /health {
    ...
}
```

当访问 `/api/hello` 时，Nginx 会优先使用 `/api/`，而不是 `/`。

---

# 7. root 与 alias 的区别

`root` 和 `alias` 都可以用来配置静态文件目录，但它们的路径处理方式不同。

## 7.1 root：拼接路径

示例：

```nginx
location /static/ {
    root /var/www/html;
}
```

当用户访问：

```text
/static/a.jpg
```

Nginx 实际查找：

```text
/var/www/html/static/a.jpg
```

也就是说：

```text
root = 指定目录 + URL 路径
```

## 7.2 alias：替换路径

示例：

```nginx
location /static/ {
    alias /var/www/assets/;
}
```

当用户访问：

```text
/static/a.jpg
```

Nginx 实际查找：

```text
/var/www/assets/a.jpg
```

也就是说：

```text
alias = 用指定目录替换 location 匹配到的路径
```

## 7.3 记忆方法

```text
root  = 拼接路径
alias = 替换路径
```

## 7.4 常见错误

错误写法：

```nginx
location /static/ {
    alias /var/www/assets;
}
```

建议 `alias` 目录末尾加 `/`：

```nginx
location /static/ {
    alias /var/www/assets/;
}
```

这样路径更清晰，不容易出错。

---

# 8. proxy_pass 与反向代理原理

## 8.1 proxy_pass 是什么？

`proxy_pass` 是 Nginx 反向代理的核心指令。

示例：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}
```

含义是：

```text
用户访问 Nginx 的 /api/
Nginx 把请求转发给 127.0.0.1:8000
```

## 8.2 什么是反向代理？

反向代理的结构如下：

```text
客户端
  ↓
Nginx
  ↓
后端服务 Flask / Java / Node.js / Go
```

客户端只知道 Nginx，不知道真实后端服务。

反向代理的好处：

1. 统一入口。
2. 隐藏后端服务。
3. 可以配置 HTTPS。
4. 可以记录访问日志。
5. 可以做负载均衡。
6. 可以做限流和安全控制。
7. 可以把静态资源和动态接口分开处理。

## 8.3 proxy_pass 后面带 `/` 和不带 `/` 的区别

这是初学者最容易出错的地方。

### 写法一：带 `/`

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}
```

访问：

```text
/api/hello
```

实际转发到：

```text
http://127.0.0.1:8000/hello
```

也就是说，`/api/` 被去掉了。

### 写法二：不带 `/`

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000;
}
```

访问：

```text
/api/hello
```

实际转发到：

```text
http://127.0.0.1:8000/api/hello
```

也就是说，`/api/` 会继续保留。

## 8.4 本阶段推荐写法

本阶段建议使用：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}
```

这样访问：

```text
http://localhost/api/hello
```

会转发到 Flask 的：

```text
http://127.0.0.1:8000/hello
```

---

# 9. 实战项目：Nginx 反向代理 Flask

## 9.1 项目目标

实现以下效果：

```text
Nginx 监听 80
Flask 监听 8000
访问 http://localhost/health 由 Nginx 返回 ok
访问 http://localhost/ 由 Nginx 返回 nginx is running
访问 http://localhost/api/ 转发到 Flask 的 /
访问 http://localhost/api/hello 转发到 Flask 的 /hello
关闭 Flask 后，访问 /api/ 返回 502
查看 access.log 和 error.log
```

---

## 9.2 创建项目目录

```bash
mkdir -p ~/nginx-flask-proxy-lab
cd ~/nginx-flask-proxy-lab
mkdir app
cd app
```

最终目录结构：

```text
~/nginx-flask-proxy-lab/
└── app/
    ├── app.py
    ├── requirements.txt
    └── Dockerfile
```

---

## 9.3 编写 Flask 应用

创建 `app.py`：

```bash
nano app.py
```

写入：

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def index():
    return jsonify({
        "message": "hello from flask",
        "path": "/"
    })

@app.get("/hello")
def hello():
    return jsonify({
        "message": "this is /hello from flask"
    })

@app.get("/health")
def health():
    return jsonify({
        "status": "flask ok"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

保存退出：

```text
Ctrl + O
Enter
Ctrl + X
```

---

## 9.4 编写 requirements.txt

创建文件：

```bash
nano requirements.txt
```

写入：

```text
flask==3.0.3
```

---

## 9.5 编写 Dockerfile

创建 Dockerfile：

```bash
nano Dockerfile
```

写入：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["python", "app.py"]
```

### Dockerfile 每一行含义

```dockerfile
FROM python:3.12-slim
```

表示使用 Python 3.12 的精简版镜像作为基础环境。

```dockerfile
WORKDIR /app
```

表示容器内部的工作目录是 `/app`。

```dockerfile
COPY requirements.txt .
```

将本地的 `requirements.txt` 复制到容器当前目录。

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

在构建镜像时安装 Flask 依赖。

```dockerfile
COPY app.py .
```

将 Flask 应用代码复制到容器中。

```dockerfile
EXPOSE 8000
```

声明容器应用会使用 `8000` 端口。

注意：`EXPOSE` 只是声明，不等于真正映射端口。真正映射端口需要在 `docker run` 时使用 `-p`。

```dockerfile
CMD ["python", "app.py"]
```

容器启动后执行 `python app.py`。

---

## 9.6 构建 Flask 镜像

确认当前目录：

```bash
pwd
```

应该类似：

```text
/home/你的用户名/nginx-flask-proxy-lab/app
```

构建镜像：

```bash
docker build -t flask-api:0.1 .
```

查看镜像：

```bash
docker images
```

你应该能看到：

```text
REPOSITORY   TAG   IMAGE ID   CREATED   SIZE
flask-api    0.1   ...        ...       ...
```

---

## 9.7 启动 Flask 容器

```bash
docker run -d \
  --name flask-api \
  -p 8000:8000 \
  flask-api:0.1
```

参数解释：

```text
-d              后台运行容器
--name          指定容器名称
-p 8000:8000    将宿主机 8000 端口映射到容器 8000 端口
flask-api:0.1   使用的镜像名称和版本
```

查看容器：

```bash
docker ps
```

测试 Flask：

```bash
curl http://127.0.0.1:8000/
```

正常返回：

```json
{
  "message": "hello from flask",
  "path": "/"
}
```

测试 `/hello`：

```bash
curl http://127.0.0.1:8000/hello
```

正常返回：

```json
{
  "message": "this is /hello from flask"
}
```

注意：在配置 Nginx 之前，一定要先确认 Flask 自己能正常访问。

---

## 9.8 安装或检查 Nginx

检查是否安装 Nginx：

```bash
nginx -v
```

如果没有安装：

```bash
sudo apt update
sudo apt install nginx -y
```

启动 Nginx：

```bash
sudo systemctl start nginx
```

设置开机自启：

```bash
sudo systemctl enable nginx
```

查看状态：

```bash
systemctl status nginx
```

测试：

```bash
curl http://localhost/
```

如果返回 Nginx 默认页面，说明 Nginx 正常。

---

## 9.9 配置 Nginx 反向代理

进入配置目录：

```bash
cd /etc/nginx/sites-available
```

备份默认配置：

```bash
sudo cp default default.bak
```

编辑默认配置：

```bash
sudo nano default
```

将文件内容替换为：

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/flask_proxy_access.log;
    error_log  /var/log/nginx/flask_proxy_error.log;

    location / {
        return 200 "nginx is running\n";
        add_header Content-Type text/plain;
    }

    location = /health {
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 配置说明

```nginx
listen 80;
```

让 Nginx 监听 80 端口。

```nginx
server_name _;
```

表示兜底匹配。

```nginx
access_log /var/log/nginx/flask_proxy_access.log;
```

指定访问日志文件。

```nginx
error_log /var/log/nginx/flask_proxy_error.log;
```

指定错误日志文件。

```nginx
location / {
    return 200 "nginx is running\n";
}
```

访问 `/` 时由 Nginx 自己返回内容。

```nginx
location = /health {
    return 200 "ok\n";
}
```

访问 `/health` 时由 Nginx 自己返回健康检查结果。

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}
```

访问 `/api/` 开头的路径时，转发给 Flask。

```nginx
proxy_set_header Host $host;
```

将原始 Host 头传给后端。

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

将客户端真实 IP 传给后端。

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

记录经过的代理链路。

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

告诉后端原始请求使用的是 HTTP 还是 HTTPS。

---

## 9.10 检查并重载 Nginx

检查配置：

```bash
sudo nginx -t
```

如果配置正确，会看到：

```text
syntax is ok
test is successful
```

重新加载 Nginx：

```bash
sudo systemctl reload nginx
```

或者：

```bash
sudo nginx -s reload
```

建议生产环境习惯使用：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

先检查，再重载。

---

## 9.11 测试 Nginx 自身响应

```bash
curl http://localhost/
```

应该返回：

```text
nginx is running
```

测试健康检查：

```bash
curl http://localhost/health
```

应该返回：

```text
ok
```

---

## 9.12 测试反向代理

访问：

```bash
curl http://localhost/api/
```

应该返回 Flask 的内容：

```json
{
  "message": "hello from flask",
  "path": "/"
}
```

访问：

```bash
curl http://localhost/api/hello
```

应该返回：

```json
{
  "message": "this is /hello from flask"
}
```

实际转发关系：

```text
http://localhost/api/hello
    ↓
Nginx 匹配 location /api/
    ↓
proxy_pass http://127.0.0.1:8000/
    ↓
http://127.0.0.1:8000/hello
```

---

# 10. 查看 access.log 与 error.log

## 10.1 access.log 的作用

`access.log` 记录客户端访问情况。

查看日志：

```bash
sudo tail -f /var/log/nginx/flask_proxy_access.log
```

打开另一个终端执行：

```bash
curl http://localhost/
curl http://localhost/health
curl http://localhost/api/
curl http://localhost/api/hello
```

日志中可能看到：

```text
127.0.0.1 - - [08/Jun/2026:09:20:10 -0700] "GET /api/hello HTTP/1.1" 200 ...
```

重点关注：

```text
客户端 IP
访问时间
请求方法
请求路径
HTTP 状态码
User-Agent
```

## 10.2 error.log 的作用

`error.log` 记录 Nginx 错误。

查看错误日志：

```bash
sudo tail -f /var/log/nginx/flask_proxy_error.log
```

常见错误包括：

```text
后端连接失败
配置文件语法错误
权限不足
文件不存在
upstream 超时
```

---

# 11. 模拟后端故障并观察 502

## 11.1 停止 Flask 容器

```bash
docker stop flask-api
```

## 11.2 再次访问 /api/

```bash
curl -i http://localhost/api/
```

你应该看到类似：

```text
HTTP/1.1 502 Bad Gateway
```

说明：

```text
Nginx 正常运行。
但是后端 Flask 不可用。
所以 Nginx 返回 502。
```

## 11.3 查看错误日志

```bash
sudo tail -n 30 /var/log/nginx/flask_proxy_error.log
```

可能看到：

```text
connect() failed (111: Connection refused) while connecting to upstream
```

含义：

```text
Nginx 想连接 127.0.0.1:8000
但是后端服务没有启动
所以连接被拒绝
```

## 11.4 恢复 Flask

```bash
docker start flask-api
```

再次测试：

```bash
curl http://localhost/api/
```

如果恢复正常，说明问题确实来自后端服务停止。

---

# 12. 负载均衡扩展练习

完成基础反向代理之后，可以继续练习 Nginx 负载均衡。

## 12.1 删除原来的 Flask 容器

```bash
docker rm -f flask-api
```

## 12.2 启动两个 Flask 容器

```bash
docker run -d --name flask-api-1 -p 8001:8000 flask-api:0.1

docker run -d --name flask-api-2 -p 8002:8000 flask-api:0.1
```

现在：

```text
宿主机 8001 → flask-api-1 容器 8000
宿主机 8002 → flask-api-2 容器 8000
```

测试：

```bash
curl http://127.0.0.1:8001/
curl http://127.0.0.1:8002/
```

## 12.3 修改 Nginx 配置

编辑：

```bash
sudo nano /etc/nginx/sites-available/default
```

修改为：

```nginx
upstream flask_backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/flask_proxy_access.log;
    error_log  /var/log/nginx/flask_proxy_error.log;

    location / {
        return 200 "nginx is running\n";
        add_header Content-Type text/plain;
    }

    location = /health {
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }

    location /api/ {
        proxy_pass http://flask_backend/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

连续访问：

```bash
curl http://localhost/api/
curl http://localhost/api/
curl http://localhost/api/
```

Nginx 会在两个后端 Flask 容器之间分发请求。

## 12.4 负载均衡的意义

负载均衡的作用：

1. 提高并发处理能力。
2. 避免单个后端服务压力过大。
3. 后端多个实例共同提供服务。
4. 某个实例故障时，可以减少对整体服务的影响。

基础结构：

```text
客户端
  ↓
Nginx
  ↓
后端 1：Flask 8001
后端 2：Flask 8002
```

---

# 13. HTTPS 证书学习路线

本阶段先理解概念即可，不需要立刻配置真实证书。

## 13.1 HTTP 和 HTTPS 的区别

HTTP：

```text
浏览器 → Nginx:80
```

HTTPS：

```text
浏览器 → Nginx:443 → 解密 HTTPS → 转发给后端服务
```

## 13.2 Nginx 中 HTTPS 的基本结构

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/fullchain.pem;
    ssl_certificate_key /path/privkey.pem;

    location /api/ {
        proxy_pass http://127.0.0.1:8000/;
    }
}
```

## 13.3 生产环境常见工具

真实生产环境常用：

```text
域名
DNS 解析
Let's Encrypt 免费证书
Certbot 自动申请证书
Nginx HTTPS 配置
证书自动续期
```

## 13.4 建议学习顺序

```text
第一步：掌握 HTTP 反向代理
第二步：学习域名解析
第三步：学习 HTTPS 证书
第四步：使用 Certbot 自动申请证书
第五步：学习证书自动续期
```

---

# 14. 常见问题与排错方法

## 14.1 访问 /api/ 返回 502

先检查后端 Flask 是否运行：

```bash
docker ps
```

测试 Flask 本身：

```bash
curl http://127.0.0.1:8000/
```

查看 Nginx 错误日志：

```bash
sudo tail -n 50 /var/log/nginx/flask_proxy_error.log
```

常见原因：

```text
Flask 容器停止
Flask 端口写错
proxy_pass 地址写错
Docker 端口没有映射
防火墙阻止连接
Flask 程序启动失败
```

---

## 14.2 Nginx 配置改了但没生效

原因通常是没有重载 Nginx。

执行：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 14.3 80 端口被占用

查看端口占用：

```bash
sudo ss -tulnp | grep :80
```

如果是 Nginx 占用，这是正常的。

如果是其他服务占用，需要停止对应服务。

---

## 14.4 Docker 容器启动失败

查看日志：

```bash
docker logs flask-api
```

删除旧容器后重新运行：

```bash
docker rm -f flask-api

docker run -d \
  --name flask-api \
  -p 8000:8000 \
  flask-api:0.1
```

---

## 14.5 proxy_pass 路径不符合预期

重点检查是否带 `/`。

写法一：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
}
```

转发结果：

```text
/api/hello → /hello
```

写法二：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000;
}
```

转发结果：

```text
/api/hello → /api/hello
```

---

## 14.6 Nginx 启动失败

检查配置语法：

```bash
sudo nginx -t
```

查看状态：

```bash
systemctl status nginx
```

查看错误日志：

```bash
sudo journalctl -xeu nginx
```

---

# 15. 常用命令总结

## 15.1 Nginx 相关命令

```bash
nginx -v
sudo nginx -t
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl status nginx
sudo nginx -s reload
```

## 15.2 日志相关命令

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/flask_proxy_access.log
sudo tail -f /var/log/nginx/flask_proxy_error.log
sudo tail -n 50 /var/log/nginx/flask_proxy_error.log
```

## 15.3 网络排查命令

```bash
curl -i http://localhost/
curl -i http://localhost/health
curl -i http://localhost/api/
curl -i http://localhost/api/hello
sudo ss -tulnp | grep :80
sudo ss -tulnp | grep :8000
```

## 15.4 Docker 相关命令

```bash
docker build -t flask-api:0.1 .
docker images
docker ps
docker ps -a
docker logs flask-api
docker stop flask-api
docker start flask-api
docker rm -f flask-api
docker run -d --name flask-api -p 8000:8000 flask-api:0.1
```

---

# 16. 面试常见问题与参考答案

## 16.1 什么是反向代理？

反向代理是指客户端访问代理服务器，由代理服务器把请求转发给后端真实服务器。客户端不知道后端服务器的真实地址，只知道 Nginx 的地址。

例如：

```text
用户 → Nginx → Flask
```

用户访问的是 Nginx，真正处理业务的是 Flask。

---

## 16.2 Nginx 为什么常用来做反向代理？

因为 Nginx 性能高、配置简单，适合做统一入口。它可以隐藏后端服务、转发请求、配置 HTTPS、记录访问日志、做负载均衡、处理静态文件，还可以进行限流和安全控制。

---

## 16.3 什么是 502 Bad Gateway？

`502 Bad Gateway` 通常表示 Nginx 能收到客户端请求，但是连接后端服务失败。

常见原因包括：

```text
后端服务没有启动
后端端口写错
proxy_pass 地址写错
Docker 端口没有映射
后端服务崩溃
网络不通
```

---

## 16.4 root 和 alias 有什么区别？

`root` 会把 URL 路径拼接到指定目录后面。

`alias` 会用指定目录替换 location 匹配到的路径。

记忆方法：

```text
root = 拼接
alias = 替换
```

---

## 16.5 Nginx reload 和 restart 有什么区别？

`reload` 是重新加载配置，不完全停止 Nginx 服务，对线上影响较小。

`restart` 是重启服务，会先停止再启动，影响更大。

生产环境修改配置后，通常使用：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 16.6 access.log 和 error.log 分别记录什么？

`access.log` 记录客户端访问情况，包括 IP、时间、请求方法、URL、状态码等。

`error.log` 记录 Nginx 的错误信息，包括配置错误、代理后端失败、权限问题、文件不存在等。

---

## 16.7 Nginx 如何实现负载均衡？

通过 `upstream` 定义后端服务器组，然后在 `proxy_pass` 中引用该组。

示例：

```nginx
upstream backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://backend/;
    }
}
```

---

## 16.8 为什么生产环境中不建议让用户直接访问后端端口？

原因包括：

1. 后端服务不应该直接暴露在公网。
2. Nginx 可以统一管理入口。
3. Nginx 可以配置 HTTPS。
4. Nginx 可以做访问日志、安全控制和限流。
5. 后端服务可以灵活扩容，不影响用户访问入口。

---

# 17. 本阶段完成标准

完成本阶段后，你需要能独立做到：

```text
1. 能写出一个完整的 Nginx server 块。
2. 能配置 /、/health、/api/ 三个 location。
3. 能解释 location 匹配的基本规则。
4. 能解释 root 和 alias 的区别。
5. 能使用 proxy_pass 代理 Flask 服务。
6. 能通过 docker run 启动 Flask 容器。
7. 能用 curl 测试 Nginx 和 Flask。
8. 能查看 access.log 和 error.log。
9. 能判断 502 是后端服务故障还是 Nginx 配置问题。
10. 能配置两个 Flask 容器并使用 upstream 做简单负载均衡。
```

---

# 18. 后续学习建议

完成第四阶段后，建议进入第五阶段：

```text
第五阶段：Docker Compose 部署 Nginx + Web + Database
```

下一阶段建议学习：

1. `docker-compose.yaml` 基础语法。
2. Nginx 容器化部署。
3. Flask 容器化部署。
4. PostgreSQL 容器化部署。
5. Redis 容器化部署。
6. 多容器之间通过 Docker 网络通信。
7. 使用 Compose 一条命令启动完整项目。

推荐下一阶段项目：

```text
Nginx + Flask + PostgreSQL + Redis
```

目标架构：

```text
浏览器
  ↓
Nginx 容器 : 80
  ↓
Flask 容器 : 8000
  ↓
PostgreSQL 容器 : 5432
Redis 容器 : 6379
```

---

# 附录：完整命令速查

```bash
# 创建项目
mkdir -p ~/nginx-flask-proxy-lab
cd ~/nginx-flask-proxy-lab
mkdir app
cd app

# 创建文件
nano app.py
nano requirements.txt
nano Dockerfile

# 构建镜像
docker build -t flask-api:0.1 .

# 启动 Flask 容器
docker run -d --name flask-api -p 8000:8000 flask-api:0.1

# 测试 Flask
curl http://127.0.0.1:8000/
curl http://127.0.0.1:8000/hello

# 安装 Nginx
sudo apt update
sudo apt install nginx -y

# 启动 Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
systemctl status nginx

# 修改 Nginx 配置
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
sudo nano /etc/nginx/sites-available/default

# 检查并重载
sudo nginx -t
sudo systemctl reload nginx

# 测试 Nginx
curl http://localhost/
curl http://localhost/health
curl http://localhost/api/
curl http://localhost/api/hello

# 查看日志
sudo tail -f /var/log/nginx/flask_proxy_access.log
sudo tail -f /var/log/nginx/flask_proxy_error.log

# 模拟 502
docker stop flask-api
curl -i http://localhost/api/
sudo tail -n 30 /var/log/nginx/flask_proxy_error.log

# 恢复服务
docker start flask-api
curl http://localhost/api/
```

---

> 建议你把这份笔记上传到 GitHub 仓库中，作为自己的运维学习记录。后续学习 Docker Compose、Ansible、CI/CD 时，可以继续按照这种方式整理学习笔记。
