---
title: "后端部署第二步：本地到公网——如何使用 Nginx 发布 FastAPI 服务"
date: 2025-09-05T18:40:15+08:00
categories:
  - 开发
  - 运维
tags:
  - FastAPI
  - Nginx
  - 部署 
  - Python后端 
  - Web服务配置
---

## 后端部署第二步：本地到公网——如何使用 Nginx 发布 FastAPI 服务

在当今的开发环境中，快速构建和部署后端服务变得至关重要。FastAPI 作为一个高性能、现代化的 Python 异步 Web 框架，广受开发者喜爱。而 Nginx 则是部署 Web 应用最常见也是最稳定的解决方案之一。

本文将手把手带你完成如下目标：

- 启动一个最小 FastAPI 应用；
- 使用 Nginx 把服务代理到 80 端口；
- 实现通过公网 IP 或域名访问服务。

不需要复杂的设置，只需几个步骤，就能让你的本地服务走向世界。

---

### 前提条件

请确保你已经具备以下环境：

- 一台可以访问公网的 Linux 服务器或云主机（如 AWS EC2、腾讯云、阿里云等）；
- 已安装 Python 3.7+；
- 已安装 Nginx；
- 有一些基本的 Linux 操作经验。

如果你准备好了，就让我们开始吧！

---

### 第一步：创建并启动一个最小 FastAPI 后端服务

首先，我们搭建一个最小可运行的 FastAPI 应用。建议使用 Python 虚拟环境：

```bash
# 安装依赖
sudo apt update
sudo apt install python3-pip -y
pip3 install fastapi uvicorn
```

接着，创建一个名为 `main.py` 的文件，内容如下：

```python
# main.py

from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello from FastAPI via Nginx!"}
```

使用 Uvicorn 启动服务：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

> 此时你已经可以通过 `http://<你的服务器IP>:8000` 在浏览器中访问这个返回 JSON 的接口。

---

### 第二步：安装并配置 Nginx

如果还未安装 Nginx，请先执行以下命令安装：

```bash
sudo apt install nginx -y
```

确保 Nginx 正常运行：

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

### 第三步：配置 Nginx 作反向代理

我们需要编辑 Nginx 配置文件，将来自公网 80 端口的请求转发到本地的 8000 端口。

新建一个配置文件（或编辑默认配置）：

```bash
sudo nano /etc/nginx/sites-available/fastapi
```

请输入以下内容：

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

替换 `your_domain_or_ip` 为你的实际公网 IP（如 `123.456.78.9`）或绑定的域名（如 `api.example.com`）。

接下来，启用该配置：

```bash
sudo ln -s /etc/nginx/sites-available/fastapi /etc/nginx/sites-enabled/
sudo nginx -t  # 检查配置是否正确
sudo systemctl reload nginx
```

---

### 第四步：在防火墙中开放 80 端口（如有）

如果你启用了防火墙（如 UFW），确保端口 80 是开放的：

```bash
sudo ufw allow 80
```

---

### 第五步：测试访问

现在，一切已准备就绪：

- 浏览器访问：http://your_domain_or_ip/
  - 示例：http://123.456.78.9/
- 你应能看到如下 JSON 输出：

```json
{"message": "Hello from FastAPI via Nginx!"}
```

恭喜你！你的 FastAPI 服务已经可以通过公网访问了！

---

### 第六步：使用 systemd 管理 FastAPI 服务（推荐）

虽然我们可以用 uvicorn 手动在终端启动 FastAPI 服务，但这种方式有两个明显的问题：

1. 当 SSH 会话断开时，uvicorn 会关闭；
2. 服务无法随系统开机自动启动；

为了让 FastAPI 稳定运行在后台并具备开机自启功能，我们推荐使用 systemd 进行进程管理。

接下来，我们将创建一个 systemd 单元文件来管理 FastAPI 服务。

---

#### 6.1 创建 FastAPI systemd 服务文件

假设你的主机上 FastAPI 项目的路径为 `/home/ubuntu/fastapi-app`，并在该目录下有一个 main.py 文件。

以下步骤以 ubuntu 用户为例：

1. 编辑 systemd 服务文件：

```bash
sudo nano /etc/systemd/system/fastapi.service
```

2. 在文件中填写以下内容：

```ini
[Unit]
Description=FastAPI Application with Uvicorn
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/fastapi-app
ExecStart=/usr/bin/python3 -m uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=3
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

请根据你的实际路径和运行用户进行调整：

- WorkingDirectory 指向你的 FastAPI 应用目录；
- ExecStart 中使用的 python3 和运行参数视你的环境而定；
- 如果你使用虚拟环境，请指定对应虚拟环境下的 python 路径，比如：

```bash
ExecStart=/home/ubuntu/fastapi-venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000
```

---

#### 6.2 启用并启动服务

创建完文件后，执行以下命令：

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable fastapi
sudo systemctl start fastapi
```

现在你的服务已经注册为系统服务并自动启动了！

---

#### 6.3 查看服务状态

使用以下命令查看运行状态：

```bash
sudo systemctl status fastapi
```

你应看到如下类似输出：

```
● fastapi.service - FastAPI Application with Uvicorn
   Loaded: loaded (/etc/systemd/system/fastapi.service; enabled)
   Active: active (running) since ...
   ...
```

---

#### 6.4 查看输出日志（可选）

你可以使用 journalctl 查看程序输出日志：

```bash
sudo journalctl -u fastapi -f
```

这将实时跟踪 FastAPI 服务的日志输出，方便调试请求和错误信息。

---

#### 6.5 重启与停止服务

重启服务：

```bash
sudo systemctl restart fastapi
```

停止服务：

```bash
sudo systemctl stop fastapi
```

---

### 为什么使用 systemd？

使用 systemd 管理服务的几大益处：

- 稳定：系统守护进程会在服务崩溃时自动重启；
- 安全：服务以特定用户运行，避免 root 权限；
- 开机自启：确保服务器重启后服务仍能使用；
- 日志集中：方便使用 journalctl 跟踪日志和排错。

这是部署生产环境服务最推荐的方式之一。

借助 systemd 配置，FastAPI 服务变成了一个真正的系统级网络服务。你无需手动启动，也不怕 SSH 会话断开。真正实现了后端“后台运行、永不掉线”的目标。

---

### 常见问题排查

- **无法访问服务？**
  - 检查是否正在运行 `uvicorn`；
  - Nginx 是否启用正确配置并重新加载；
  - 防火墙是否允许 80 端口；
  - 是否将域名正确地解析到了服务器 IP。

---

### 结语：让服务走得更远

通过 Nginx 将本地服务转发到公网不仅提升了系统稳定性和安全性，同时也为后续做 HTTPS（SSL）配置、负载均衡等部署操作打下了坚实基础。

今天，你不仅写了一个 FastAPI app，更为自己的开发部署能力迈出了重要一步。

后续你可以探索：

- 使用 `systemd` 管理 FastAPI 服务；
- 接入 Let's Encrypt 实现 HTTPS；
- 使用 Docker 容器进一步标准化部署流程。