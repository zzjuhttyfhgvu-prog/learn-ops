# Stage 3：Shell 脚本学习笔记

> 主题：把重复的 Linux / Docker / PostgreSQL / 服务检查命令组合成脚本，实现基础自动化运维。  
> 适合阶段：已经会手动敲 `docker ps`、`docker exec`、`curl`、`mkdir`、`find`、`grep` 等命令，下一步开始学习 Shell 脚本。

---

## 目录

1. [本阶段学习目标](#1-本阶段学习目标)
2. [Shell 脚本是什么](#2-shell-脚本是什么)
3. [实验环境准备](#3-实验环境准备)
4. [Shell 脚本基础语法](#4-shell-脚本基础语法)
5. [项目一：PostgreSQL 数据库备份脚本](#5-项目一postgresql-数据库备份脚本)
6. [当前 Docker 状态分析：为什么备份脚本暂时不能运行](#6-当前-docker-状态分析为什么备份脚本暂时不能运行)
7. [创建并启动 PostgreSQL 容器](#7-创建并启动-postgresql-容器)
8. [项目二：服务健康检查脚本](#8-项目二服务健康检查脚本)
9. [定时执行脚本：cron / crontab](#9-定时执行脚本cron--crontab)
10. [常见错误与排查方法](#10-常见错误与排查方法)
11. [7 天练习计划](#11-7-天练习计划)
12. [本阶段完成标准](#12-本阶段完成标准)

---

# 1. 本阶段学习目标

第三阶段的核心目标是：

> 不再只会手动敲命令，而是能够把重复操作写成 Shell 脚本，让系统自动执行。

你需要重点掌握：

1. 变量
2. `if` 判断
3. `for` 循环
4. 函数
5. 命令退出状态 `$?`
6. 日志输出
7. 参数传递
8. 定时执行脚本

完成本阶段后，你应该能够写出两个典型运维脚本：

1. PostgreSQL 数据库自动备份脚本；
2. Nginx / Prometheus / Grafana / PostgreSQL 服务健康检查脚本。

---

# 2. Shell 脚本是什么

Shell 脚本本质上就是一个文本文件，里面写了一组 Linux 命令。

平时你可能会手动执行：

```bash
mkdir -p ~/backups
docker ps
date
```

如果把这些命令写进一个文件：

```bash
#!/bin/bash

mkdir -p ~/backups
docker ps
date
```

然后执行这个文件，系统就会自动按顺序运行这些命令。

这就是 Shell 脚本。

## 2.1 Shell 脚本在运维中的作用

在运维工作中，Shell 脚本常用于：

1. 自动备份数据库；
2. 自动检查服务是否正常；
3. 批量创建目录；
4. 批量处理日志；
5. 自动部署服务；
6. 定时清理旧文件；
7. 监控服务器状态；
8. 配合 `cron` 实现定时任务。

---

# 3. 实验环境准备

建议先创建一个专门的练习目录：

```bash
mkdir -p ~/shell-lab
cd ~/shell-lab
```

以后本阶段所有脚本都放到这个目录下。

查看当前目录：

```bash
pwd
```

查看目录内容：

```bash
ls -lh
```

---

# 4. Shell 脚本基础语法

---

## 4.1 `#!/bin/bash` 的作用

每个 Bash 脚本第一行通常写：

```bash
#!/bin/bash
```

这行叫 **shebang**。

它的作用是告诉系统：

> 这个脚本文件要交给 `/bin/bash` 来解释执行。

如果没有这行，有些情况下脚本仍然可以运行，但不够规范。不同系统或不同 Shell 下可能出现兼容问题。

---

## 4.2 变量

变量用来保存值。

示例：

```bash
NAME="blogdb"
BACKUP_DIR="$HOME/backups"

echo "$NAME"
echo "$BACKUP_DIR"
```

注意：

```bash
NAME="blogdb"
```

等号两边不能有空格。

错误写法：

```bash
NAME = "blogdb"
```

变量使用时建议加双引号：

```bash
echo "$NAME"
```

原因是：如果变量内容里包含空格，不加双引号可能会被 Shell 错误拆分。

例如：

```bash
DIR="$HOME/my backups"
mkdir -p "$DIR"
```

这里必须加双引号，否则 `my backups` 可能会被拆成两个参数。

---

## 4.3 命令替换

命令替换用于把一条命令的输出结果保存到变量里。

例如获取当前日期：

```bash
DATE=$(date +%F_%H-%M-%S)
echo "$DATE"
```

其中：

```bash
$(...)
```

表示先执行括号里的命令，然后把结果替换到当前位置。

例如：

```bash
date +%F_%H-%M-%S
```

可能输出：

```text
2026-06-05_15-30-20
```

所以：

```bash
DATE=$(date +%F_%H-%M-%S)
```

就是把当前日期时间保存到 `DATE` 变量中。

在备份脚本中，这个变量常用于生成带日期的备份文件名。

---

## 4.4 `if` 判断

基本格式：

```bash
if 条件; then
    命令
else
    命令
fi
```

示例：判断目录是否存在。

```bash
if [ -d "$HOME/backups" ]; then
    echo "backups directory exists"
else
    echo "backups directory does not exist"
fi
```

常见判断条件：

| 条件 | 含义 |
|---|---|
| `-d 路径` | 判断目录是否存在 |
| `-f 路径` | 判断普通文件是否存在 |
| `-z 字符串` | 判断字符串是否为空 |
| `-n 字符串` | 判断字符串是否非空 |
| `数字1 -eq 数字2` | 判断两个数字是否相等 |
| `数字1 -ne 数字2` | 判断两个数字是否不相等 |
| `数字1 -gt 数字2` | 判断数字1是否大于数字2 |
| `数字1 -lt 数字2` | 判断数字1是否小于数字2 |

示例：判断文件是否存在。

```bash
if [ -f "/etc/hosts" ]; then
    echo "file exists"
else
    echo "file does not exist"
fi
```

---

## 4.5 `for` 循环

`for` 循环用于重复执行命令。

```bash
for name in nginx prometheus grafana; do
    echo "checking $name"
done
```

输出：

```text
checking nginx
checking prometheus
checking grafana
```

在运维中，`for` 循环常用于：

1. 批量检查多个服务；
2. 批量创建目录；
3. 批量处理日志文件；
4. 批量执行命令。

---

## 4.6 函数

函数用于把一组命令封装起来，方便重复调用。

示例：

```bash
check_url() {
    local name="$1"
    local url="$2"

    echo "Checking $name: $url"
    curl -fsS "$url" >/dev/null

    if [ $? -eq 0 ]; then
        echo "$name is OK"
    else
        echo "$name is DOWN"
    fi
}

check_url "Nginx" "http://localhost:80"
```

函数中的参数：

| 参数 | 含义 |
|---|---|
| `$1` | 函数接收的第一个参数 |
| `$2` | 函数接收的第二个参数 |
| `$3` | 函数接收的第三个参数 |

函数的好处是：

1. 减少重复代码；
2. 让脚本结构更清楚；
3. 以后修改逻辑时，只需要改函数本身。

---

## 4.7 命令退出状态 `$?`

Linux 中每条命令执行结束后，都会返回一个状态码。

查看上一条命令的状态码：

```bash
echo $?
```

规则：

| 状态码 | 含义 |
|---|---|
| `0` | 成功 |
| 非 `0` | 失败 |

例如：

```bash
curl http://localhost:80
echo $?
```

如果访问成功，通常返回：

```text
0
```

如果访问失败，可能返回非 0。

因此脚本中经常这样判断：

```bash
curl -fsS http://localhost:80 >/dev/null

if [ $? -eq 0 ]; then
    echo "Nginx OK"
else
    echo "Nginx failed"
fi
```

这是 Shell 脚本中非常重要的思想：

> 脚本不能只执行命令，还必须判断命令是否执行成功。

---

## 4.8 日志输出

脚本不能只把结果打印到屏幕上，因为屏幕输出很快就消失了。

更好的做法是写入日志文件。

示例：

```bash
LOG_FILE="$HOME/shell-lab/script.log"

echo "script started" | tee -a "$LOG_FILE"
```

其中：

```bash
tee -a
```

表示：

1. 在屏幕上显示内容；
2. 同时追加写入日志文件。

日志中建议加入时间：

```bash
echo "[$(date '+%F %T')] script started" | tee -a "$LOG_FILE"
```

这样以后排查问题时，可以知道：

1. 脚本什么时候执行；
2. 是否成功；
3. 失败发生在哪一步。

---

## 4.9 参数传递

脚本可以接收外部传入的参数。

创建测试脚本：

```bash
nano args_demo.sh
```

写入：

```bash
#!/bin/bash

echo "script name: $0"
echo "first argument: $1"
echo "second argument: $2"
echo "all arguments: $@"
echo "argument count: $#"
```

保存后执行：

```bash
chmod +x args_demo.sh
./args_demo.sh blogdb learn-postgres
```

输出类似：

```text
script name: ./args_demo.sh
first argument: blogdb
second argument: learn-postgres
all arguments: blogdb learn-postgres
argument count: 2
```

常见参数：

| 参数 | 含义 |
|---|---|
| `$0` | 脚本名称 |
| `$1` | 第一个参数 |
| `$2` | 第二个参数 |
| `$@` | 所有参数 |
| `$#` | 参数个数 |
| `$?` | 上一条命令退出状态 |
| `$$` | 当前脚本进程 ID |

---

# 5. 项目一：PostgreSQL 数据库备份脚本

---

## 5.1 项目目标

编写脚本：

```bash
backup_blogdb.sh
```

实现：

1. 自动备份 PostgreSQL 的 `blogdb`；
2. 备份文件名带日期；
3. 保存到 `~/backups` 目录；
4. 只保留最近 7 天的备份；
5. 输出日志。

---

## 5.2 先确认 PostgreSQL 容器是否正在运行

执行：

```bash
docker ps
```

如果 PostgreSQL 容器正在运行，你应该能看到类似：

```text
learn-postgres   postgres:16   Up ...   0.0.0.0:5432->5432/tcp
```

如果只看到类似：

```text
buildx_buildkit_armbuilder0
```

说明当前没有正在运行的 PostgreSQL 容器。

这个问题在本笔记后面的第 6 节和第 7 节会详细说明。

---

## 5.3 手动测试备份命令

在写脚本前，先手动测试命令是否能正常运行。

创建备份目录：

```bash
mkdir -p ~/backups
```

执行测试备份：

```bash
docker exec learn-postgres pg_dump -U admin blogdb > ~/backups/blogdb_test.sql
```

查看文件是否生成：

```bash
ls -lh ~/backups
```

如果看到：

```text
blogdb_test.sql
```

说明备份命令是通的。

---

## 5.4 编写数据库备份脚本

进入练习目录：

```bash
cd ~/shell-lab
nano backup_blogdb.sh
```

写入：

```bash
#!/bin/bash

BACKUP_DIR="$HOME/backups"
LOG_FILE="$BACKUP_DIR/backup.log"

DATE=$(date +%F_%H-%M-%S)

DB_NAME="blogdb"
DB_USER="admin"
CONTAINER_NAME="learn-postgres"

BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql"

mkdir -p "$BACKUP_DIR"

echo "[$(date '+%F %T')] Backup started." | tee -a "$LOG_FILE"

docker exec "$CONTAINER_NAME" pg_dump -U "$DB_USER" "$DB_NAME" > "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "[$(date '+%F %T')] Backup completed: $BACKUP_FILE" | tee -a "$LOG_FILE"
else
    echo "[$(date '+%F %T')] Backup failed." | tee -a "$LOG_FILE"
    exit 1
fi

find "$BACKUP_DIR" -name "${DB_NAME}_*.sql" -mtime +7 -delete

echo "[$(date '+%F %T')] Old backups cleaned." | tee -a "$LOG_FILE"
```

保存退出。

---

## 5.5 给脚本执行权限

```bash
chmod +x backup_blogdb.sh
```

查看权限：

```bash
ls -l backup_blogdb.sh
```

如果看到类似：

```text
-rwxr-xr-x
```

说明脚本已经具有执行权限。

其中：

| 权限字符 | 含义 |
|---|---|
| `r` | read，读权限 |
| `w` | write，写权限 |
| `x` | execute，执行权限 |

---

## 5.6 运行脚本

```bash
./backup_blogdb.sh
```

查看备份文件：

```bash
ls -lh ~/backups
```

查看日志：

```bash
cat ~/backups/backup.log
```

---

## 5.7 数据库备份脚本的运行原理

核心命令是：

```bash
docker exec "$CONTAINER_NAME" pg_dump -U "$DB_USER" "$DB_NAME" > "$BACKUP_FILE"
```

完整流程如下：

```text
宿主机 Shell 脚本
        |
        | docker exec
        v
进入 PostgreSQL 容器
        |
        | pg_dump -U admin blogdb
        v
导出 blogdb 数据库内容
        |
        | > 输出重定向
        v
保存到宿主机 ~/backups 目录
```

关键点解释：

### 1. `docker exec`

```bash
docker exec learn-postgres pg_dump -U admin blogdb
```

表示在已经运行的 `learn-postgres` 容器内部执行 `pg_dump` 命令。

前提是：

> `learn-postgres` 容器必须存在，并且必须正在运行。

### 2. `pg_dump`

`pg_dump` 是 PostgreSQL 官方提供的数据库导出工具。

```bash
pg_dump -U admin blogdb
```

含义：

| 参数 | 含义 |
|---|---|
| `pg_dump` | 导出 PostgreSQL 数据库 |
| `-U admin` | 使用 `admin` 用户连接数据库 |
| `blogdb` | 要备份的数据库名称 |

### 3. `>` 输出重定向

```bash
> "$BACKUP_FILE"
```

表示把前面命令的输出写入文件。

如果文件不存在，会自动创建。  
如果文件已存在，会覆盖。

---

## 5.8 清理 7 天前备份的原理

脚本中的清理命令：

```bash
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql" -mtime +7 -delete
```

拆开理解：

| 片段 | 含义 |
|---|---|
| `find "$BACKUP_DIR"` | 在备份目录中查找文件 |
| `-name "${DB_NAME}_*.sql"` | 只匹配 `blogdb_*.sql` 类型文件 |
| `-mtime +7` | 修改时间超过 7 天 |
| `-delete` | 删除匹配到的文件 |

示例：

```text
blogdb_2026-06-05_15-30-20.sql
blogdb_2026-06-06_02-00-00.sql
```

这些文件都符合：

```bash
blogdb_*.sql
```

---

## 5.9 为什么要做恢复测试

备份文件生成了，并不代表备份一定可用。

真正可靠的备份，必须能够恢复。

查看 SQL 文件内容：

```bash
head ~/backups/blogdb_*.sql
```

创建测试恢复库：

```bash
docker exec -it learn-postgres createdb -U admin blogdb_test_restore
```

恢复备份到测试库：

```bash
cat ~/backups/你的备份文件.sql | docker exec -i learn-postgres psql -U admin blogdb_test_restore
```

这里用到了：

```bash
docker exec -i
```

`-i` 表示保持标准输入打开，这样才能把宿主机上的 SQL 文件内容输入到容器内部的 `psql` 命令中。

---

# 6. 当前 Docker 状态分析：为什么备份脚本暂时不能运行

根据你当前截图中的情况：

```bash
docker ps
```

只显示：

```text
buildx_buildkit_armbuilder0
```

这说明当前正在运行的容器只有 Docker Buildx 的构建容器。

它不是 PostgreSQL 容器。

另外，Docker Desktop 的 Images 页面显示有：

```text
postgres:16
```

这说明你本地已经有 PostgreSQL 镜像。

但是：

> 有镜像不等于有容器；有容器不等于容器正在运行。

---

## 6.1 镜像和容器的区别

可以这样理解：

```text
镜像 image      = 安装包 / 模板
容器 container  = 根据镜像运行起来的软件实例
```

例如：

```text
postgres:16 镜像
```

相当于你已经下载了 PostgreSQL 的安装模板。

但是你还需要用这个镜像启动一个容器：

```text
learn-postgres 容器
```

只有容器运行起来后，才能执行：

```bash
docker exec learn-postgres pg_dump -U admin blogdb
```

---

## 6.2 查看所有容器，包括停止的容器

执行：

```bash
docker ps -a
```

或者更清楚地查看：

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

可能出现三种情况。

### 情况一：没有 `learn-postgres`

说明你还没有创建 PostgreSQL 容器。

需要执行第 7 节的 `docker run` 命令创建它。

### 情况二：有 `learn-postgres`，但状态是 `Exited`

说明容器存在，但已经停止。

启动它：

```bash
docker start learn-postgres
```

然后检查：

```bash
docker ps
```

### 情况三：有 `learn-postgres`，状态是 `Up`

说明容器正在运行。

这时可以继续执行备份脚本。

---

# 7. 创建并启动 PostgreSQL 容器

如果你执行：

```bash
docker ps -a
```

没有看到 `learn-postgres`，可以用下面命令创建一个新的 PostgreSQL 容器。

```bash
docker run --name learn-postgres \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=admin123 \
  -e POSTGRES_DB=blogdb \
  -p 5432:5432 \
  -d postgres:16
```

---

## 7.1 命令参数解释

| 参数 | 含义 |
|---|---|
| `docker run` | 根据镜像创建并启动一个新容器 |
| `--name learn-postgres` | 容器名字叫 `learn-postgres` |
| `-e POSTGRES_USER=admin` | 创建数据库用户 `admin` |
| `-e POSTGRES_PASSWORD=admin123` | 设置用户密码为 `admin123` |
| `-e POSTGRES_DB=blogdb` | 自动创建数据库 `blogdb` |
| `-p 5432:5432` | 把容器的 5432 端口映射到宿主机的 5432 端口 |
| `-d` | 后台运行容器 |
| `postgres:16` | 使用 `postgres:16` 镜像 |

---

## 7.2 检查容器是否运行

```bash
docker ps
```

应该看到类似：

```text
learn-postgres   postgres:16   Up ...   0.0.0.0:5432->5432/tcp
```

---

## 7.3 进入 PostgreSQL 测试

```bash
docker exec -it learn-postgres psql -U admin -d blogdb
```

如果成功，会进入 PostgreSQL 命令行：

```text
blogdb=#
```

退出：

```sql
\q
```

---

## 7.4 测试备份命令

```bash
mkdir -p ~/backups

docker exec learn-postgres pg_dump -U admin blogdb > ~/backups/blogdb_test.sql

ls -lh ~/backups
```

看到 `blogdb_test.sql` 后，再运行脚本：

```bash
cd ~/shell-lab
chmod +x backup_blogdb.sh
./backup_blogdb.sh
```

---

## 7.5 如果提示容器名已经存在

如果执行 `docker run` 时出现：

```text
Conflict. The container name "/learn-postgres" is already in use
```

说明这个容器名字已经被占用。

先查看：

```bash
docker ps -a
```

如果容器只是停止了，启动即可：

```bash
docker start learn-postgres
```

如果你确定不需要旧容器，可以删除后重建：

```bash
docker rm learn-postgres
```

然后重新执行 `docker run`。

注意：删除容器可能会丢失容器内未持久化的数据。学习阶段可以这么做，生产环境不能随便删除。

---

# 8. 项目二：服务健康检查脚本

---

## 8.1 项目目标

编写脚本：

```bash
check_services.sh
```

检查：

1. Nginx 是否能访问；
2. Prometheus 是否能访问；
3. Grafana 是否能访问；
4. PostgreSQL 容器是否正在运行；
5. 输出检查日志。

---

## 8.2 常见服务端口

| 服务 | 常见端口 | 示例访问地址 |
|---|---:|---|
| Nginx | 80 或 8080 | `http://localhost:80` 或 `http://localhost:8080` |
| Prometheus | 9090 | `http://localhost:9090` |
| Grafana | 3000 | `http://localhost:3000` |
| PostgreSQL | 5432 | 通常不用浏览器访问 |

如果 Nginx 是这样启动的：

```bash
docker run -p 8080:80 nginx
```

那么宿主机访问地址是：

```text
http://localhost:8080
```

如果 Nginx 映射到宿主机 80 端口，则访问：

```text
http://localhost:80
```

---

## 8.3 手动测试服务

先手动测试：

```bash
curl -I http://localhost:80
curl -I http://localhost:9090
curl -I http://localhost:3000
docker ps
```

如果 Nginx 使用的是 8080：

```bash
curl -I http://localhost:8080
```

---

## 8.4 编写健康检查脚本

进入脚本目录：

```bash
cd ~/shell-lab
nano check_services.sh
```

写入：

```bash
#!/bin/bash

LOG_DIR="$HOME/service-logs"
LOG_FILE="$LOG_DIR/health_check.log"

NGINX_URL="http://localhost:80"
PROMETHEUS_URL="http://localhost:9090"
GRAFANA_URL="http://localhost:3000"

POSTGRES_CONTAINER="learn-postgres"

mkdir -p "$LOG_DIR"

log() {
    echo "[$(date '+%F %T')] $1" | tee -a "$LOG_FILE"
}

check_url() {
    local name="$1"
    local url="$2"

    curl -fsS "$url" >/dev/null

    if [ $? -eq 0 ]; then
        log "$name is OK: $url"
    else
        log "$name is DOWN: $url"
    fi
}

check_container() {
    local container_name="$1"

    docker ps --format "{{.Names}}" | grep -w "$container_name" >/dev/null

    if [ $? -eq 0 ]; then
        log "Container is running: $container_name"
    else
        log "Container is NOT running: $container_name"
    fi
}

log "Health check started."

check_url "Nginx" "$NGINX_URL"
check_url "Prometheus" "$PROMETHEUS_URL"
check_url "Grafana" "$GRAFANA_URL"

check_container "$POSTGRES_CONTAINER"

log "Health check finished."
```

如果你的 Nginx 是 8080，需要修改：

```bash
NGINX_URL="http://localhost:8080"
```

---

## 8.5 给脚本执行权限并运行

```bash
chmod +x check_services.sh
./check_services.sh
```

查看日志：

```bash
cat ~/service-logs/health_check.log
```

---

## 8.6 健康检查脚本的原理

### 1. `curl -fsS`

```bash
curl -fsS "$url" >/dev/null
```

含义：

| 参数 | 含义 |
|---|---|
| `-f` | 如果 HTTP 状态码是错误状态，则认为失败 |
| `-s` | 静默模式，不显示进度条 |
| `-S` | 如果失败，显示错误信息 |
| `>/dev/null` | 丢弃正常输出，只关心成功或失败 |

执行完后，用 `$?` 判断结果：

```bash
if [ $? -eq 0 ]; then
    log "$name is OK"
else
    log "$name is DOWN"
fi
```

### 2. 为什么用函数

下面三行检查逻辑完全一样，只是服务名和 URL 不一样：

```bash
check_url "Nginx" "$NGINX_URL"
check_url "Prometheus" "$PROMETHEUS_URL"
check_url "Grafana" "$GRAFANA_URL"
```

所以把重复逻辑封装成函数：

```bash
check_url() {
    ...
}
```

这样脚本更清晰，也更容易维护。

### 3. PostgreSQL 容器检查原理

```bash
docker ps --format "{{.Names}}" | grep -w "$container_name" >/dev/null
```

拆开理解：

| 命令片段 | 含义 |
|---|---|
| `docker ps` | 查看正在运行的容器 |
| `--format "{{.Names}}"` | 只输出容器名 |
| `grep -w "$container_name"` | 精确查找指定容器名 |
| `>/dev/null` | 不显示输出，只根据退出状态判断是否找到 |

如果找到了，`grep` 返回 `0`。  
如果没找到，`grep` 返回非 `0`。

---

# 9. 定时执行脚本：cron / crontab

运维中，备份脚本和健康检查脚本通常不会手动运行，而是定时运行。

Linux 中常用 `cron` 实现定时任务。

---

## 9.1 编辑定时任务

```bash
crontab -e
```

第一次打开时，可能会让你选择编辑器。

新手建议选择：

```text
nano
```

---

## 9.2 每天凌晨 2 点备份数据库

先查看当前用户名：

```bash
whoami
```

假设用户名是 `zzjuh`，则脚本路径可能是：

```text
/home/zzjuh/shell-lab/backup_blogdb.sh
```

在 `crontab` 中加入：

```cron
0 2 * * * /home/zzjuh/shell-lab/backup_blogdb.sh
```

如果你的用户名不是 `zzjuh`，要替换成你自己的用户名。

---

## 9.3 每 5 分钟检查一次服务

```cron
*/5 * * * * /home/zzjuh/shell-lab/check_services.sh
```

---

## 9.4 cron 时间格式

```text
* * * * * command
| | | | |
| | | | +---- 星期几，0-7，0 和 7 都表示周日
| | | +------ 月份，1-12
| | +-------- 日期，1-31
| +---------- 小时，0-23
+------------ 分钟，0-59
```

常见示例：

| 表达式 | 含义 |
|---|---|
| `0 2 * * *` | 每天 2:00 执行 |
| `*/5 * * * *` | 每 5 分钟执行一次 |
| `30 8 * * 1` | 每周一 8:30 执行 |
| `0 0 * * 0` | 每周日 0:00 执行 |

---

## 9.5 cron 中为什么要用绝对路径

不要在 `crontab` 中写：

```cron
*/5 * * * * ./check_services.sh
```

应该写完整路径：

```cron
*/5 * * * * /home/你的用户名/shell-lab/check_services.sh
```

原因是：

> cron 执行任务时，默认工作目录不一定是你当前的终端目录。

所以脚本路径、日志路径、命令路径都尽量写清楚。

---

# 10. 常见错误与排查方法

---

## 10.1 `Permission denied`

错误示例：

```text
Permission denied
```

原因：脚本没有执行权限。

解决：

```bash
chmod +x script.sh
```

---

## 10.2 `command not found`

可能原因：

1. 命令写错；
2. 脚本第一行没有 `#!/bin/bash`；
3. Windows 换行符导致脚本异常；
4. cron 环境变量与终端环境不同。

如果怀疑是 Windows 换行符问题，可以执行：

```bash
dos2unix script.sh
```

如果没有安装：

```bash
sudo apt install dos2unix -y
```

---

## 10.3 `docker: permission denied`

如果普通用户执行 Docker 报权限错误，先测试：

```bash
docker ps
```

如果没有权限，可以执行：

```bash
sudo usermod -aG docker $USER
```

然后退出 WSL 或终端，重新登录。

---

## 10.4 `No such container: learn-postgres`

说明 `learn-postgres` 容器不存在。

检查所有容器：

```bash
docker ps -a
```

如果没有，需要创建：

```bash
docker run --name learn-postgres \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=admin123 \
  -e POSTGRES_DB=blogdb \
  -p 5432:5432 \
  -d postgres:16
```

---

## 10.5 容器存在但没有运行

查看：

```bash
docker ps -a
```

如果状态是 `Exited`，启动：

```bash
docker start learn-postgres
```

---

## 10.6 PostgreSQL 备份文件为空

需要检查：

### 1. 容器是否运行

```bash
docker ps
```

### 2. 数据库是否存在

```bash
docker exec -it learn-postgres psql -U admin -l
```

### 3. 是否能连接数据库

```bash
docker exec -it learn-postgres psql -U admin -d blogdb
```

### 4. 脚本是否报错

```bash
cat ~/backups/backup.log
```

---

## 10.7 cron 没有执行

查看当前定时任务：

```bash
crontab -l
```

查看 cron 日志：

```bash
grep CRON /var/log/syslog
```

如果在 WSL 中 cron 没启动，可以执行：

```bash
sudo service cron start
```

---

# 11. 7 天练习计划

---

## 第 1 天：变量、`echo`、`date`

创建脚本：

```bash
nano demo_variable.sh
```

写入：

```bash
#!/bin/bash

NAME="blogdb"
DATE=$(date +%F)

echo "Database name: $NAME"
echo "Today is: $DATE"
```

执行：

```bash
chmod +x demo_variable.sh
./demo_variable.sh
```

目标：理解变量和命令替换。

---

## 第 2 天：`if` 判断和 `$?`

```bash
nano demo_if.sh
```

写入：

```bash
#!/bin/bash

curl -fsS http://localhost:80 >/dev/null

if [ $? -eq 0 ]; then
    echo "Nginx is OK"
else
    echo "Nginx is DOWN"
fi
```

执行：

```bash
chmod +x demo_if.sh
./demo_if.sh
```

目标：理解命令成功和失败如何判断。

---

## 第 3 天：`for` 循环

```bash
nano demo_for.sh
```

写入：

```bash
#!/bin/bash

for service in nginx prometheus grafana; do
    echo "checking $service"
done
```

执行：

```bash
chmod +x demo_for.sh
./demo_for.sh
```

目标：理解批量处理。

---

## 第 4 天：函数

```bash
nano demo_function.sh
```

写入：

```bash
#!/bin/bash

say_hello() {
    echo "Hello, $1"
}

say_hello "Nginx"
say_hello "PostgreSQL"
```

执行：

```bash
chmod +x demo_function.sh
./demo_function.sh
```

目标：理解函数和函数参数。

---

## 第 5 天：完成数据库备份脚本

重点练习命令：

```bash
docker exec
pg_dump
>
find -mtime +7 -delete
tee -a
```

目标：独立写出 `backup_blogdb.sh`。

---

## 第 6 天：完成服务健康检查脚本

重点练习命令：

```bash
curl -fsS
docker ps
grep
if
function
log
```

目标：独立写出 `check_services.sh`。

---

## 第 7 天：配置定时任务

重点练习命令：

```bash
crontab -e
crontab -l
```

目标：让备份脚本和健康检查脚本自动执行。

---

# 12. 本阶段完成标准

学完这一阶段，你应该能做到：

1. 看懂简单 Shell 脚本；
2. 会写变量、`if`、`for`、函数；
3. 会判断命令是否执行成功；
4. 会把日志写入文件；
5. 会给脚本传参数；
6. 会写 PostgreSQL 数据库备份脚本；
7. 会写服务健康检查脚本；
8. 会用 `crontab` 定时执行脚本；
9. 能区分 Docker 镜像和容器；
10. 能判断容器是否存在、是否正在运行；
11. 能根据日志排查脚本执行失败的原因；
12. 初步具备自动化运维思维。

---

# 13. 最后总结

Shell 脚本不是一门脱离 Linux 命令的新技术。

它的本质是：

> 把你已经会的命令，按照固定逻辑组织起来，并加入变量、判断、循环、函数、日志和定时任务。

对于运维学习来说，Shell 脚本是从“手动操作”走向“自动化操作”的关键一步。

你现在最需要重点掌握的是：

```text
命令能不能执行成功？
失败后怎么判断？
结果有没有日志？
能不能定时自动执行？
脚本重复执行会不会出问题？
```

只要围绕这些问题练习，你就会逐渐形成真正的运维脚本思维。
