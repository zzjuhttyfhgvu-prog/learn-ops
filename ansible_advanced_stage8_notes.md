# Ansible 进阶学习笔记：Roles、变量分层、条件、循环、Tags 与 Vault

> 适用阶段：第八阶段：Ansible 进阶  
> 前置基础：已经学习过 inventory、tasks、handlers、templates、vars  
> 学习目标：能够使用 Ansible Role 组织真实项目，并掌握 group_vars、host_vars、when、loop、tags、ansible-vault 的实际用法。

---

## 目录

1. 本阶段学习目标
2. 本阶段核心知识点
3. 推荐学习路线
4. 实验环境准备
5. 项目目录结构
6. 创建 ansible.cfg
7. 创建 inventory.ini
8. 学习 roles
9. 学习 defaults 与 vars
10. 学习 tasks、when、loop、tags
11. 学习 handlers
12. 学习 templates
13. 学习 group_vars
14. 学习 host_vars
15. 运行完整 playbook
16. 学习 tags 局部执行
17. 学习 ansible-vault
18. 综合练习项目
19. 常见问题排查
20. 核心原理总结
21. 面试常见问题与参考答案
22. 阶段完成标准
23. 下一阶段建议

---

# 1. 本阶段学习目标

你前面已经学过 Ansible 的基础内容：

- inventory
- tasks
- handlers
- templates
- vars

这些内容已经可以让你写一个简单的 playbook。

但是在真实公司或真实项目中，Ansible 通常不会只写一个大 playbook，而是会把配置拆成标准结构：

```text
roles/
group_vars/
host_vars/
site.yml
inventory.ini
ansible.cfg
```

本阶段的目标是：从“会写简单 playbook”升级到“会组织一个真实 Ansible 自动化项目”。

最终你应该能够做到：

```text
1. 把一个普通 playbook 拆成 role
2. 使用 group_vars 管理主机组变量
3. 使用 host_vars 管理单台主机变量
4. 使用 when 进行条件判断
5. 使用 loop 批量执行任务
6. 使用 tags 局部执行任务
7. 使用 ansible-vault 加密敏感变量
8. 一条命令自动部署 Nginx
```

---

# 2. 本阶段核心知识点

## 2.1 roles

Role 是 Ansible 推荐的标准项目组织方式。

普通 playbook 可能是这样：

```text
deploy-nginx.yml
inventory.ini
```

进阶后的项目结构会变成：

```text
roles/
  nginx/
    tasks/
    handlers/
    templates/
    defaults/
    vars/
```

Role 的作用是把不同功能拆分成模块。

例如：

```text
roles/nginx/       专门负责部署 Nginx
roles/postgresql/  专门负责部署 PostgreSQL
roles/docker/      专门负责部署 Docker
```

这样项目更清晰，也更容易复用。

---

## 2.2 group_vars

`group_vars` 用于给一组主机设置变量。

例如 inventory 中有：

```ini
[web]
web1
web2

[db]
db1
```

那么可以创建：

```text
group_vars/web.yml
group_vars/db.yml
```

分别给 web 组和 db 组设置变量。

---

## 2.3 host_vars

`host_vars` 用于给某一台具体主机设置变量。

例如：

```text
host_vars/web1.yml
```

只对 `web1` 这台主机生效。

---

## 2.4 when

`when` 用于条件判断。

例如：

```yaml
when: ansible_os_family == "Debian"
```

意思是：只有目标机器是 Debian / Ubuntu 系统时，才执行这个任务。

---

## 2.5 loop

`loop` 用于循环执行任务。

例如安装多个软件包：

```yaml
loop:
  - nginx
  - curl
  - git
```

这样不用重复写多个安装任务。

---

## 2.6 tags

`tags` 用于给任务打标签，方便只执行一部分任务。

例如：

```bash
ansible-playbook site.yml --tags install
ansible-playbook site.yml --tags config
ansible-playbook site.yml --tags service
```

---

## 2.7 ansible-vault

`ansible-vault` 用于加密敏感变量。

例如：

```text
数据库密码
API Token
SSH 密码
云厂商 Access Key
管理员密码
```

这些内容不应该明文写在 Git 仓库中。

---

# 3. 推荐学习路线

建议按 5 天学习：

## 第 1 天：roles

学习目标：

```text
把普通 playbook 拆成标准 role 结构
```

重点掌握：

```text
roles/nginx/tasks/main.yml
roles/nginx/handlers/main.yml
roles/nginx/templates/
roles/nginx/defaults/main.yml
roles/nginx/vars/main.yml
```

---

## 第 2 天：group_vars 和 host_vars

学习目标：

```text
把变量从 playbook 中拆出来
```

重点掌握：

```text
group_vars/web.yml
group_vars/db.yml
group_vars/all/common.yml
host_vars/web1.yml
```

---

## 第 3 天：when 和 loop

学习目标：

```text
让 playbook 根据条件执行，并能批量处理重复任务
```

重点掌握：

```yaml
when: ansible_os_family == "Debian"

loop:
  - nginx
  - curl
  - git
```

---

## 第 4 天：tags

学习目标：

```text
只执行 playbook 的某一部分
```

重点掌握：

```bash
--tags
--skip-tags
--list-tags
--list-tasks
```

---

## 第 5 天：ansible-vault

学习目标：

```text
加密敏感变量，避免密码明文出现在项目中
```

重点掌握：

```bash
ansible-vault create
ansible-vault view
ansible-vault edit
ansible-vault rekey
--ask-vault-pass
--vault-password-file
```

---

# 4. 实验环境准备

本实验可以在 WSL / Ubuntu 中完成。

建议环境：

```text
控制节点：WSL Ubuntu
目标节点：本机 localhost
测试服务：Nginx
测试端口：8088
```

之所以使用 8088 而不是 80，是为了避免和你之前 Docker、Nginx 或其他服务占用 80 端口冲突。

检查 Ansible 是否安装：

```bash
ansible --version
```

如果没有安装：

```bash
sudo apt update
sudo apt install -y ansible
```

---

# 5. 项目目录结构

创建项目目录：

```bash
mkdir -p ~/ansible-advanced-lab
cd ~/ansible-advanced-lab
```

创建 role 目录：

```bash
mkdir -p roles/nginx/{tasks,handlers,templates,defaults,vars}
mkdir -p group_vars/all
mkdir -p host_vars
```

最终目录结构如下：

```text
ansible-advanced-lab/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
│   ├── web.yml
│   ├── db.yml
│   └── all/
│       ├── common.yml
│       └── vault.yml
├── host_vars/
│   └── web1.yml
└── roles/
    └── nginx/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── templates/
        │   └── nginx.conf.j2
        ├── defaults/
        │   └── main.yml
        └── vars/
            └── main.yml
```

每个文件的作用：

```text
ansible.cfg                  Ansible 配置文件
inventory.ini                主机清单
site.yml                     总入口 playbook
group_vars/web.yml           web 组变量
group_vars/db.yml            db 组变量
group_vars/all/common.yml    全局普通变量
group_vars/all/vault.yml     全局加密变量
host_vars/web1.yml           web1 单台主机变量
roles/nginx/tasks/main.yml   Nginx 任务
roles/nginx/handlers/main.yml Nginx handler
roles/nginx/templates/       Nginx 配置模板
roles/nginx/defaults/main.yml 默认变量
roles/nginx/vars/main.yml     role 内部变量
```

---

# 6. 创建 ansible.cfg

创建文件：

```bash
nano ansible.cfg
```

写入：

```ini
[defaults]
inventory = ./inventory.ini
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False
interpreter_python = auto_silent
```

解释：

```text
inventory = ./inventory.ini
指定默认 inventory 文件。

roles_path = ./roles
指定 roles 目录位置。

host_key_checking = False
实验环境关闭 SSH 指纹检查，避免第一次连接时交互确认。

retry_files_enabled = False
不生成 .retry 文件。

interpreter_python = auto_silent
自动识别 Python 解释器。
```

---

# 7. 创建 inventory.ini

创建文件：

```bash
nano inventory.ini
```

写入：

```ini
[web]
web1 ansible_host=127.0.0.1 ansible_connection=local

[db]
db1 ansible_host=127.0.0.1 ansible_connection=local
```

解释：

```text
[web]
表示 web 主机组。

[db]
表示 db 主机组。

web1 / db1
是 Ansible 中定义的主机名。

ansible_host=127.0.0.1
表示目标地址是本机。

ansible_connection=local
表示本机执行，不通过 SSH。
```

测试 inventory：

```bash
ansible all -m ping
```

如果成功，会看到类似：

```text
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

db1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

# 8. 学习 roles

创建入口 playbook：

```bash
nano site.yml
```

写入：

```yaml
---
- name: Deploy Nginx using role
  hosts: web
  become: true
  gather_facts: true

  roles:
    - nginx
```

解释：

```text
name
playbook 的名称。

hosts: web
只对 web 主机组执行。

become: true
使用 sudo 权限。

gather_facts: true
收集目标主机信息，例如 ansible_os_family。

roles:
调用 roles/nginx 这个 role。
```

Role 的执行逻辑：

```text
site.yml
  ↓
roles/nginx/tasks/main.yml
  ↓
如果 task 中有 notify
  ↓
roles/nginx/handlers/main.yml
```

---

# 9. 学习 defaults 与 vars

## 9.1 创建 defaults/main.yml

创建文件：

```bash
nano roles/nginx/defaults/main.yml
```

写入：

```yaml
---
nginx_port: 8088
nginx_server_name: localhost
nginx_root: /var/www/ansible-nginx

nginx_packages:
  - nginx
  - curl
  - git

nginx_index_title: "Ansible Nginx Demo"
nginx_index_message: "Deployed by Ansible Role"
```

`defaults/main.yml` 适合放默认变量。

这些变量可以被 `group_vars` 和 `host_vars` 覆盖。

---

## 9.2 创建 vars/main.yml

创建文件：

```bash
nano roles/nginx/vars/main.yml
```

写入：

```yaml
---
nginx_service_name: nginx
nginx_config_path: /etc/nginx/sites-available/ansible-demo
nginx_enabled_path: /etc/nginx/sites-enabled/ansible-demo
nginx_default_site_path: /etc/nginx/sites-enabled/default
```

`vars/main.yml` 适合放 role 内部相对固定的变量。

---

## 9.3 defaults 与 vars 的区别

简单理解：

```text
defaults/main.yml
默认值，优先级较低，适合给用户覆盖。

vars/main.yml
role 内部变量，优先级较高，通常不希望外部随意覆盖。
```

实际项目建议：

```text
可修改配置        放 defaults/main.yml
环境差异          放 group_vars/
单台机器差异      放 host_vars/
敏感信息          放 ansible-vault
role 内部路径变量  放 vars/main.yml
```

---

# 10. 学习 tasks、when、loop、tags

创建文件：

```bash
nano roles/nginx/tasks/main.yml
```

写入：

```yaml
---
- name: Update apt cache on Debian family
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"
  tags:
    - install

- name: Install required packages with loop
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop: "{{ nginx_packages }}"
  when: ansible_os_family == "Debian"
  tags:
    - install
    - packages

- name: Create nginx root directory
  ansible.builtin.file:
    path: "{{ nginx_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: "0755"
  tags:
    - config

- name: Deploy index page
  ansible.builtin.copy:
    dest: "{{ nginx_root }}/index.html"
    content: |
      <html>
      <head>
        <title>{{ nginx_index_title }}</title>
      </head>
      <body>
        <h1>{{ nginx_index_title }}</h1>
        <p>{{ nginx_index_message }}</p>
        <p>Environment: {{ env_name }}</p>
      </body>
      </html>
    owner: www-data
    group: www-data
    mode: "0644"
  notify: Reload nginx
  tags:
    - config

- name: Deploy nginx config from template
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_path }}"
    mode: "0644"
  notify: Reload nginx
  tags:
    - config

- name: Disable default nginx site
  ansible.builtin.file:
    path: "{{ nginx_default_site_path }}"
    state: absent
  notify: Reload nginx
  tags:
    - config

- name: Enable ansible nginx site
  ansible.builtin.file:
    src: "{{ nginx_config_path }}"
    dest: "{{ nginx_enabled_path }}"
    state: link
    force: true
  notify: Reload nginx
  tags:
    - config

- name: Ensure nginx service is started and enabled
  ansible.builtin.service:
    name: "{{ nginx_service_name }}"
    state: started
    enabled: true
  tags:
    - service
```

---

## 10.1 when 的作用

例如：

```yaml
when: ansible_os_family == "Debian"
```

意思是：

```text
如果目标系统属于 Debian 系列，例如 Ubuntu，就执行这个任务。
如果不是，就跳过这个任务。
```

查看目标主机的系统类型：

```bash
ansible web -m setup -a "filter=ansible_os_family"
```

可能输出：

```json
"ansible_os_family": "Debian"
```

`when` 的价值是让同一个 role 能适配不同环境。

例如以后你可以写：

```yaml
- name: Install packages on RedHat family
  ansible.builtin.dnf:
    name: "{{ nginx_packages }}"
    state: present
  when: ansible_os_family == "RedHat"
```

---

## 10.2 loop 的作用

例如：

```yaml
loop: "{{ nginx_packages }}"
```

而变量是：

```yaml
nginx_packages:
  - nginx
  - curl
  - git
```

实际执行效果等价于：

```text
安装 nginx
安装 curl
安装 git
```

`loop` 适合用于：

```text
批量安装软件包
批量创建目录
批量创建用户
批量复制文件
批量开放端口
```

---

## 10.3 tags 的作用

例如：

```yaml
tags:
  - install
```

运行时可以只执行安装任务：

```bash
ansible-playbook site.yml --tags install
```

常见 tag 设计：

```text
install   安装软件
config    配置文件
service   管理服务
debug     调试任务
deploy    部署任务
```

---

# 11. 学习 handlers

创建文件：

```bash
nano roles/nginx/handlers/main.yml
```

写入：

```yaml
---
- name: Reload nginx
  ansible.builtin.service:
    name: "{{ nginx_service_name }}"
    state: reloaded
```

## 11.1 handler 的触发机制

普通任务中写：

```yaml
notify: Reload nginx
```

表示如果这个任务产生了变化，就触发名为 `Reload nginx` 的 handler。

例如：

```text
Nginx 配置模板变化
  ↓
template 任务状态 changed
  ↓
notify: Reload nginx
  ↓
play 执行后半段统一 reload nginx
```

## 11.2 handler 的优势

如果配置没有变化，handler 不会执行。

这符合自动化运维中的一个重要原则：

```text
没有变化就不重复重启服务。
```

这样可以减少不必要的服务中断。

---

# 12. 学习 templates

创建模板文件：

```bash
nano roles/nginx/templates/nginx.conf.j2
```

写入：

```nginx
server {
    listen {{ nginx_port }};
    server_name {{ nginx_server_name }};

    root {{ nginx_root }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /health {
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }
}
```

## 12.1 模板的作用

模板文件中使用变量：

```nginx
listen {{ nginx_port }};
```

如果变量是：

```yaml
nginx_port: 8088
```

生成的真实配置就是：

```nginx
listen 8088;
```

## 12.2 templates 的使用场景

适合用于：

```text
Nginx 配置
Prometheus 配置
Grafana 配置
Docker Compose 文件
systemd service 文件
应用程序配置文件
```

---

# 13. 学习 group_vars

## 13.1 创建全局变量

创建文件：

```bash
nano group_vars/all/common.yml
```

写入：

```yaml
---
env_name: dev
```

`group_vars/all/` 中的变量会对所有主机生效。

---

## 13.2 创建 web 组变量

创建文件：

```bash
nano group_vars/web.yml
```

写入：

```yaml
---
nginx_port: 8088
nginx_index_title: "Web Group Nginx"
nginx_index_message: "This value comes from group_vars/web.yml"

nginx_packages:
  - nginx
  - curl
  - git
```

说明：

```text
只要主机属于 [web] 组，就可以使用这些变量。
```

也就是说，`web1` 会使用 `group_vars/web.yml` 中的变量。

---

## 13.3 创建 db 组变量

创建文件：

```bash
nano group_vars/db.yml
```

写入：

```yaml
---
db_name: appdb
db_user: appuser
db_password: "{{ vault_db_password }}"
```

虽然本项目暂时不部署数据库，但这个结构是为了后续扩展 PostgreSQL Role 做准备。

---

# 14. 学习 host_vars

创建文件：

```bash
nano host_vars/web1.yml
```

写入：

```yaml
---
nginx_index_message: "This value comes from host_vars/web1.yml and overrides group_vars/web.yml"
```

## 14.1 host_vars 的作用

`group_vars/web.yml` 是 web 组所有机器共用的变量。

`host_vars/web1.yml` 是 web1 单台机器专属变量。

如果两边定义了同一个变量，通常单台主机变量会覆盖组变量。

例如：

```yaml
# group_vars/web.yml
nginx_index_message: "From group_vars"

# host_vars/web1.yml
nginx_index_message: "From host_vars"
```

最终 web1 使用：

```text
From host_vars
```

## 14.2 查看最终变量

执行：

```bash
ansible-inventory --host web1
```

如果使用了 vault，还需要：

```bash
ansible-inventory --host web1 --ask-vault-pass
```

---

# 15. 运行完整 playbook

第一次运行：

```bash
ansible-playbook site.yml --ask-become-pass
```

如果你的用户执行 sudo 不需要密码，也可以：

```bash
ansible-playbook site.yml
```

检查 Nginx 配置：

```bash
sudo nginx -t
```

检查端口：

```bash
ss -tulnp | grep 8088
```

访问测试：

```bash
curl http://localhost:8088
curl http://localhost:8088/health
```

如果成功，`/health` 应该返回：

```text
ok
```

---

# 16. 学习 tags 局部执行

## 16.1 查看所有 tags

```bash
ansible-playbook site.yml --list-tags
```

可能看到：

```text
install
packages
config
service
```

---

## 16.2 查看所有 tasks

```bash
ansible-playbook site.yml --list-tasks
```

---

## 16.3 只执行安装任务

```bash
ansible-playbook site.yml --tags install --ask-become-pass
```

只会执行：

```text
更新 apt 缓存
安装 nginx/curl/git
```

---

## 16.4 只执行配置任务

```bash
ansible-playbook site.yml --tags config --ask-become-pass
```

只会执行：

```text
创建网页目录
生成 index.html
生成 Nginx 配置
启用站点
禁用默认站点
```

---

## 16.5 只执行服务任务

```bash
ansible-playbook site.yml --tags service --ask-become-pass
```

只会执行：

```text
启动 nginx
设置 nginx 开机自启
```

---

## 16.6 跳过安装任务

```bash
ansible-playbook site.yml --skip-tags install --ask-become-pass
```

意思是：

```text
不要执行安装软件包相关任务，只执行其他任务。
```

实际场景：

```text
软件已经安装好了，只想更新配置。
```

这时可以跳过安装任务，提高执行效率。

---

# 17. 学习 ansible-vault

## 17.1 创建加密变量文件

执行：

```bash
ansible-vault create group_vars/all/vault.yml
```

系统会让你输入 Vault 密码。

然后写入：

```yaml
---
vault_db_password: "ChangeMe_123!"
vault_admin_token: "my-secret-token"
```

保存退出后，这个文件会被加密。

直接查看时会变成类似：

```text
$ANSIBLE_VAULT;1.1;AES256
343533633...
```

---

## 17.2 查看加密文件

```bash
ansible-vault view group_vars/all/vault.yml
```

---

## 17.3 编辑加密文件

```bash
ansible-vault edit group_vars/all/vault.yml
```

---

## 17.4 修改 Vault 密码

```bash
ansible-vault rekey group_vars/all/vault.yml
```

---

## 17.5 运行带 Vault 的 playbook

因为项目中引用了：

```yaml
db_password: "{{ vault_db_password }}"
```

所以运行时需要提供 vault 密码：

```bash
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

如果不加 `--ask-vault-pass`，Ansible 会报错：

```text
Attempting to decrypt but no vault secrets found
```

---

## 17.6 使用 Vault 密码文件

实验环境可以创建密码文件：

```bash
nano .vault_pass
```

写入你的 vault 密码。

设置权限：

```bash
chmod 600 .vault_pass
```

运行：

```bash
ansible-playbook site.yml --vault-password-file .vault_pass --ask-become-pass
```

但是必须注意：

```text
.vault_pass 绝对不能上传到 GitHub。
```

创建 `.gitignore`：

```bash
nano .gitignore
```

写入：

```gitignore
.vault_pass
*.retry
```

---

# 18. 综合练习项目

## 18.1 项目目标

使用一条命令完成：

```text
1. 识别目标系统类型
2. 如果是 Debian/Ubuntu，则安装 nginx、curl、git
3. 根据 group_vars/web.yml 设置 Nginx 端口
4. 根据 host_vars/web1.yml 覆盖网页内容
5. 使用 Jinja2 模板生成 Nginx 配置
6. 配置变更后自动 reload nginx
7. 使用 tags 支持局部执行
8. 使用 ansible-vault 加密敏感变量
9. curl localhost:8088/health 返回 ok
```

---

## 18.2 第一遍练习：照着敲，跑通

执行：

```bash
ansible-playbook site.yml --ask-become-pass
```

访问：

```bash
curl http://localhost:8088/health
```

看到：

```text
ok
```

说明成功。

---

## 18.3 第二遍练习：修改变量

修改：

```bash
nano group_vars/web.yml
```

把端口改成：

```yaml
nginx_port: 8090
```

重新执行：

```bash
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

测试：

```bash
curl http://localhost:8090/health
```

---

## 18.4 第三遍练习：只运行部分任务

只更新配置：

```bash
ansible-playbook site.yml --tags config --ask-vault-pass --ask-become-pass
```

只启动服务：

```bash
ansible-playbook site.yml --tags service --ask-vault-pass --ask-become-pass
```

查看任务列表：

```bash
ansible-playbook site.yml --list-tasks
```

查看 tags：

```bash
ansible-playbook site.yml --list-tags
```

---

# 19. 常见问题排查

## 19.1 报错：Missing sudo password

原因：

```text
playbook 中使用了 become: true，但运行时没有提供 sudo 密码。
```

解决：

```bash
ansible-playbook site.yml --ask-become-pass
```

---

## 19.2 报错：Attempting to decrypt but no vault secrets found

原因：

```text
项目中有 ansible-vault 加密文件，但运行时没有提供 vault 密码。
```

解决：

```bash
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

---

## 19.3 curl 访问不到 8088

检查 Nginx 配置：

```bash
sudo nginx -t
```

检查端口：

```bash
ss -tulnp | grep nginx
```

检查服务：

```bash
sudo systemctl status nginx
```

如果 WSL 没有启用 systemd，可以用：

```bash
sudo service nginx status
sudo service nginx restart
```

---

## 19.4 变量没有生效

查看最终变量：

```bash
ansible-inventory --host web1 --ask-vault-pass
```

重点检查：

```text
1. group_vars/web.yml 是否和 inventory 里的 [web] 对应
2. host_vars/web1.yml 是否和 inventory 里的 web1 对应
3. YAML 缩进是否正确
4. 变量名是否写错
5. 是否被更高优先级变量覆盖
```

---

## 19.5 handler 没有执行

原因：

```text
handler 只有在对应 task 状态为 changed 并且有 notify 时才会执行。
```

如果模板没有变化，handler 不执行是正常的。

检查 task 中是否有：

```yaml
notify: Reload nginx
```

---

## 19.6 端口被占用

检查端口：

```bash
ss -tulnp | grep 8088
```

如果被占用，可以修改：

```bash
nano group_vars/web.yml
```

改成：

```yaml
nginx_port: 8090
```

重新执行 playbook。

---

## 19.7 YAML 缩进错误

YAML 对缩进非常敏感。

错误写法：

```yaml
tasks:
- name: install nginx
apt:
name: nginx
```

正确写法：

```yaml
tasks:
  - name: install nginx
    ansible.builtin.apt:
      name: nginx
      state: present
```

建议：

```text
统一使用 2 个空格缩进
不要使用 Tab
```

---

# 20. 核心原理总结

## 20.1 roles 的原理

Role 的本质是：

```text
把一个大的 playbook 拆成标准目录结构。
```

好处：

```text
1. 结构清晰
2. 易于复用
3. 易于维护
4. 适合多人协作
5. 适合企业级自动化项目
```

例如：

```yaml
roles:
  - nginx
  - postgresql
  - docker
```

就可以组合多个功能。

---

## 20.2 group_vars 的原理

`group_vars` 解决的是：

```text
同一组机器共享变量。
```

例如：

```text
web 组：nginx_port = 8088
db 组：db_port = 5432
monitor 组：prometheus_port = 9090
```

这样变量不需要写死在 playbook 中。

---

## 20.3 host_vars 的原理

`host_vars` 解决的是：

```text
单台机器的特殊配置。
```

例如：

```text
web1 使用 8088 端口
web2 使用 8089 端口
web3 使用 8090 端口
```

这些差异适合放在 `host_vars` 中。

---

## 20.4 when 的原理

`when` 解决的是：

```text
根据条件决定是否执行任务。
```

常见使用场景：

```text
不同操作系统执行不同安装命令
只有变量开启时才配置某功能
只有特定主机才执行某任务
```

例如：

```yaml
when: ansible_os_family == "Debian"
```

---

## 20.5 loop 的原理

`loop` 解决的是：

```text
重复执行同类任务。
```

适合用于：

```text
安装多个软件包
创建多个用户
创建多个目录
复制多个配置文件
开放多个防火墙端口
```

---

## 20.6 tags 的原理

`tags` 解决的是：

```text
大型 playbook 中只运行部分任务。
```

在真实项目中非常常用。

例如：

```bash
ansible-playbook site.yml --tags config
```

只执行配置任务，而不重新安装软件。

---

## 20.7 ansible-vault 的原理

`ansible-vault` 解决的是：

```text
敏感变量不能明文存储。
```

适合加密：

```text
数据库密码
API Token
SSH 密码
云厂商 Access Key
管理员密码
```

不适合加密：

```text
端口号
软件包名
普通路径
环境名称
服务名称
```

---

# 21. 面试常见问题与参考答案

## 21.1 Ansible Role 是什么？

Role 是 Ansible 的标准项目组织结构，用来把 tasks、handlers、templates、files、vars、defaults 等内容拆分到固定目录中。

它可以提高 playbook 的复用性、可读性和可维护性。

---

## 21.2 为什么要使用 Role？

因为如果所有任务都写在一个 playbook 中，文件会越来越大，不方便维护。

使用 Role 后，可以把不同功能拆开：

```text
roles/nginx
roles/mysql
roles/docker
roles/prometheus
```

这样每个 role 只负责一个功能模块。

---

## 21.3 group_vars 和 host_vars 有什么区别？

`group_vars` 是给一组主机设置变量。

`host_vars` 是给某一台具体主机设置变量。

例如：

```text
group_vars/web.yml 作用于 web 组所有机器
host_vars/web1.yml 只作用于 web1 这一台机器
```

---

## 21.4 defaults 和 vars 有什么区别？

简单理解：

```text
defaults/main.yml
放默认变量，优先级低，方便外部覆盖。

vars/main.yml
放 role 内部变量，优先级高，通常不希望外部覆盖。
```

实际项目中：

```text
可变配置放 defaults
环境变量放 group_vars
单机变量放 host_vars
敏感变量放 vault
```

---

## 21.5 when 有什么作用？

`when` 用于条件判断。

例如：

```yaml
when: ansible_os_family == "Debian"
```

表示只有目标系统是 Debian 系列时才执行该任务。

常用于适配不同操作系统。

---

## 21.6 loop 有什么作用？

`loop` 用于循环执行任务。

例如安装多个软件包：

```yaml
loop:
  - nginx
  - curl
  - git
```

这样可以避免重复写多个 task。

---

## 21.7 handlers 什么时候执行？

handler 只有在被 `notify` 触发时才执行。

并且通常只有相关任务状态为 `changed` 时才触发。

常用于：

```text
restart nginx
reload nginx
restart docker
restart prometheus
```

---

## 21.8 tags 有什么作用？

tags 可以让我们只运行 playbook 中的一部分任务。

例如：

```bash
ansible-playbook site.yml --tags config
```

只运行配置相关任务。

---

## 21.9 ansible-vault 用来做什么？

`ansible-vault` 用来加密敏感变量，例如密码、token、密钥等。

它可以避免敏感信息明文出现在 Git 仓库中。

---

## 21.10 Ansible 的幂等性是什么？

幂等性是指：

```text
同一个 playbook 执行多次，最终结果一致。
```

例如：

```text
nginx 已经安装，再执行安装任务，不会重复安装。
配置文件没有变化，handler 不会重复 reload。
目录已经存在，不会重复创建。
```

幂等性是 Ansible 非常重要的特性。

---

## 21.11 notify 和 handler 的关系是什么？

`notify` 用于通知 handler。

例如：

```yaml
notify: Reload nginx
```

如果这个任务发生了变化，就会触发名为 `Reload nginx` 的 handler。

---

## 21.12 为什么敏感变量不能直接写在 playbook 里？

因为 playbook 通常会上传到 Git 仓库。

如果密码明文写在里面，可能导致：

```text
数据库密码泄露
API Token 泄露
服务器账号泄露
云服务密钥泄露
```

所以应该使用 `ansible-vault` 加密。

---

# 22. 阶段完成标准

学完本阶段后，你应该能够完成：

```text
1. 能解释 Role 是什么
2. 能创建 roles/nginx 标准目录
3. 能写 tasks/main.yml
4. 能写 handlers/main.yml
5. 能写 templates/nginx.conf.j2
6. 能使用 defaults/main.yml 设置默认变量
7. 能使用 group_vars/web.yml 设置组变量
8. 能使用 host_vars/web1.yml 设置单机变量
9. 能用 when 判断系统类型
10. 能用 loop 安装多个包
11. 能用 tags 局部执行任务
12. 能用 ansible-vault 加密密码
13. 能用一条命令自动部署 Nginx
14. 能排查常见执行错误
```

---

# 23. 下一阶段建议

完成本阶段后，建议进入：

```text
第九阶段：Ansible 综合项目
```

综合项目建议：

```text
一条命令自动部署：

Nginx
Docker
PostgreSQL
Redis
Flask Web 应用
Prometheus node-exporter
```

项目目标：

```text
1. 创建多个 role
2. 使用 group_vars 区分 web/db/monitor
3. 使用 ansible-vault 管理数据库密码
4. 使用 templates 生成配置文件
5. 使用 handlers 自动重载服务
6. 使用 tags 实现局部部署
7. 支持重复执行，保持幂等性
```

建议项目结构：

```text
ansible-final-project/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
│   ├── web.yml
│   ├── db.yml
│   └── all/
│       ├── common.yml
│       └── vault.yml
└── roles/
    ├── nginx/
    ├── docker/
    ├── postgresql/
    ├── redis/
    └── flask_app/
```

这会更接近真实运维自动化项目，也更适合作为简历项目。

---

# 24. 最终记忆口诀

```text
Role 管结构
group_vars 管组
host_vars 管单机
when 管条件
loop 管重复
tags 管局部执行
vault 管密码
handler 管服务重载
template 管配置生成
```

---

# 25. 本阶段推荐命令清单

```bash
# 查看 Ansible 版本
ansible --version

# 测试主机连通性
ansible all -m ping

# 查看系统变量
ansible web -m setup -a "filter=ansible_os_family"

# 查看主机最终变量
ansible-inventory --host web1

# 执行完整 playbook
ansible-playbook site.yml --ask-become-pass

# 带 vault 执行
ansible-playbook site.yml --ask-vault-pass --ask-become-pass

# 查看 tags
ansible-playbook site.yml --list-tags

# 查看 tasks
ansible-playbook site.yml --list-tasks

# 只执行安装任务
ansible-playbook site.yml --tags install --ask-become-pass

# 只执行配置任务
ansible-playbook site.yml --tags config --ask-become-pass

# 跳过安装任务
ansible-playbook site.yml --skip-tags install --ask-become-pass

# 创建 vault 文件
ansible-vault create group_vars/all/vault.yml

# 查看 vault 文件
ansible-vault view group_vars/all/vault.yml

# 编辑 vault 文件
ansible-vault edit group_vars/all/vault.yml

# 修改 vault 密码
ansible-vault rekey group_vars/all/vault.yml

# 检查 Nginx 配置
sudo nginx -t

# 查看端口
ss -tulnp | grep nginx

# 测试服务
curl http://localhost:8088/health
```

---

# 26. 总结

这一阶段的本质是：

```text
从“写一个 playbook”升级到“组织一个真实 Ansible 项目”。
```

你不只是要会写命令，还要学会如何组织结构、管理变量、复用模块、加密密码、按标签执行任务。

这正是 Ansible 从入门到实际运维工作的关键一步。

学完之后，你写出的自动化脚本会更接近企业实际使用方式，也更适合作为简历项目的一部分。
