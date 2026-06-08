# 第 5 周：Dockerfile + Docker Compose 综合项目学习笔记

> 项目主题：使用 Dockerfile 构建自己的 Flask Web 镜像，并用 Docker Compose 编排 `Nginx + Flask + PostgreSQL + Redis` 多容器项目。  
> 适合对象：刚开始学习 Docker、Nginx、后端服务部署、运维基础的初学者。  
> 本笔记整理自完整实操与排错过程，重点包括：项目搭建、代码配置、启动测试、Nginx 反向代理、Redis/PostgreSQL 连接、404 排查、80 端口冲突排查。

---

## 一、本阶段学习目标

第 5 周的目标不是简单运行别人写好的镜像，而是要完成一个接近真实运维场景的小型多服务项目。

你需要掌握：

1. 使用 `Dockerfile` 构建自己的 Flask Web 镜像。
2. 使用 `Docker Compose` 同时启动多个服务。
3. 使用 Nginx 作为统一入口。
4. 使用 Nginx 将 `/api/` 请求反向代理到 Flask。
5. Flask 连接 Redis，并实现访问计数。
6. Flask 连接 PostgreSQL，并查询数据库版本。
7. 学会查看容器状态、日志和容器内部文件。
8. 学会排查 Nginx 404、502、端口冲突等常见问题。

最终项目架构如下：

```text
浏览器 / curl
    ↓
Docker Nginx:80
    ↓ proxy_pass
Flask Web:8000
    ↓              ↓
Redis:6379     PostgreSQL:5432
```

---

## 二、本项目最终访问效果

项目启动成功后，应能访问以下地址：

| 地址 | 作用 | 预期结果 |
|---|---|---|
| `http://localhost/` | Nginx 首页 | 返回 `nginx is running` |
| `http://localhost/api/health` | Flask 健康检查 | 返回 Flask 正常状态 |
| `http://localhost/api/redis` | 测试 Flask 连接 Redis | 返回 Redis 访问计数 |
| `http://localhost/api/db` | 测试 Flask 连接 PostgreSQL | 返回 PostgreSQL 版本 |

---

## 三、核心知识点总览

### 1. Dockerfile 是什么

`Dockerfile` 是用来描述“如何构建镜像”的文件。

流程如下：

```text
应用代码 + Dockerfile
        ↓ docker build
Docker 镜像 image
        ↓ docker run / docker compose up
Docker 容器 container
```

常见指令：

| 指令 | 作用 |
|---|---|
| `FROM` | 指定基础镜像 |
| `WORKDIR` | 设置容器中的工作目录 |
| `COPY` | 复制文件到镜像中 |
| `RUN` | 构建镜像时执行命令 |
| `CMD` | 容器启动时执行命令 |
| `EXPOSE` | 声明容器内部监听端口 |
| `ENV` | 设置环境变量 |

---

### 2. Docker Compose 是什么

Docker Compose 用于管理多个容器。它通过一个 `compose.yaml` 文件描述多个服务。

本项目中有 4 个服务：

| 服务名 | 镜像/构建方式 | 作用 |
|---|---|---|
| `nginx` | `nginx:1.27` | 对外入口，反向代理 |
| `web` | 自己写 Dockerfile 构建 | Flask Web 应用 |
| `db` | `postgres:16` | PostgreSQL 数据库 |
| `redis` | `redis:7` | Redis 缓存/计数器 |

Compose 里最重要的概念是：**服务名就是容器内部网络里的 DNS 名称**。

例如：

```text
Nginx 容器访问 Flask： http://web:8000
Flask 容器访问 PostgreSQL： host=db
Flask 容器访问 Redis： host=redis
```

---

## 四、项目目录结构

在 WSL Ubuntu 中创建项目：

```bash
mkdir -p ~/compose-project
cd ~/compose-project
mkdir -p app nginx
```

最终目录结构：

```text
compose-project/
├── compose.yaml
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app.py
│   └── .dockerignore
└── nginx/
    └── default.conf
```

---

## 五、编写 Flask 应用

进入 `app` 目录：

```bash
cd ~/compose-project/app
nano app.py
```

写入如下内容：

```python
import os
from flask import Flask, jsonify
import redis
import psycopg2

app = Flask(__name__)

REDIS_HOST = os.getenv("REDIS_HOST", "redis")
DB_HOST = os.getenv("DB_HOST", "db")
DB_NAME = os.getenv("POSTGRES_DB", "blogdb")
DB_USER = os.getenv("POSTGRES_USER", "bloguser")
DB_PASSWORD = os.getenv("POSTGRES_PASSWORD", "blogpass")


@app.route("/")
def index():
    return jsonify({
        "message": "Hello from Flask",
        "status": "ok"
    })


@app.route("/api/health")
def health():
    return jsonify({
        "message": "Flask app is healthy",
        "status": "ok"
    })


@app.route("/api/redis")
def redis_check():
    r = redis.Redis(host=REDIS_HOST, port=6379, decode_responses=True)
    r.incr("visit_count")
    count = r.get("visit_count")

    return jsonify({
        "redis_host": REDIS_HOST,
        "status": "ok",
        "visit_count": count
    })


@app.route("/api/db")
def db_check():
    conn = psycopg2.connect(
        host=DB_HOST,
        database=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD
    )

    cur = conn.cursor()
    cur.execute("SELECT version();")
    version = cur.fetchone()[0]

    cur.close()
    conn.close()

    return jsonify({
        "db_host": DB_HOST,
        "postgres_version": version,
        "status": "ok"
    })


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

保存：

```text
Ctrl + O
Enter
Ctrl + X
```

### Flask 代码解释

#### 1. 为什么使用环境变量

```python
REDIS_HOST = os.getenv("REDIS_HOST", "redis")
DB_HOST = os.getenv("DB_HOST", "db")
```

这样做的好处是：

1. 代码不写死具体地址。
2. 容器环境可以通过 Compose 传入配置。
3. 更接近真实生产环境。

#### 2. 为什么 Redis 主机名是 `redis`

因为 Compose 文件中服务名叫：

```yaml
redis:
  image: redis:7
```

所以 Flask 容器可以直接通过 `redis` 访问 Redis 容器。

#### 3. 为什么 PostgreSQL 主机名是 `db`

因为 Compose 文件中服务名叫：

```yaml
db:
  image: postgres:16
```

所以 Flask 容器可以通过 `db` 访问 PostgreSQL 容器。

#### 4. 为什么不能写 `localhost`

在容器中，`localhost` 表示“当前容器自己”。

如果 Flask 容器里写：

```python
host="localhost"
```

含义是：

```text
Flask 容器自己
```

不是 PostgreSQL 容器，也不是 Redis 容器。

所以在 Docker Compose 项目中，容器之间通信应该使用服务名：

```text
web → db
web → redis
nginx → web
```

---

## 六、编写 Python 依赖文件

在 `app` 目录下创建：

```bash
nano requirements.txt
```

写入：

```txt
flask==3.0.3
redis==5.0.8
psycopg2-binary==2.9.9
gunicorn==22.0.0
```

依赖说明：

| 依赖 | 作用 |
|---|---|
| `flask` | Python Web 框架 |
| `redis` | Python 连接 Redis |
| `psycopg2-binary` | Python 连接 PostgreSQL |
| `gunicorn` | 生产环境常用 Python Web Server |

---

## 七、编写 Dockerfile

在 `app` 目录下创建：

```bash
nano Dockerfile
```

写入：

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

### Dockerfile 逐行解释

#### 1. 基础镜像

```dockerfile
FROM python:3.12-slim
```

表示使用 Python 3.12 精简版镜像作为基础环境。

#### 2. Python 环境变量

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
```

含义：

| 变量 | 作用 |
|---|---|
| `PYTHONDONTWRITEBYTECODE=1` | 不生成 `.pyc` 缓存文件 |
| `PYTHONUNBUFFERED=1` | 日志实时输出，方便 `docker logs` 查看 |

#### 3. 工作目录

```dockerfile
WORKDIR /app
```

容器内部的工作目录设置为 `/app`。

#### 4. 先复制依赖文件

```dockerfile
COPY requirements.txt .
```

先复制依赖文件，是为了利用 Docker 构建缓存。只要 `requirements.txt` 不变，依赖安装层可以复用。

#### 5. 安装依赖

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

在构建镜像时安装 Python 依赖。

#### 6. 复制应用代码

```dockerfile
COPY app.py .
```

把 Flask 源码复制进镜像。

#### 7. 声明端口

```dockerfile
EXPOSE 8000
```

声明容器内部服务监听 8000 端口。

注意：`EXPOSE` 只是声明，不等于自动映射到宿主机。

#### 8. 启动命令

```dockerfile
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

表示容器启动后运行 Gunicorn。

其中：

```text
app:app
```

第一个 `app` 是文件名 `app.py`，第二个 `app` 是 Flask 对象：

```python
app = Flask(__name__)
```

---

## 八、编写 `.dockerignore`

在 `app` 目录下创建：

```bash
nano .dockerignore
```

写入：

```txt
__pycache__
*.pyc
.env
.git
```

作用：

1. 避免把缓存文件复制进镜像。
2. 避免把 `.env` 敏感配置复制进镜像。
3. 避免把 Git 目录复制进镜像。
4. 减小镜像构建上下文。

---

## 九、编写 Nginx 配置

进入项目根目录：

```bash
cd ~/compose-project
nano nginx/default.conf
```

推荐最终使用下面这版：

```nginx
server {
    listen 80;
    server_name _;

    location / {
        return 200 "Nginx is running. Try /api/health, /api/redis, /api/db\n";
        add_header Content-Type text/plain;
    }

    location /api/ {
        proxy_pass http://web:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Nginx 配置解释

#### 1. 监听 80 端口

```nginx
listen 80;
```

表示 Nginx 容器内部监听 80 端口。

#### 2. 根路径返回提示

```nginx
location / {
    return 200 "Nginx is running. Try /api/health, /api/redis, /api/db\n";
    add_header Content-Type text/plain;
}
```

当访问：

```bash
curl http://localhost/
```

会直接返回文本。

#### 3. `/api/` 转发到 Flask

```nginx
location /api/ {
    proxy_pass http://web:8000;
}
```

含义：

```text
用户访问 http://localhost/api/redis
Nginx 转发到 http://web:8000/api/redis
```

其中 `web` 是 Compose 中 Flask 服务的服务名。

---

## 十、关于 `proxy_pass` 的重点理解

本次排错过程中，`proxy_pass` 是一个重点。

推荐初学者先使用：

```nginx
location /api/ {
    proxy_pass http://web:8000;
}
```

这样 Nginx 会保留原始路径。

例如：

```text
/api/redis → http://web:8000/api/redis
/api/db    → http://web:8000/api/db
```

不要一开始就写成：

```nginx
proxy_pass http://flask_backend/api/;
```

因为带路径的 `proxy_pass` 容易让初学者混淆 URI 替换规则。实际项目中可以使用，但排错时建议先用最简单写法。

---

## 十一、编写 Compose 文件

回到项目根目录：

```bash
cd ~/compose-project
nano compose.yaml
```

写入：

```yaml
services:
  nginx:
    image: nginx:1.27
    container_name: week5-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web
    networks:
      - app-net

  web:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: week5-flask:0.1
    container_name: week5-web
    environment:
      REDIS_HOST: redis
      DB_HOST: db
      POSTGRES_DB: blogdb
      POSTGRES_USER: bloguser
      POSTGRES_PASSWORD: blogpass
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-net

  db:
    image: postgres:16
    container_name: week5-postgres
    environment:
      POSTGRES_DB: blogdb
      POSTGRES_USER: bloguser
      POSTGRES_PASSWORD: blogpass
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bloguser -d blogdb"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - app-net

  redis:
    image: redis:7
    container_name: week5-redis
    networks:
      - app-net

volumes:
  pgdata:

networks:
  app-net:
```

### Compose 文件解释

#### 1. Nginx 服务

```yaml
nginx:
  image: nginx:1.27
  container_name: week5-nginx
  ports:
    - "80:80"
```

表示：

```text
宿主机 80 端口 → Nginx 容器 80 端口
```

外部用户只需要访问：

```bash
http://localhost/
```

#### 2. Nginx 挂载配置文件

```yaml
volumes:
  - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
```

含义：

```text
宿主机 ./nginx/default.conf
挂载到容器 /etc/nginx/conf.d/default.conf
```

`:ro` 表示只读挂载。

#### 3. Web 服务由 Dockerfile 构建

```yaml
web:
  build:
    context: ./app
    dockerfile: Dockerfile
  image: week5-flask:0.1
```

含义：

```text
使用 ./app/Dockerfile 构建镜像
镜像名为 week5-flask:0.1
```

#### 4. 环境变量

```yaml
environment:
  REDIS_HOST: redis
  DB_HOST: db
  POSTGRES_DB: blogdb
  POSTGRES_USER: bloguser
  POSTGRES_PASSWORD: blogpass
```

这些变量会传入 Flask 容器，被 `os.getenv()` 读取。

#### 5. PostgreSQL 数据持久化

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

作用是把数据库数据保存到 Docker volume 中。

如果不使用 volume，容器删除后数据库数据也会丢失。

#### 6. 健康检查

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U bloguser -d blogdb"]
```

用于检测 PostgreSQL 是否真正可以连接。

#### 7. 网络

```yaml
networks:
  app-net:
```

Compose 会创建一个内部网络。加入这个网络的服务，可以通过服务名互相访问。

---

## 十二、启动项目

在项目根目录执行：

```bash
cd ~/compose-project
docker compose up -d --build
```

参数说明：

| 参数 | 含义 |
|---|---|
| `up` | 启动服务 |
| `-d` | 后台运行 |
| `--build` | 启动前重新构建镜像 |

查看容器状态：

```bash
docker compose ps
```

正常应看到：

```text
week5-nginx      Up
week5-web        Up
week5-postgres   Up healthy
week5-redis      Up
```

---

## 十三、测试项目

### 1. 测试 Nginx 首页

```bash
curl http://localhost/
```

预期返回：

```text
Nginx is running. Try /api/health, /api/redis, /api/db
```

### 2. 测试 Flask 健康检查

```bash
curl http://localhost/api/health
```

预期返回：

```json
{"message":"Flask app is healthy","status":"ok"}
```

或者：

```json
{"status":"flask ok"}
```

具体取决于你的 `app.py` 中写的返回内容。

### 3. 测试 Redis

```bash
curl http://localhost/api/redis
```

预期返回：

```json
{"redis_host":"redis","status":"ok","visit_count":"1"}
```

多执行几次：

```bash
curl http://localhost/api/redis
curl http://localhost/api/redis
curl http://localhost/api/redis
```

`visit_count` 应该会不断增加。

这说明：

```text
Nginx → Flask → Redis
```

链路正常。

### 4. 测试 PostgreSQL

```bash
curl http://localhost/api/db
```

预期返回类似：

```json
{
  "db_host": "db",
  "postgres_version": "PostgreSQL 16.x ...",
  "status": "ok"
}
```

这说明：

```text
Nginx → Flask → PostgreSQL
```

链路正常。

---

## 十四、为什么 `/db` 不会转发，而 `/api/db` 会转发

本项目 Nginx 配置是：

```nginx
location /api/ {
    proxy_pass http://web:8000;
}
```

所以只有 `/api/` 开头的请求会转发给 Flask。

正确：

```bash
curl http://localhost/api/db
curl http://localhost/api/redis
curl http://localhost/api/health
```

不正确：

```bash
curl http://localhost/db
curl http://localhost/api.db
```

因为 `/db` 和 `/api.db` 都不匹配 `location /api/`。

它们会匹配：

```nginx
location / {
    return 200 "Nginx is running...";
}
```

所以返回的是 Nginx 首页提示。

---

## 十五、常用管理命令

### 1. 查看服务状态

```bash
docker compose ps
```

### 2. 查看所有日志

```bash
docker compose logs
```

### 3. 查看 Web 服务日志

```bash
docker compose logs web
```

只看最后 30 行：

```bash
docker compose logs --tail=30 web
```

实时查看：

```bash
docker compose logs -f web
```

### 4. 查看 Nginx 日志

```bash
docker compose logs nginx
```

### 5. 重启 Nginx 容器

```bash
docker compose restart nginx
```

### 6. 强制重新创建 Nginx 容器

```bash
docker compose up -d --force-recreate nginx
```

### 7. 重新构建 Web 镜像

```bash
docker compose build --no-cache web
```

### 8. 停止项目

```bash
docker compose down
```

### 9. 停止项目并删除数据库 volume

```bash
docker compose down -v
```

注意：`-v` 会删除 PostgreSQL 数据卷，数据库数据会丢失。

---

## 十六、进入容器排查

### 1. 进入 Flask 容器

```bash
docker compose exec web sh
```

查看文件：

```bash
pwd
ls
cat /app/app.py
```

退出：

```bash
exit
```

### 2. 查看 Flask 容器内真实路由

这是本次排错最有用的命令之一：

```bash
docker compose exec web python -c "from app import app; print(app.url_map)"
```

如果正常，会看到：

```text
/api/health
/api/redis
/api/db
```

本次实际排错中，命令输出证明 Flask 容器内部确实有：

```text
<Route '/api/redis' (HEAD, OPTIONS, GET) -> redis_check>
<Route '/api/db' (HEAD, OPTIONS, GET) -> db_check>
```

这说明 Flask 代码没有问题。

### 3. 用 Flask test_client 在容器内部直接测试

```bash
docker compose exec web python -c "from app import app; c=app.test_client(); r=c.get('/api/redis'); print(r.status_code); print(r.data.decode())"
```

本次排错中返回：

```text
200
{"redis_host":"redis","status":"ok","visit_count":"1"}
```

这进一步说明：

```text
Flask 路由正常
Redis 正常
Flask 连接 Redis 正常
```

### 4. 进入 PostgreSQL 容器

```bash
docker compose exec db bash
```

连接数据库：

```bash
psql -U bloguser -d blogdb
```

执行：

```sql
SELECT version();
\dt
\q
```

退出容器：

```bash
exit
```

### 5. 进入 Redis 容器

```bash
docker compose exec redis redis-cli
```

执行：

```bash
ping
get visit_count
exit
```

---

## 十七、本次排错过程完整复盘

本次问题的现象是：

```bash
curl http://localhost/
```

正常返回 Nginx 首页。

```bash
curl http://localhost/api/health
```

正常返回 Flask 健康状态。

但是：

```bash
curl http://localhost/api/redis
```

返回：

```html
<!doctype html>
<html lang=en>
<title>404 Not Found</title>
<h1>Not Found</h1>
<p>The requested URL was not found on the server...</p>
```

一开始容易误判为 Redis 有问题，但实际上不是。

---

## 十八、排错第一步：确认 Flask 是否有 `/api/redis` 路由

执行：

```bash
grep -n "redis" app/app.py
grep -n "route" app/app.py
```

看到宿主机代码里有：

```python
@app.route("/api/redis")
```

说明宿主机文件里确实写了 Redis 路由。

但宿主机代码存在，不代表容器里的代码已经更新。

所以继续检查容器内部：

```bash
docker compose exec web python -c "from app import app; print(app.url_map)"
```

看到：

```text
<Route '/api/redis' (HEAD, OPTIONS, GET) -> redis_check>
```

说明容器内 Flask 也有这个路由。

因此排除：

```text
不是 Flask 路由缺失
不是镜像旧代码问题
```

---

## 十九、排错第二步：确认 Redis 是否正常

在 Flask 容器内部测试：

```bash
docker compose exec web python -c "from app import app; c=app.test_client(); r=c.get('/api/redis'); print(r.status_code); print(r.data.decode())"
```

返回：

```text
200
{"redis_host":"redis","status":"ok","visit_count":"1"}
```

说明：

```text
Flask 内部调用 /api/redis 正常
Flask 连接 Redis 正常
Redis 容器正常
```

因此排除：

```text
不是 Redis 问题
不是 Flask 连接 Redis 问题
```

---

## 二十、排错第三步：怀疑 Nginx 转发路径

因为 Flask 内部测试正常，但是从外部通过 Nginx 访问 404：

```bash
curl http://localhost/api/redis
```

返回 404。

这说明问题在：

```text
curl → Nginx → Flask
```

这一段，重点是 Nginx。

因此建议检查 Nginx 容器实际加载的配置：

```bash
docker compose exec nginx nginx -T | grep -A20 -B5 "location /api"
```

并把配置改成最简单、最明确的写法：

```nginx
location /api/ {
    proxy_pass http://web:8000;
}
```

---

## 二十一、排错第四步：确认不是 `localhost:8000` 问题

执行：

```bash
curl http://localhost:8000/api/redis
```

返回：

```text
Failed to connect to localhost port 8000
```

这并不代表 Flask 挂了。

原因是：`compose.yaml` 中 `web` 服务没有映射端口：

```yaml
web:
  # 没有 ports:
  #   - "8000:8000"
```

所以宿主机不能直接访问 Flask 的 8000。

这是正常的生产式结构：

```text
外部用户只能访问 Nginx
Flask 只在 Docker 内部网络中暴露给 Nginx
```

---

## 二十二、排错第五步：发现系统 Nginx 占用了 80 端口

后来执行：

```bash
sudo ss -tulnp | grep ':80'
```

发现输出中有：

```text
tcp LISTEN 0 511 0.0.0.0:80 users:(("nginx",pid=...))
```

这说明：

```text
WSL 系统里的 Nginx 正在占用 80 端口
```

不是 Docker Nginx。

这就是问题关键之一。

### 停止系统 Nginx

执行：

```bash
sudo systemctl stop nginx
```

再次查看：

```bash
sudo ss -tulnp | grep ':80'
```

发现 `:80` 不再被系统 Nginx 占用。

然后重新启动 Docker 项目：

```bash
docker compose down
docker compose up -d
```

此时 Docker Nginx 才真正接管 80 端口。

---

## 二十三、最终问题解决结果

停止系统 Nginx 并重新启动 Docker Compose 后，测试成功：

```bash
curl http://localhost/api/redis
```

返回：

```json
{"redis_host":"redis","status":"ok","visit_count":"1"}
```

测试 PostgreSQL：

```bash
curl http://localhost/api/db
```

返回类似：

```json
{
  "db_host": "db",
  "postgres_version": "PostgreSQL 16.14 ...",
  "status": "ok"
}
```

说明完整链路已经正常：

```text
curl
 ↓
Docker Nginx:80
 ↓
Flask Web:8000
 ↓           ↓
Redis       PostgreSQL
```

---

## 二十四、本次排错中的关键结论

### 1. `sudo systemctl reload nginx` 重载的不是 Docker Nginx

曾经执行过：

```bash
sudo systemctl reload nginx
```

这个命令重载的是 WSL 系统里的 Nginx。

但是本项目使用的是 Docker 容器里的 Nginx：

```text
week5-nginx
```

所以应该使用：

```bash
docker compose restart nginx
```

或者更彻底：

```bash
docker compose up -d --force-recreate nginx
```

---

### 2. 系统 Nginx 和 Docker Nginx 可能抢占 80 端口

如果系统 Nginx 已经占用 80，Docker Nginx 可能无法正确绑定 80。

查看端口占用：

```bash
sudo ss -tulnp | grep ':80'
```

如果看到：

```text
nginx
```

说明系统 Nginx 占用。

如果看到：

```text
docker-proxy
```

说明 Docker 正在占用。

停止系统 Nginx：

```bash
sudo systemctl stop nginx
```

禁止系统 Nginx 开机自启：

```bash
sudo systemctl disable nginx
```

以后如果要重新启用：

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

### 3. Nginx 官方镜像里可能没有 `wget`

曾经执行：

```bash
docker compose exec nginx sh -c "wget -qO- http://web:8000/api/redis"
```

返回：

```text
wget: not found
```

这不是网络问题，只是 `nginx:1.27` 镜像里默认没有安装 `wget`。

如果要在 Docker 网络中临时测试，可以使用 curl 镜像：

```bash
docker run --rm --network compose-project_app-net curlimages/curl http://web:8000/api/redis
```

---

### 4. 查看端口占用的命令

查看 80 端口：

```bash
sudo ss -tulnp | grep ':80'
```

查看 8000 端口：

```bash
lsof -i :8000
sudo ss -tulnp | grep ':8000'
```

本项目中 `8000` 不需要暴露到宿主机，所以 `localhost:8000` 访问失败是正常的。

---

## 二十五、404 的几种常见原因

### 情况 1：Flask 没有对应路由

现象：

```bash
curl http://localhost/api/redis
```

返回 404。

检查：

```bash
grep -n "api/redis" app/app.py
```

如果没有：

```python
@app.route("/api/redis")
```

说明 Flask 没有这个接口。

---

### 情况 2：宿主机代码改了，但容器没重新构建

检查容器内部代码：

```bash
docker compose exec web grep -n "api/redis" /app/app.py
```

或查看路由：

```bash
docker compose exec web python -c "from app import app; print(app.url_map)"
```

如果容器内没有新路由，需要重新构建：

```bash
docker compose down
docker compose build --no-cache web
docker compose up -d
```

---

### 情况 3：Nginx `proxy_pass` 路径写错

推荐写法：

```nginx
location /api/ {
    proxy_pass http://web:8000;
}
```

然后重建 Nginx：

```bash
docker compose up -d --force-recreate nginx
```

---

### 情况 4：访问的不是 Docker Nginx，而是系统 Nginx

检查：

```bash
sudo ss -tulnp | grep ':80'
```

如果看到系统 Nginx：

```bash
sudo systemctl stop nginx
```

然后：

```bash
docker compose down
docker compose up -d
```

---

## 二十六、502 的常见原因

如果 Nginx 返回 502，通常说明：

```text
Nginx 正常运行，但是后端 Flask 不可用
```

排查命令：

```bash
docker compose ps
docker compose logs web
docker compose logs nginx
```

常见原因：

1. Flask 容器退出了。
2. Gunicorn 启动失败。
3. Nginx 代理地址写错。
4. Flask 没有监听 `0.0.0.0:8000`。
5. Web 服务没有加入同一个 Docker network。

---

## 二十七、修改代码后的正确流程

### 1. 修改 Flask 代码

如果修改了：

```text
app/app.py
```

建议执行：

```bash
docker compose up -d --build
```

如果仍不生效：

```bash
docker compose down
docker compose build --no-cache web
docker compose up -d
```

### 2. 修改 Nginx 配置

如果修改了：

```text
nginx/default.conf
```

执行：

```bash
docker compose up -d --force-recreate nginx
```

或者：

```bash
docker compose restart nginx
```

检查容器内 Nginx 实际配置：

```bash
docker compose exec nginx nginx -T | grep -A20 -B5 "location /api"
```

---

## 二十八、推荐最终命令清单

### 启动项目

```bash
cd ~/compose-project
docker compose up -d --build
```

### 查看状态

```bash
docker compose ps
```

### 测试接口

```bash
curl http://localhost/
curl http://localhost/api/health
curl http://localhost/api/redis
curl http://localhost/api/db
```

### 查看日志

```bash
docker compose logs
```

### 查看 Web 日志

```bash
docker compose logs --tail=30 web
```

### 查看 Nginx 日志

```bash
docker compose logs --tail=30 nginx
```

### 查看 Flask 路由

```bash
docker compose exec web python -c "from app import app; print(app.url_map)"
```

### 在 Flask 容器内部测试 Redis 路由

```bash
docker compose exec web python -c "from app import app; c=app.test_client(); r=c.get('/api/redis'); print(r.status_code); print(r.data.decode())"
```

### 查看 80 端口占用

```bash
sudo ss -tulnp | grep ':80'
```

### 停止系统 Nginx

```bash
sudo systemctl stop nginx
```

### 禁止系统 Nginx 开机自启

```bash
sudo systemctl disable nginx
```

### 重新构建 Web 镜像

```bash
docker compose build --no-cache web
```

### 强制重建 Nginx 容器

```bash
docker compose up -d --force-recreate nginx
```

### 停止项目

```bash
docker compose down
```

### 停止项目并删除数据库数据

```bash
docker compose down -v
```

---

## 二十九、本阶段应该真正理解的内容

完成本项目后，需要能独立说清楚：

1. Dockerfile 是用来构建镜像的。
2. Docker Compose 是用来编排多个容器的。
3. Nginx 是整个项目的对外入口。
4. Flask 不直接暴露给宿主机，而是在 Docker 网络中被 Nginx 访问。
5. Redis 和 PostgreSQL 也不需要暴露到宿主机。
6. 容器之间通信不能用 `localhost`，应该用 Compose 服务名。
7. `web:8000` 表示访问 `web` 服务的 8000 端口。
8. `db` 表示 PostgreSQL 服务名。
9. `redis` 表示 Redis 服务名。
10. `ports: "80:80"` 表示宿主机 80 端口映射到容器 80 端口。
11. `volumes: pgdata:/var/lib/postgresql/data` 用于数据库持久化。
12. 修改 Flask 代码后需要重新 build Web 镜像。
13. 修改 Nginx 配置后需要重启或强制重建 Nginx 容器。
14. 系统 Nginx 和 Docker Nginx 可能抢占 80 端口。
15. `sudo systemctl reload nginx` 操作的是系统 Nginx，不是 Docker Nginx。
16. 404 不一定是后端没启动，也可能是路由或代理路径问题。
17. 502 通常表示 Nginx 找不到可用后端。
18. `localhost:8000` 不能访问不一定是错误，可能是没有映射端口。

---

## 三十、面试可能会问的问题与参考答案

### 1. Dockerfile 和 Docker Compose 有什么区别？

Dockerfile 用来构建单个镜像，描述镜像中应该安装什么、复制什么代码、启动什么命令。Docker Compose 用来编排多个容器，比如同时启动 Nginx、Web、数据库和 Redis。

---

### 2. 为什么容器里连接数据库不能写 `localhost`？

因为容器中的 `localhost` 指的是当前容器自己。如果 Flask 容器里写 `localhost`，它会尝试连接 Flask 容器自身，而不是 PostgreSQL 容器。在 Compose 网络中应该使用服务名，例如 `db`。

---

### 3. 为什么本项目只暴露 Nginx 的 80 端口？

因为 Nginx 是统一入口。Flask、PostgreSQL 和 Redis 都属于内部服务，不应该直接暴露给外部。这样更安全，也更接近真实生产部署。

---

### 4. `depends_on` 能保证数据库完全可用吗？

普通 `depends_on` 只能保证容器启动顺序，不能保证服务已经完全可用。为了让 Web 等待 PostgreSQL 真正可连接，可以配合 `healthcheck` 和：

```yaml
depends_on:
  db:
    condition: service_healthy
```

---

### 5. Nginx 返回 502 通常说明什么？

说明 Nginx 本身能处理请求，但是后端服务不可用。常见原因包括 Flask 容器没启动、Web 端口写错、服务名写错、Flask 没监听 `0.0.0.0`。

---

### 6. Nginx 返回 404 一定是 Nginx 的问题吗？

不一定。404 可能由 Nginx 返回，也可能是 Flask 返回。需要通过日志、路由表、容器内部测试判断。比如本次问题中，Flask 内部路由正常，最终发现是系统 Nginx 端口占用和代理链路混乱导致。

---

### 7. 如何确认 Flask 容器里实际有哪些路由？

可以执行：

```bash
docker compose exec web python -c "from app import app; print(app.url_map)"
```

---

### 8. 如何确认 80 端口被谁占用？

执行：

```bash
sudo ss -tulnp | grep ':80'
```

如果看到 `nginx`，说明系统 Nginx 占用。若看到 `docker-proxy`，说明 Docker 容器占用。

---

### 9. 修改 Flask 代码后为什么访问结果没变？

因为 Flask 代码被复制进镜像了。修改宿主机文件后，如果没有 volume 挂载源码，就需要重新构建镜像：

```bash
docker compose up -d --build
```

或：

```bash
docker compose build --no-cache web
```

---

### 10. PostgreSQL 为什么要使用 volume？

数据库数据保存在容器内部文件系统中，如果容器被删除，数据可能丢失。使用 volume 可以把数据持久化，即使容器删除后，数据仍然保留在 Docker volume 中。

---

## 三十一、下一步建议学习内容

完成本项目后，可以继续学习：

1. `.env` 环境变量文件管理。
2. PostgreSQL 初始化 SQL 脚本。
3. Nginx access.log 和 error.log 分析。
4. Docker volume 数据备份与恢复。
5. Docker network 深入理解。
6. 多阶段构建优化镜像体积。
7. 使用非 root 用户运行容器。
8. 使用 healthcheck 检查 Flask 健康状态。
9. 使用 GitHub Actions 自动构建镜像。
10. 将项目部署到云服务器。

---

## 三十二、最终复盘

本周项目从零完成了一个完整的 Docker Compose 多服务项目：

```text
Nginx + Flask + PostgreSQL + Redis
```

你不仅完成了项目搭建，还经历了真实排错过程：

1. `/api/redis` 返回 404。
2. 检查 Flask 代码中是否有路由。
3. 检查容器内 Flask 是否加载了新代码。
4. 用 Flask `test_client` 证明 Redis 路由正常。
5. 怀疑 Nginx 代理路径。
6. 修改 `proxy_pass http://web:8000;`。
7. 检查 80 端口占用。
8. 发现系统 Nginx 占用了 80。
9. 停止系统 Nginx。
10. 重启 Docker Compose。
11. 最终 `/api/redis` 和 `/api/db` 全部访问成功。

这个过程非常接近真实运维工作中的排错方式。真正重要的不是只把项目跑通，而是理解每一步背后的原因：

```text
请求从哪里来？
到了哪个容器？
路径有没有被改？
服务名能不能解析？
容器有没有加载新代码？
端口到底被谁占用？
日志说明了什么？
```

掌握这些思路后，你就已经开始具备 Docker + Nginx + 后端服务部署的基础运维能力。
