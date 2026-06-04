# Linux System Administration Stage 1: Advanced Basics with WSL

> Learning environment: Windows + WSL Ubuntu  
> Goal: Build a systematic foundation in Linux system administration, including users, permissions, processes, systemd services, disks, logs, cron jobs, and package management.

---

## 1. Learning Objectives

At this stage, you should move beyond only knowing how to run Docker commands. You should understand how the Linux system itself works.

After completing this stage, you should be able to:

1. Create and manage Linux users.
2. Configure sudo privileges.
3. Understand and modify file permissions.
4. Manage Linux processes.
5. Use systemd to manage services.
6. View service logs with journalctl.
7. Check disk usage and mount points.
8. Create cron scheduled tasks.
9. Install and remove software packages with apt.
10. Troubleshoot common Linux service failures.

---

## 2. Environment Preparation

This guide assumes you are using **Ubuntu inside WSL2 on Windows**.

Open your WSL terminal and run:

```bash
whoami
pwd
cat /etc/os-release
uname -a
```

Explanation:

```bash
whoami
```

Shows the current user.

```bash
pwd
```

Shows the current working directory.

```bash
cat /etc/os-release
```

Shows the Linux distribution information.

```bash
uname -a
```

Shows kernel and system information.

---

## 3. Check Whether systemd Is Available in WSL

Because this stage includes service management with `systemctl`, you need to confirm that systemd is enabled.

Run:

```bash
ps -p 1 -o comm=
```

If the output is:

```bash
systemd
```

then systemd is enabled.

If the output is not `systemd`, edit the WSL configuration file:

```bash
sudo nano /etc/wsl.conf
```

Add the following content:

```ini
[boot]
systemd=true
```

Save and exit nano:

```text
Ctrl + O
Enter
Ctrl + X
```

Then open **Windows PowerShell** and run:

```powershell
wsl --shutdown
```

Reopen WSL and check again:

```bash
ps -p 1 -o comm=
```

The expected output is:

```bash
systemd
```

---

## 4. User and Permission Management

### 4.1 Check Current User Information

Run:

```bash
whoami
id
groups
```

Explanation:

```bash
whoami
```

Displays the current username.

```bash
id
```

Displays the current user's UID, GID, and group membership.

```bash
groups
```

Displays the groups that the current user belongs to.

A typical output may look like:

```text
uid=1000(username) gid=1000(username) groups=1000(username),27(sudo)
```

The important part is whether the user belongs to the `sudo` group.

---

### 4.2 Create a New User named deploy

Run:

```bash
sudo useradd -m -s /bin/bash deploy
```

Explanation:

```bash
sudo
```

Runs the command with administrator privileges.

```bash
useradd
```

Creates a new user.

```bash
-m
```

Creates the home directory automatically.

```bash
-s /bin/bash
```

Sets the user's default shell to Bash.

Check whether the user was created:

```bash
id deploy
ls -ld /home/deploy
```

---

### 4.3 Set a Password for deploy

Run:

```bash
sudo passwd deploy
```

Enter the password twice.

In Linux, passwords are not displayed while typing. This is normal.

---

### 4.4 Switch to the deploy User

Run:

```bash
su - deploy
```

Then check:

```bash
whoami
pwd
```

Expected output:

```text
deploy
/home/deploy
```

Exit the deploy user:

```bash
exit
```

---

### 4.5 Configure sudo Privileges for deploy

Run:

```bash
sudo usermod -aG sudo deploy
```

Explanation:

```bash
usermod
```

Modifies a user account.

```bash
-aG sudo
```

Adds the user to the sudo group without removing existing groups.

Check:

```bash
groups deploy
```

Expected output:

```text
deploy : deploy sudo
```

Test sudo privileges:

```bash
su - deploy
sudo whoami
```

Expected output:

```text
root
```

Exit:

```bash
exit
```

---

## 5. File Permission Management

### 5.1 Create the Application Directory

Run:

```bash
sudo mkdir -p /opt/myapp
```

Check the directory:

```bash
ls -ld /opt/myapp
```

You may see:

```text
drwxr-xr-x 2 root root 4096 ...
```

This means the directory is owned by the `root` user and the `root` group.

---

### 5.2 Change Ownership to deploy

Run:

```bash
sudo chown -R deploy:deploy /opt/myapp
```

Explanation:

```bash
chown
```

Changes file or directory ownership.

```bash
-R
```

Applies changes recursively.

```bash
deploy:deploy
```

The first `deploy` is the user, and the second `deploy` is the group.

Check:

```bash
ls -ld /opt/myapp
```

Expected output:

```text
drwxr-xr-x 2 deploy deploy 4096 ...
```

---

### 5.3 Practice chmod

Create a test file:

```bash
sudo -u deploy touch /opt/myapp/test.txt
ls -l /opt/myapp/test.txt
```

You may see:

```text
-rw-r--r-- 1 deploy deploy 0 ...
```

The permission string can be divided into three parts:

```text
rw-   r--   r--
user  group others
```

Permission meanings:

```text
r = read
w = write
x = execute
```

Numeric permissions:

```text
r = 4
w = 2
x = 1
```

Examples:

```text
7 = 4 + 2 + 1 = rwx
6 = 4 + 2 = rw-
5 = 4 + 1 = r-x
4 = r--
```

Change the file permission:

```bash
chmod 600 /opt/myapp/test.txt
ls -l /opt/myapp/test.txt
```

Expected output:

```text
-rw------- 1 deploy deploy ...
```

This means only the owner can read and write the file.

Change it back to a common file permission:

```bash
chmod 644 /opt/myapp/test.txt
```

---

## 6. Create a Simple Python Web Service

### 6.1 Switch to deploy

```bash
su - deploy
cd /opt/myapp
```

---

### 6.2 Create app.py

Run:

```bash
nano app.py
```

Add the following code:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import os
import time

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        data = {
            "message": "hello from myapp",
            "path": self.path,
            "pid": os.getpid(),
            "time": time.strftime("%Y-%m-%d %H:%M:%S")
        }

        body = json.dumps(data, ensure_ascii=False).encode("utf-8")

        self.send_response(200)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

server = HTTPServer(("127.0.0.1", 8000), Handler)
print("myapp running at http://127.0.0.1:8000")
server.serve_forever()
```

Save and exit nano:

```text
Ctrl + O
Enter
Ctrl + X
```

---

### 6.3 Run the Service Manually

Run:

```bash
python3 app.py
```

Expected output:

```text
myapp running at http://127.0.0.1:8000
```

Open another WSL terminal and test:

```bash
curl http://127.0.0.1:8000
```

Expected output:

```json
{"message": "hello from myapp", "path": "/", "pid": 1234, "time": "2026-06-04 09:30:00"}
```

Test another path:

```bash
curl http://127.0.0.1:8000/test
```

The response should include:

```json
"path": "/test"
```

Stop the service manually:

```text
Ctrl + C
```

Exit the deploy user:

```bash
exit
```

---

## 7. Process Management

### 7.1 View Processes

Start the Python service again as deploy:

```bash
su - deploy
cd /opt/myapp
python3 app.py
```

Open another WSL terminal and run:

```bash
ps aux | grep app.py
```

You may see:

```text
deploy    1234  0.0  ... python3 app.py
```

Explanation:

```bash
ps aux
```

Shows all system processes.

```bash
grep app.py
```

Filters lines containing `app.py`.

---

### 7.2 Kill a Process

If the PID is `1234`, run:

```bash
kill 1234
```

If normal termination fails:

```bash
kill -9 1234
```

Difference:

```bash
kill PID
```

Requests the process to exit normally.

```bash
kill -9 PID
```

Forces the process to terminate immediately.

In production, use normal `kill` first. Do not use `kill -9` unless necessary.

---

### 7.3 Use top and htop

Run:

```bash
top
```

Press `q` to exit.

Install htop:

```bash
sudo apt update
sudo apt install -y htop
```

Run:

```bash
htop
```

Exit with `F10` or `q`.

---

## 8. Manage the Python Service with systemd

Now you will turn the Python program into a real Linux service.

### 8.1 Create a systemd Service File

Run:

```bash
sudo nano /etc/systemd/system/myapp.service
```

Add:

```ini
[Unit]
Description=My Python Web App
After=network.target

[Service]
User=deploy
Group=deploy
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Explanation:

```ini
[Unit]
```

Defines basic service metadata and startup order.

```ini
Description=My Python Web App
```

Describes the service.

```ini
After=network.target
```

Starts the service after the network target.

```ini
[Service]
```

Defines how the service runs.

```ini
User=deploy
Group=deploy
```

Runs the service as the deploy user instead of root.

This follows an important security principle: services should not run as root unless necessary.

```ini
WorkingDirectory=/opt/myapp
```

Sets the working directory.

```ini
ExecStart=/usr/bin/python3 /opt/myapp/app.py
```

Defines the actual command used to start the service.

```ini
Restart=always
```

Restarts the service automatically if it crashes.

```ini
RestartSec=3
```

Waits 3 seconds before restarting.

```ini
[Install]
WantedBy=multi-user.target
```

Allows the service to be enabled for automatic startup.

---

### 8.2 Reload systemd

Whenever you create or modify a service file, run:

```bash
sudo systemctl daemon-reload
```

---

### 8.3 Start the Service

```bash
sudo systemctl start myapp
```

Check status:

```bash
systemctl status myapp
```

If it is working, you should see:

```text
active (running)
```

Test:

```bash
curl http://127.0.0.1:8000
```

---

### 8.4 Enable Automatic Startup

Run:

```bash
sudo systemctl enable myapp
```

Check:

```bash
systemctl is-enabled myapp
```

Expected output:

```text
enabled
```

Note: In WSL, automatic startup is not exactly the same as on a real Linux server. A real server starts systemd during boot. WSL starts systemd when the distribution is launched.

---

### 8.5 Common systemctl Commands

Stop service:

```bash
sudo systemctl stop myapp
```

Start service:

```bash
sudo systemctl start myapp
```

Restart service:

```bash
sudo systemctl restart myapp
```

View status:

```bash
systemctl status myapp
```

Check whether the service is active:

```bash
systemctl is-active myapp
```

Check whether the service is enabled:

```bash
systemctl is-enabled myapp
```

---

## 9. View Logs with journalctl

### 9.1 Basic Log Commands

View all logs for the service:

```bash
journalctl -u myapp
```

View the latest 50 lines:

```bash
journalctl -u myapp -n 50
```

Follow logs in real time:

```bash
journalctl -u myapp -f
```

Exit real-time log mode:

```text
Ctrl + C
```

View today's logs:

```bash
journalctl -u myapp --since today
```

View logs from the last 10 minutes:

```bash
journalctl -u myapp --since "10 minutes ago"
```

---

### 9.2 Make Python Logs Appear Immediately

Python's `print()` output may be buffered. To disable buffering, edit the service file:

```bash
sudo nano /etc/systemd/system/myapp.service
```

Change:

```ini
ExecStart=/usr/bin/python3 /opt/myapp/app.py
```

to:

```ini
ExecStart=/usr/bin/python3 -u /opt/myapp/app.py
```

Then run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
journalctl -u myapp -f
```

In another terminal, access the service:

```bash
curl http://127.0.0.1:8000/test
```

Observe the log output.

---

## 10. Troubleshooting Practice

Troubleshooting is one of the most important skills in system administration.

---

### 10.1 Error 1: Wrong Python File Path

Edit the service file:

```bash
sudo nano /etc/systemd/system/myapp.service
```

Change:

```ini
ExecStart=/usr/bin/python3 -u /opt/myapp/app.py
```

to:

```ini
ExecStart=/usr/bin/python3 -u /opt/myapp/app_wrong.py
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl status myapp
```

The service should fail.

Check logs:

```bash
journalctl -u myapp -n 50
```

You should see an error similar to:

```text
can't open file '/opt/myapp/app_wrong.py'
```

This means the file path in the systemd service is wrong.

Fix the service file and restore:

```ini
ExecStart=/usr/bin/python3 -u /opt/myapp/app.py
```

Then run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl status myapp
```

---

### 10.2 Error 2: Python Syntax Error

Edit the Python file:

```bash
sudo -u deploy nano /opt/myapp/app.py
```

Add an invalid line at the end:

```python
this is wrong
```

Restart the service:

```bash
sudo systemctl restart myapp
systemctl status myapp
```

Check logs:

```bash
journalctl -u myapp -n 50
```

You should see a Python error such as:

```text
SyntaxError
```

Fix the file by removing the invalid line:

```bash
sudo -u deploy nano /opt/myapp/app.py
```

Restart and test:

```bash
sudo systemctl restart myapp
systemctl status myapp
curl http://127.0.0.1:8000
```

---

### 10.3 Error 3: Port Already in Use

Stop the service:

```bash
sudo systemctl stop myapp
```

Manually start the Python app to occupy port 8000:

```bash
cd /opt/myapp
python3 app.py
```

Open another WSL terminal and run:

```bash
sudo systemctl start myapp
systemctl status myapp
journalctl -u myapp -n 50
```

You may see:

```text
Address already in use
```

This means port 8000 is already occupied.

Find the process using the port:

```bash
sudo ss -lntp | grep 8000
```

Or:

```bash
sudo lsof -i :8000
```

If `lsof` is not installed:

```bash
sudo apt install -y lsof
```

Stop the manually started Python program with:

```text
Ctrl + C
```

Then restart the systemd service:

```bash
sudo systemctl start myapp
systemctl status myapp
```

---

## 11. Disk and Mount Management

### 11.1 Check Disk Usage

Run:

```bash
df -h
```

Important columns:

```text
Filesystem
Size
Used
Avail
Mounted on
```

Explanation:

```bash
df -h
```

Shows disk usage of file systems in human-readable format.

---

### 11.2 Check Directory Size

Check `/opt/myapp`:

```bash
du -sh /opt/myapp
```

Check all directories under `/opt`:

```bash
sudo du -sh /opt/* 2>/dev/null
```

Explanation:

```bash
du
```

Shows disk usage of files and directories.

```bash
-s
```

Shows only the summary.

```bash
-h
```

Uses human-readable units.

---

### 11.3 View Mount Points

Run:

```bash
mount | head
```

Or:

```bash
findmnt
```

In WSL, Windows disks are usually mounted under:

```text
/mnt/c
/mnt/d
```

For example:

```bash
cd /mnt/c
ls
```

Recommendation:

For Linux projects, prefer Linux paths such as:

```text
/home/yourname
/opt/myapp
```

Avoid putting long-term Linux projects under:

```text
/mnt/c/Users/...
```

because Windows and Linux file systems have different permission and performance behavior.

---

## 12. Scheduled Tasks with crontab

### 12.1 Create a Simple Cron Job

Switch to deploy:

```bash
su - deploy
```

Edit cron jobs:

```bash
crontab -e
```

If prompted to select an editor, choose nano.

Add:

```cron
* * * * * echo "$(date) myapp cron test" >> /home/deploy/cron.log
```

Explanation:

```cron
* * * * *
```

Means the command runs every minute.

The five fields are:

```text
minute hour day-of-month month day-of-week
```

Wait 1 to 2 minutes, then check:

```bash
cat /home/deploy/cron.log
```

List current cron jobs:

```bash
crontab -l
```

Remove all cron jobs for the current user:

```bash
crontab -r
```

Be careful: `crontab -r` deletes all cron jobs immediately.

Exit deploy:

```bash
exit
```

---

## 13. Package Management with apt

### 13.1 Update Package Index

```bash
sudo apt update
```

`apt update` updates the local package index. It does not upgrade installed software.

---

### 13.2 Install Common Tools

```bash
sudo apt install -y curl vim nano htop lsof tree net-tools
```

Tool explanations:

```text
curl       Test HTTP requests.
vim/nano   Text editors.
htop       View system resource usage interactively.
lsof       Check open files and port usage.
tree       Display directory structure as a tree.
net-tools  Provides traditional network tools such as ifconfig.
```

---

### 13.3 Check Whether a Package Is Installed

```bash
dpkg -l | grep htop
```

Check command paths:

```bash
which htop
which python3
which systemctl
```

---

### 13.4 Remove Software

Remove a package:

```bash
sudo apt remove -y tree
```

Remove package and configuration files:

```bash
sudo apt purge -y tree
```

Remove unused dependencies:

```bash
sudo apt autoremove -y
```

---

## 14. Complete Practice Workflow

You can complete the entire stage using the following workflow.

```bash
# 1. Check system
whoami
id
cat /etc/os-release
ps -p 1 -o comm=

# 2. Create user
sudo useradd -m -s /bin/bash deploy
sudo passwd deploy
sudo usermod -aG sudo deploy
groups deploy

# 3. Create project directory
sudo mkdir -p /opt/myapp
sudo chown -R deploy:deploy /opt/myapp
ls -ld /opt/myapp

# 4. Write Python service
su - deploy
cd /opt/myapp
nano app.py
python3 app.py

# 5. Test access
curl http://127.0.0.1:8000

# 6. Create systemd service
exit
sudo nano /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl start myapp
systemctl status myapp

# 7. Enable auto-start
sudo systemctl enable myapp
systemctl is-enabled myapp

# 8. View logs
journalctl -u myapp -n 50
journalctl -u myapp -f

# 9. Troubleshoot
sudo systemctl restart myapp
systemctl status myapp
journalctl -u myapp -n 50

# 10. Check processes and ports
ps aux | grep app.py
sudo ss -lntp | grep 8000
top
htop

# 11. Check disk usage
df -h
du -sh /opt/myapp
findmnt

# 12. Cron practice
su - deploy
crontab -e
crontab -l
```

---

## 15. Key Concepts You Must Understand

### 15.1 Linux Is More Than Docker

Docker containers ultimately run on a Linux system.

You need to understand:

```text
Who is the user?
Does the user have enough permissions?
Does the process exist?
Is the port occupied?
Is the service running?
Where are the logs?
Is the disk full?
Is the scheduled task running?
Is the required software installed?
```

These are core Linux system administration skills.

---

### 15.2 systemd Is the Core of Linux Service Management

On real servers, services such as Nginx, MySQL, Redis, and Docker are often managed by systemd.

Common commands:

```bash
systemctl status nginx
systemctl restart docker
systemctl enable mysql
journalctl -u nginx
```

The `myapp.service` you created is a simplified production-style Linux service.

---

### 15.3 Permission Issues Are Very Common

You must be able to understand:

```bash
ls -l
```

Example:

```text
-rw-r--r-- 1 deploy deploy app.py
```

You should know:

```text
Who owns this file?
Who can read it?
Who can write it?
Who can execute it?
```

Many service failures are caused by permission problems rather than code problems.

---

### 15.4 Logs Are the First Place to Look

When a service fails, do not guess randomly.

Use this troubleshooting sequence:

```bash
systemctl status service_name
journalctl -u service_name -n 50
ps aux | grep service_name
ss -lntp | grep port
df -h
```

---

## 16. Additional Practice Tasks

### Practice 1: Change the Port

In `app.py`, change:

```python
server = HTTPServer(("127.0.0.1", 8000), Handler)
```

to:

```python
server = HTTPServer(("127.0.0.1", 9000), Handler)
```

Then run:

```bash
sudo systemctl restart myapp
curl http://127.0.0.1:9000
sudo ss -lntp | grep 9000
```

---

### Practice 2: Change the Service User

In the service file, change:

```ini
User=deploy
Group=deploy
```

to your current WSL user.

Restart the service and check:

```bash
sudo systemctl restart myapp
ps aux | grep app.py
```

After testing, change it back to `deploy`.

---

### Practice 3: Create a Permission Error

Run:

```bash
sudo chown -R root:root /opt/myapp
```

Restart the service:

```bash
sudo systemctl restart myapp
systemctl status myapp
journalctl -u myapp -n 50
```

Observe whether there are permission errors.

Fix it:

```bash
sudo chown -R deploy:deploy /opt/myapp
sudo systemctl restart myapp
```

---

### Practice 4: Create a Health Check Script

Create the script:

```bash
sudo -u deploy nano /opt/myapp/health_check.sh
```

Add:

```bash
#!/bin/bash

if curl -s http://127.0.0.1:8000 >/dev/null; then
    echo "$(date) myapp is OK" >> /home/deploy/myapp_health.log
else
    echo "$(date) myapp is DOWN" >> /home/deploy/myapp_health.log
fi
```

Add execute permission:

```bash
sudo chmod +x /opt/myapp/health_check.sh
```

Create a cron job as deploy:

```bash
su - deploy
crontab -e
```

Add:

```cron
* * * * * /opt/myapp/health_check.sh
```

Check the log:

```bash
cat /home/deploy/myapp_health.log
```

This is similar to a real service health check used in operations work.

---

## 17. Completion Standard

After finishing this stage, you should be able to independently:

1. Create Linux users.
2. Configure sudo privileges.
3. Understand `rwx` file permissions.
4. Modify file ownership and permissions.
5. View processes.
6. Kill abnormal processes.
7. Create a systemd service.
8. Start, stop, and restart services.
9. Enable services at startup.
10. View logs with journalctl.
11. Troubleshoot service startup failures.
12. Check disk space.
13. Understand WSL mount points such as `/mnt/c`.
14. Create cron scheduled tasks.
15. Install and remove software packages with apt.

---

## 18. Common Interview Questions

### 18.1 What is the difference between root and a normal user?

`root` is the superuser with the highest privileges on the system. A normal user has limited permissions and usually needs `sudo` to perform administrative operations.

---

### 18.2 What is the difference between chmod 755 and chmod 644?

`755` means:

```text
owner: read, write, execute
group: read, execute
others: read, execute
```

It is commonly used for directories and executable scripts.

`644` means:

```text
owner: read, write
group: read
others: read
```

It is commonly used for regular text files.

---

### 18.3 What does chown do?

`chown` changes the owner and group of a file or directory.

Example:

```bash
sudo chown -R deploy:deploy /opt/myapp
```

This changes the owner and group of `/opt/myapp` and all its contents to `deploy`.

---

### 18.4 How do you troubleshoot a failed systemd service?

Use this sequence:

```bash
systemctl status service_name
journalctl -u service_name -n 50
check configuration files
check file paths
check permissions
check port usage
check disk space
```

Do not only look at `failed`. The real reason is usually in `journalctl`.

---

### 18.5 What does journalctl -u myapp -f mean?

It follows logs for the `myapp` service in real time.

```bash
-u myapp
```

Specifies the service.

```bash
-f
```

Follows new log output continuously.

---

### 18.6 What is the difference between ps aux and top?

`ps aux` shows a snapshot of current processes.

`top` shows dynamic real-time process and resource usage.

---

### 18.7 What is the difference between kill and kill -9?

`kill PID` sends a normal termination signal and allows the process to exit cleanly.

`kill -9 PID` forcefully terminates the process immediately.

In production, avoid using `kill -9` unless necessary.

---

### 18.8 What is the difference between df -h and du -sh?

`df -h` shows disk usage of file systems.

`du -sh directory` shows the actual disk usage of a specific directory.

---

### 18.9 What is the difference between WSL and a real Linux server?

WSL is a Linux-compatible environment running on Windows. It is suitable for learning commands, scripting, service management, and development workflows.

A real Linux server has a complete boot process, independent networking, disk partitions, firewall rules, security policies, and remote access configuration.

WSL is suitable for your current Linux foundation learning, but later you should also practice on a virtual machine or cloud server.

---

## 19. Suggested Next Stage

After completing this stage, move to:

```text
Stage 2: Linux Networking and Remote Management
```

Recommended topics:

```text
IP addresses
ports
DNS
ping
curl
ss
netstat
ssh
scp
firewall
Nginx reverse proxy
```

You have now learned how to manage a local Linux service. The next step is to learn how other machines access your service, how services communicate, and how to troubleshoot network problems.
