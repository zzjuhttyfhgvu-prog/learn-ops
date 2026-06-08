# 第六阶段：CI/CD 入门学习笔记

> 目标：理解并实践“开发者提交代码后，服务如何自动测试、构建镜像、推送镜像，并最终自动更新上线”的完整流程。

---

## 目录

1. [本阶段学习目标](#1-本阶段学习目标)
2. [CI/CD 的核心概念](#2-cicd-的核心概念)
3. [整体流程图](#3-整体流程图)
4. [准备练习项目 learn-cicd-demo](#4-准备练习项目-learn-cicd-demo)
5. [创建 Flask 示例服务](#5-创建-flask-示例服务)
6. [创建自动测试](#6-创建自动测试)
7. [创建 Dockerfile 与 .dockerignore](#7-创建-dockerfile-与-dockerignore)
8. [本地运行与测试](#8-本地运行与测试)
9. [Git 分支与本地提交](#9-git-分支与本地提交)
10. [推送项目到 GitHub](#10-推送项目到-github)
11. [GitHub Actions：自动测试](#11-github-actions自动测试)
12. [GitHub Actions：自动构建 Docker 镜像](#12-github-actions自动构建-docker-镜像)
13. [GitHub Actions：自动推送 Docker Hub](#13-github-actions自动推送-docker-hub)
14. [升级：服务器自动部署](#14-升级服务器自动部署)
15. [本次实操中的错误与解决方法](#15-本次实操中的错误与解决方法)
16. [常见错误排查清单](#16-常见错误排查清单)
17. [推荐练习任务](#17-推荐练习任务)
18. [面试常问问题](#18-面试常问问题)
19. [阶段验收标准](#19-阶段验收标准)

---

## 1. 本阶段学习目标

当你已经学习过 GitHub、Docker、Ansible 之后，就可以进入 CI/CD 阶段。

本阶段重点学习内容：

1. Git 分支
2. GitHub Actions
3. 自动测试
4. 自动构建 Docker 镜像
5. 自动推送 Docker 镜像
6. 自动部署

最终要理解：

```text
开发提交代码
  ↓
GitHub Actions 自动运行
  ↓
自动测试
  ↓
自动构建 Docker 镜像
  ↓
自动推送 Docker Hub
  ↓
服务器拉取新镜像
  ↓
Docker Compose 重启服务
  ↓
新版本上线
```

---

## 2. CI/CD 的核心概念

### 2.1 什么是 CI？

CI 是 Continuous Integration，中文叫“持续集成”。

它的核心作用是：

```text
每次提交代码后，自动检查代码能不能正常运行。
```

例如：

```text
git push
  ↓
GitHub Actions 自动启动
  ↓
安装 Python 依赖
  ↓
运行 pytest
  ↓
测试通过或失败
```

CI 的价值：

- 尽早发现代码错误；
- 避免把坏代码合并到主分支；
- 让项目变得更加稳定；
- 减少人工重复测试。

### 2.2 什么是 CD？

CD 有两种常见解释：

1. Continuous Delivery：持续交付；
2. Continuous Deployment：持续部署。

在你当前阶段，可以先简单理解为：

```text
测试通过后，自动构建镜像并部署服务。
```

例如：

```text
代码 push 到 GitHub
  ↓
自动测试
  ↓
自动构建 Docker 镜像
  ↓
自动推送 Docker Hub
  ↓
服务器自动拉取新镜像
  ↓
服务自动重启
```

### 2.3 CI 和 CD 的区别

| 概念 | 主要作用 | 典型操作 |
|---|---|---|
| CI | 检查代码是否正确 | 自动测试、代码检查、构建验证 |
| CD | 把正确的代码发布出去 | 构建镜像、推送镜像、自动部署 |

---

## 3. 整体流程图

本阶段目标流程：

```text
开发者
  |
  | git push
  v
GitHub Repository
  |
  | 触发 workflow
  v
GitHub Actions Runner
  |
  | checkout code
  v
运行测试 pytest
  |
  | 测试通过
  v
docker build
  |
  | 构建成功
  v
docker login
  |
  | 登录成功
  v
docker push
  |
  v
Docker Hub
  |
  | 服务器 pull
  v
Linux Server
  |
  | docker compose up -d
  v
新版本服务上线
```

你需要理解：

```text
CI/CD 本质上就是把人工执行的命令，写进自动化流程里。
```

---

## 4. 准备练习项目 learn-cicd-demo

### 4.1 创建项目目录

在 WSL 中执行：

```bash
mkdir learn-cicd-demo
cd learn-cicd-demo
```

### 4.2 初始化 Git 仓库

```bash
git init
```

你可能会看到类似提示：

```text
Using 'master' as the name for the initial branch
```

这不是错误，只是 Git 默认创建了 `master` 分支。

后面可以改成 `main`：

```bash
git branch -M main
```

查看当前分支：

```bash
git branch
```

正常应该显示：

```text
* main
```

---

## 5. 创建 Flask 示例服务

### 5.1 创建 app.py

在项目根目录执行：

```bash
nano app.py
```

写入：

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def index():
    return jsonify({"message": "hello cicd"})

@app.get("/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

保存退出：

```text
Ctrl + O
Enter
Ctrl + X
```

### 5.2 代码解释

```python
from flask import Flask, jsonify
```

导入 Flask 框架和 JSON 响应工具。

```python
app = Flask(__name__)
```

创建 Flask 应用对象。

```python
@app.get("/")
def index():
    return jsonify({"message": "hello cicd"})
```

定义首页接口，访问 `/` 时返回一段 JSON。

```python
@app.get("/health")
def health():
    return jsonify({"status": "ok"})
```

定义健康检查接口。CI/CD 和运维中经常使用 `/health` 判断服务是否正常。

```python
app.run(host="0.0.0.0", port=8000)
```

让 Flask 监听所有网卡的 8000 端口。

---

## 6. 创建自动测试

### 6.1 创建 requirements.txt

```bash
nano requirements.txt
```

写入：

```txt
flask
pytest
```

### 6.2 创建 tests 目录

```bash
mkdir tests
```

### 6.3 创建测试文件

```bash
nano tests/test_app.py
```

写入：

```python
from app import app

def test_health():
    client = app.test_client()
    response = client.get("/health")

    assert response.status_code == 200
    assert response.get_json()["status"] == "ok"
```

### 6.4 测试文件解释

```python
from app import app
```

从项目根目录的 `app.py` 中导入 Flask 应用对象。

```python
client = app.test_client()
```

创建一个测试客户端，不需要真正启动浏览器或 curl，就能模拟请求。

```python
response = client.get("/health")
```

模拟访问 `/health` 接口。

```python
assert response.status_code == 200
```

判断 HTTP 状态码是否为 200。

```python
assert response.get_json()["status"] == "ok"
```

判断接口返回的 JSON 数据是否符合预期。

---

## 7. 创建 Dockerfile 与 .dockerignore

### 7.1 创建 Dockerfile

```bash
nano Dockerfile
```

写入：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

### 7.2 Dockerfile 解释

```dockerfile
FROM python:3.12-slim
```

使用 Python 3.12 的轻量基础镜像。

```dockerfile
WORKDIR /app
```

设置容器内部工作目录为 `/app`。

```dockerfile
COPY requirements.txt .
```

先复制依赖文件。

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

安装 Python 依赖。

```dockerfile
COPY . .
```

复制项目代码到容器中。

```dockerfile
EXPOSE 8000
```

声明容器服务监听 8000 端口。

```dockerfile
CMD ["python", "app.py"]
```

容器启动时运行 Flask 应用。

### 7.3 创建 .dockerignore

```bash
nano .dockerignore
```

写入：

```txt
.git
__pycache__
.pytest_cache
.venv
```

作用：构建 Docker 镜像时，不把这些无关文件打包进去。

---

## 8. 本地运行与测试

### 8.1 创建 Python 虚拟环境

```bash
python3 -m venv .venv
```

### 8.2 激活虚拟环境

```bash
source .venv/bin/activate
```

激活后命令行前面会出现：

```text
(.venv)
```

### 8.3 安装依赖

```bash
pip install -r requirements.txt
```

### 8.4 运行 pytest

```bash
pytest
```

正常结果应该类似：

```text
collected 1 item

tests/test_app.py .                                             [100%]

1 passed
```

### 8.5 本地运行 Flask 服务

```bash
python app.py
```

打开另一个终端执行：

```bash
curl http://localhost:8000/health
```

正常返回：

```json
{"status":"ok"}
```

### 8.6 本地构建 Docker 镜像

```bash
docker build -t learn-cicd-demo:local .
```

### 8.7 本地运行 Docker 容器

```bash
docker run --name learn-cicd-demo -d -p 8000:8000 learn-cicd-demo:local
```

测试：

```bash
curl http://localhost:8000/health
```

停止并删除容器：

```bash
docker stop learn-cicd-demo
docker rm learn-cicd-demo
```

---

## 9. Git 分支与本地提交

### 9.1 查看当前分支

```bash
git branch
```

### 9.2 把默认分支改成 main

如果当前是 `master`，执行：

```bash
git branch -M main
```

注意：如果 `main` 还不存在，不要执行：

```bash
git checkout main
```

否则会报错：

```text
error: pathspec 'main' did not match any file(s) known to git
```

正确做法是：

```bash
git branch -M main
```

### 9.3 创建 .gitignore

Python 项目中不应该提交虚拟环境和缓存文件。

创建 `.gitignore`：

```bash
cat > .gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
.pytest_cache/
EOF
```

### 9.4 移除已经被 Git 追踪的缓存文件

如果之前不小心把 `__pycache__`、`.pyc` 文件提交进去了，可以执行：

```bash
git rm -r --cached tests/__pycache__ 2>/dev/null || true
git rm -r --cached __pycache__ 2>/dev/null || true
git rm -r --cached .pytest_cache 2>/dev/null || true
```

### 9.5 提交代码

```bash
git status
git add .
git commit -m "init cicd demo"
```

---

## 10. 推送项目到 GitHub

### 10.1 在 GitHub 创建仓库

建议仓库名：

```text
learn-cicd-demo
```

### 10.2 添加远程仓库地址

如果你的 GitHub 用户名是：

```text
zzjuhttyfhgvu-prog
```

则执行：

```bash
git remote add origin git@github.com:zzjuhttyfhgvu-prog/learn-cicd-demo.git
```

如果已经有 origin，但是地址不对，执行：

```bash
git remote set-url origin git@github.com:zzjuhttyfhgvu-prog/learn-cicd-demo.git
```

查看远程地址：

```bash
git remote -v
```

### 10.3 推送 main 分支

```bash
git push -u origin main
```

如果提示：

```text
Repository not found
```

通常说明：

1. GitHub 上还没有创建这个仓库；
2. 仓库名写错；
3. SSH key 没有配置好；
4. 当前 GitHub 账号没有权限。

---

## 11. GitHub Actions：自动测试

GitHub Actions 的 workflow 文件通常放在：

```text
.github/workflows/
```

### 11.1 创建目录

```bash
mkdir -p .github/workflows
```

### 11.2 创建 ci.yml

```bash
nano .github/workflows/ci.yml
```

写入：

```yaml
name: CI Test

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  test:
    name: Run Python Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest
```

### 11.3 workflow 解释

```yaml
name: CI Test
```

工作流名称。

```yaml
on:
  push:
    branches:
      - main
```

当代码 push 到 main 分支时触发。

```yaml
pull_request:
```

当有人向 main 分支提交 Pull Request 时触发。

```yaml
runs-on: ubuntu-latest
```

GitHub 会提供一台临时 Ubuntu 机器运行任务。

```yaml
uses: actions/checkout@v4
```

把仓库代码拉取到临时机器中。

```yaml
uses: actions/setup-python@v5
```

安装指定版本的 Python。

```yaml
run: pytest
```

运行测试。

### 11.4 提交并推送 workflow

```bash
git add .
git commit -m "add github actions test workflow"
git push
```

然后进入 GitHub 仓库页面：

```text
Actions
```

查看工作流是否执行成功。

---

## 12. GitHub Actions：自动构建 Docker 镜像

在测试通过后，再构建 Docker 镜像。

修改 `.github/workflows/ci.yml`：

```yaml
name: CI Build

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  test:
    name: Run Python Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest

  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build -t learn-cicd-demo:test .
```

关键点：

```yaml
needs: test
```

含义：

```text
test 成功后，docker-build 才会执行。
```

如果测试失败，Docker 构建不会执行。

---

## 13. GitHub Actions：自动推送 Docker Hub

### 13.1 准备 Docker Hub 仓库

登录 Docker Hub，创建一个仓库，例如：

```text
learn-cicd-demo
```

假设 Docker Hub 用户名是：

```text
yourdockername
```

最终镜像名就是：

```text
yourdockername/learn-cicd-demo:latest
```

### 13.2 创建 Docker Hub Token

不要把 Docker Hub 密码写进 GitHub Actions。

建议创建 Docker Hub Personal Access Token，用于 CI/CD 自动登录。

大致路径：

```text
Docker Hub
  ↓
Account settings
  ↓
Personal access tokens
  ↓
Generate new token
```

权限选择：

```text
Read & Write
```

### 13.3 在 GitHub 添加 Secrets

进入 GitHub 仓库：

```text
Settings
  ↓
Secrets and variables
  ↓
Actions
  ↓
New repository secret
```

添加两个 Secret：

```text
DOCKERHUB_USERNAME
```

值填写 Docker Hub 用户名。

```text
DOCKERHUB_TOKEN
```

值填写 Docker Hub Token。

### 13.4 完整 workflow：测试 + 构建 + 推送

```yaml
name: CI Docker Build and Push

on:
  push:
    branches:
      - main

jobs:
  test:
    name: Run Python Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest

  docker:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v4
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Docker image
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:latest .

      - name: Push Docker image
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:latest
```

### 13.5 使用 commit ID 作为镜像标签

只使用 `latest` 不方便定位具体版本。

可以同时打两个标签：

```yaml
      - name: Build Docker image
        run: |
          docker build \
            -t ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:latest \
            -t ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:${{ github.sha }} \
            .

      - name: Push Docker image
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:latest
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:${{ github.sha }}
```

理解：

```text
latest = 最新版本
github.sha = 本次提交对应的精确版本
```

生产环境更推荐使用明确版本号，而不是只依赖 `latest`。

---

## 14. 升级：服务器自动部署

升级后的流程：

```text
push 代码
  ↓
GitHub Actions
  ↓
测试
  ↓
构建 Docker 镜像
  ↓
推送 Docker Hub
  ↓
SSH 登录服务器
  ↓
docker compose pull
  ↓
docker compose up -d
```

### 14.1 服务器准备目录

在服务器上执行：

```bash
sudo mkdir -p /opt/learn-cicd-demo
sudo chown -R $USER:$USER /opt/learn-cicd-demo
cd /opt/learn-cicd-demo
```

### 14.2 创建服务器端 compose.yaml

```bash
nano compose.yaml
```

写入：

```yaml
services:
  web:
    image: yourdockername/learn-cicd-demo:latest
    container_name: learn-cicd-demo
    ports:
      - "8000:8000"
    restart: always
```

把 `yourdockername` 换成你的 Docker Hub 用户名。

### 14.3 服务器手动测试

```bash
docker compose pull
docker compose up -d
```

查看容器：

```bash
docker ps
```

测试服务：

```bash
curl http://localhost:8000/health
```

### 14.4 生成 SSH 部署密钥

在本地 WSL 执行：

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy
```

生成：

```text
~/.ssh/github_actions_deploy      私钥
~/.ssh/github_actions_deploy.pub  公钥
```

私钥放到 GitHub Secrets；公钥放到服务器 `~/.ssh/authorized_keys`。

### 14.5 把公钥放到服务器

本地查看公钥：

```bash
cat ~/.ssh/github_actions_deploy.pub
```

登录服务器：

```bash
ssh 用户名@服务器IP
```

在服务器执行：

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

把公钥粘贴进去。

设置权限：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 14.6 测试 SSH 登录

本地执行：

```bash
ssh -i ~/.ssh/github_actions_deploy 用户名@服务器IP
```

能登录说明密钥配置正确。

### 14.7 GitHub 添加部署 Secrets

在 GitHub 仓库添加：

```text
SERVER_HOST
SERVER_USER
SERVER_SSH_KEY
```

查看私钥：

```bash
cat ~/.ssh/github_actions_deploy
```

完整复制，包括：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

### 14.8 完整 CI/CD 部署 workflow

```yaml
name: CI CD Deploy

on:
  push:
    branches:
      - main

jobs:
  test:
    name: Run Python Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest

  docker:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v4
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Docker image
        run: |
          docker build \
            -t ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:latest \
            -t ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:${{ github.sha }} \
            .

      - name: Push Docker image
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:latest
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/learn-cicd-demo:${{ github.sha }}

  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    needs: docker

    steps:
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SERVER_SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy with Docker Compose
        run: |
          ssh -i ~/.ssh/deploy_key ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} "
            cd /opt/learn-cicd-demo &&
            docker compose pull &&
            docker compose up -d &&
            docker image prune -f
          "
```

---

## 15. 本次实操中的错误与解决方法

### 15.1 错误一：pytest 报 `ModuleNotFoundError: No module named 'app'`

报错内容：

```text
ModuleNotFoundError: No module named 'app'
```

测试文件中有：

```python
from app import app
```

这表示 Python 要从当前项目中找到 `app.py` 文件。

如果项目根目录没有 `app.py`，就会报错。

#### 解决方法一：确认当前目录

```bash
cd ~/learn-cicd-demo
pwd
```

应该显示：

```text
/home/zy/learn-cicd-demo
```

#### 解决方法二：确认 app.py 是否存在

```bash
ls -la
```

应该看到：

```text
app.py
```

如果没有，就重新创建：

```bash
cat > app.py <<'EOF'
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def index():
    return jsonify({"message": "hello cicd"})

@app.get("/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
EOF
```

#### 解决方法三：重新写测试文件

```bash
cat > tests/test_app.py <<'EOF'
from app import app

def test_health():
    client = app.test_client()
    response = client.get("/health")

    assert response.status_code == 200
    assert response.get_json()["status"] == "ok"
EOF
```

#### 解决方法四：添加 pytest.ini

```bash
cat > pytest.ini <<'EOF'
[pytest]
pythonpath = .
testpaths = tests
EOF
```

作用：告诉 pytest 从当前项目根目录寻找 Python 模块。

#### 解决方法五：测试 import 是否正常

```bash
python -c "import app; print(app)"
```

正常输出类似：

```text
<module 'app' from '/home/zy/learn-cicd-demo/app.py'>
```

再运行：

```bash
pytest
```

### 15.2 错误二：测试文件名写成 test_qpp.py

你之前出现过类似文件：

```text
tests/test_qpp.py
```

虽然文件名不一定导致错误，但容易混淆。

建议改成：

```bash
mv tests/test_qpp.py tests/test_app.py
```

如果提示不存在，说明已经改过，可以忽略。

### 15.3 错误三：误提交 Python 缓存文件

你之前的提交中出现了：

```text
tests/__pycache__/test_qpp.cpython-312-pytest-9.0.3.pyc
```

这是 Python 自动生成的缓存文件，不应该提交。

解决：

```bash
cat > .gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
.pytest_cache/
EOF

git rm -r --cached tests/__pycache__ 2>/dev/null || true
git rm -r --cached __pycache__ 2>/dev/null || true
git rm -r --cached .pytest_cache 2>/dev/null || true

git add .gitignore
git commit -m "add gitignore and remove python cache files"
```

### 15.4 错误四：`git checkout main` 报错

报错：

```text
error: pathspec 'main' did not match any file(s) known to git
```

原因：当前仓库里还没有 `main` 分支。

错误操作：

```bash
git checkout main
```

正确操作：

```bash
git branch -M main
```

然后查看：

```bash
git branch
```

应该显示：

```text
* main
```

---

## 16. 常见错误排查清单

### 16.1 pytest 找不到模块

现象：

```text
ModuleNotFoundError: No module named 'app'
```

检查：

```bash
pwd
ls -la
find . -maxdepth 3 -type f
python -c "import app; print(app)"
```

重点确认：

```text
app.py 必须在项目根目录
测试文件中 from app import app 要和文件名一致
pytest.ini 中 pythonpath = .
```

### 16.2 Docker 构建失败

检查：

```bash
ls -la
cat Dockerfile
cat requirements.txt
```

常见原因：

```text
Dockerfile 文件名写错
requirements.txt 不存在
依赖安装失败
当前目录不是项目根目录
```

### 16.3 Docker 容器端口访问失败

检查容器：

```bash
docker ps
```

检查日志：

```bash
docker logs learn-cicd-demo
```

确认端口映射：

```bash
docker run --name learn-cicd-demo -d -p 8000:8000 learn-cicd-demo:local
```

### 16.4 GitHub Actions 测试失败

看 GitHub 仓库：

```text
Actions
  ↓
点击失败的 workflow
  ↓
点击失败的 job
  ↓
查看具体报错日志
```

常见原因：

```text
requirements.txt 缺少 pytest
app.py 没有提交
测试文件路径错误
YAML 缩进错误
```

### 16.5 Docker Hub 登录失败

检查 GitHub Secrets：

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
```

检查 YAML：

```yaml
username: ${{ secrets.DOCKERHUB_USERNAME }}
password: ${{ secrets.DOCKERHUB_TOKEN }}
```

常见原因：

```text
Secret 名字大小写不一致
Docker Hub token 填错
token 没有 Read & Write 权限
Docker Hub 用户名写错
```

### 16.6 Docker push 失败

常见原因：

```text
Docker Hub 仓库不存在
镜像名写错
没有登录成功
用户名不匹配
```

正确格式：

```text
DockerHub用户名/仓库名:标签
```

例如：

```text
yourdockername/learn-cicd-demo:latest
```

### 16.7 SSH 自动部署失败

常见报错：

```text
Permission denied
```

检查：

```bash
ssh -i ~/.ssh/github_actions_deploy 用户名@服务器IP
```

服务器权限：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

GitHub Secrets：

```text
SERVER_HOST
SERVER_USER
SERVER_SSH_KEY
```

---

## 17. 推荐练习任务

### 练习 1：只做自动测试

目标：

```text
push 代码后自动执行 pytest
```

你要完成：

```text
.github/workflows/ci.yml
```

并在 GitHub Actions 中看到绿色通过。

### 练习 2：故意制造测试失败

把 `app.py` 里的：

```python
return jsonify({"status": "ok"})
```

改成：

```python
return jsonify({"status": "bad"})
```

提交：

```bash
git add .
git commit -m "break health check"
git push
```

观察 GitHub Actions 失败。

再改回来：

```python
return jsonify({"status": "ok"})
```

再次提交并观察恢复成功。

这个练习用于理解：

```text
CI 的作用不是保证每次都成功，而是帮你及时发现失败。
```

### 练习 3：自动构建 Docker 镜像

目标：

```text
pytest 成功后自动 docker build
```

重点观察：

```text
test job 成功后，docker-build job 才开始。
```

### 练习 4：自动推送 Docker Hub

目标：

```text
GitHub Actions 自动 push 镜像到 Docker Hub
```

完成后去 Docker Hub 查看：

```text
learn-cicd-demo:latest
```

### 练习 5：服务器自动部署

目标：

```text
push 代码后服务器自动更新服务
```

验证：

```bash
curl http://服务器IP:8000/health
```

---

## 18. 面试常问问题

### 18.1 什么是 CI/CD？

CI 是持续集成，指代码提交后自动进行测试、检查和构建，尽早发现问题。

CD 是持续交付或持续部署，指测试通过后自动把应用发布到测试环境或生产环境。

### 18.2 GitHub Actions 的核心组成是什么？

主要包括：

```text
workflow
job
step
runner
action
secret
```

解释：

- workflow：整个自动化流程；
- job：流程中的任务；
- step：任务中的具体步骤；
- runner：执行任务的机器；
- action：别人封装好的可复用动作；
- secret：保存密码、token、私钥等敏感信息。

### 18.3 为什么要先测试再构建镜像？

因为测试失败说明代码可能有问题。

如果不测试就构建和部署，可能会把错误版本发布到服务器上。

正确流程：

```text
测试失败
  ↓
停止流程
  ↓
不构建镜像
  ↓
不部署
```

### 18.4 `needs` 有什么作用？

`needs` 用来控制 job 的执行顺序。

例如：

```yaml
needs: test
```

表示当前 job 必须等 `test` 成功后才执行。

### 18.5 为什么不能把密码写进 ci.yml？

因为 `.github/workflows/ci.yml` 会提交到代码仓库。

如果把密码写进去，就可能泄露。

错误示例：

```yaml
password: 123456
```

正确示例：

```yaml
password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### 18.6 `latest` 标签有什么问题？

`latest` 只能表示最新镜像，但不能准确说明对应哪次代码提交。

生产环境更推荐：

```text
Git commit SHA
v1.0.0
v1.0.1
```

---

## 19. 阶段验收标准

学完这一阶段后，你应该能独立完成：

```text
修改 app.py
  ↓
git add .
  ↓
git commit
  ↓
git push
  ↓
GitHub Actions 自动运行 pytest
  ↓
自动 docker build
  ↓
自动 docker push
  ↓
服务器自动 docker compose pull
  ↓
服务器自动 docker compose up -d
  ↓
浏览器访问新版本服务
```

如果你能跑通这个流程，说明你已经真正入门 CI/CD。

---

## 附录 A：推荐最终项目结构

```text
learn-cicd-demo/
├── .github/
│   └── workflows/
│       └── ci.yml
├── tests/
│   └── test_app.py
├── .dockerignore
├── .gitignore
├── app.py
├── Dockerfile
├── pytest.ini
└── requirements.txt
```

服务器目录：

```text
/opt/learn-cicd-demo/
└── compose.yaml
```

---

## 附录 B：最小可用命令汇总

### 初始化项目

```bash
mkdir learn-cicd-demo
cd learn-cicd-demo
git init
git branch -M main
```

### 创建 app.py

```bash
cat > app.py <<'EOF'
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def index():
    return jsonify({"message": "hello cicd"})

@app.get("/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
EOF
```

### 创建 requirements.txt

```bash
cat > requirements.txt <<'EOF'
flask
pytest
EOF
```

### 创建测试

```bash
mkdir -p tests
cat > tests/test_app.py <<'EOF'
from app import app

def test_health():
    client = app.test_client()
    response = client.get("/health")

    assert response.status_code == 200
    assert response.get_json()["status"] == "ok"
EOF
```

### 创建 pytest.ini

```bash
cat > pytest.ini <<'EOF'
[pytest]
pythonpath = .
testpaths = tests
EOF
```

### 创建 Dockerfile

```bash
cat > Dockerfile <<'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
EOF
```

### 创建忽略文件

```bash
cat > .gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
.pytest_cache/
EOF

cat > .dockerignore <<'EOF'
.git
__pycache__
.pytest_cache
.venv
EOF
```

### 本地测试

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pytest
```

### Docker 测试

```bash
docker build -t learn-cicd-demo:local .
docker run --name learn-cicd-demo -d -p 8000:8000 learn-cicd-demo:local
curl http://localhost:8000/health
docker stop learn-cicd-demo
docker rm learn-cicd-demo
```

### Git 提交与推送

```bash
git add .
git commit -m "init cicd demo"
git remote add origin git@github.com:zzjuhttyfhgvu-prog/learn-cicd-demo.git
git push -u origin main
```

---

## 附录 C：学习重点总结

你需要重点掌握下面几个能力：

1. 看懂 GitHub Actions 的 YAML 文件；
2. 理解 workflow、job、step、runner 的关系；
3. 会用 pytest 做最基本的自动测试；
4. 会写 Dockerfile；
5. 会在 GitHub Actions 中构建镜像；
6. 会使用 GitHub Secrets 保存敏感信息；
7. 会推送镜像到 Docker Hub；
8. 会通过 SSH 和 Docker Compose 实现自动部署；
9. 会根据报错日志定位问题。

---

