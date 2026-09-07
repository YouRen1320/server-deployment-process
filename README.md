# 阿里云服务器 Nginx + HTTPS + Docker (NestJS) 部署全流程指南

> [!WARNING]
> **历史部署笔记，已停止维护，请勿直接用于新生产环境。** 文中开放应用端口、手工管理证书、直接使用 root 及强制结束 Nginx 等做法不符合当前推荐的最小权限和可回滚运维方式。较新的安全部署示例请查看 [ai-gateway-kit](https://github.com/YouRen1320/ai-gateway-kit)。本文仅保留用于追溯当时的学习过程。

**文档日期**: 2025-12-30
**项目环境**:

- 服务器: 阿里云轻量应用服务器 (CentOS)
- 后端: NestJS (运行在 Docker 容器, 端口 3000)
- 网关: Nginx (运行在宿主机)
- 域名: iyouren.top

---

## 第一阶段：域名与基础设施准备

### 1. 域名解析 (DNS)

登录阿里云控制台 -> 云解析 DNS，添加两条 **A 记录**：

- **主机记录**: `@`  -> **记录值**: 服务器公网 IP
- **主机记录**: `www` -> **记录值**: 服务器公网 IP

### 2. 服务器防火墙设置

登录轻量应用服务器控制台 -> 防火墙 -> 添加规则：

- 放行端口: **80** (HTTP)
- 放行端口: **443** (HTTPS)
- 放行端口: **3000** (如果你需要直接测试 IP 访问)

---

## 第二阶段：服务器环境安装

### 1. 登录服务器

使用 SSH 终端登录服务器。

### 2. 安装 Nginx

由于服务器是纯净环境或只装了 Docker，需要手动安装 Nginx。

> **注意**: 确保你的 NestJS应用 Docker 容器已经启动并映射了 3000 端口。
> 例如：`docker run -d --name nest-app -p 3000:3000 <your-image-name>`

```bash
# 安装 Nginx
sudo yum install nginx -y

# 设置开机自启
sudo systemctl enable nginx

# 启动 Nginx
sudo systemctl start nginx

第三阶段：SSL 证书配置
1. 准备证书文件
在阿里云 SSL 控制台下载 Nginx 格式的证书，解压后获得：

xxx.pem (公钥)

xxx.key (私钥)

2. 创建存放目录
Bash

sudo mkdir -p /etc/nginx/cert
3. 写入证书文件
这里可以使用 `scp` 命令从本地上传，或者使用 vi 编辑器直接粘贴内容。

**方法 A: 使用 SCP 上传 (推荐，更安全)**
在本地电脑终端执行：
```bash
scp -r /path/to/your/cert root@<服务器IP>:/etc/nginx/
```

**方法 B: 直接粘贴 (适合无本地环境)**

写入公钥 (.pem):

Bash

sudo vi /etc/nginx/cert/server.pem
操作：按 i 进入编辑模式 -> 粘贴 .pem 文件内容 -> 按 Esc -> 输入 :wq 保存退出。

写入私钥 (.key):

Bash

sudo vi /etc/nginx/cert/server.key
操作：按 i 进入编辑模式 -> 粘贴 .key 文件内容 -> 按 Esc -> 输入 :wq 保存退出。

第四阶段：Nginx 反向代理配置 (核心)
我们需要修改 Nginx 配置，使其监听 443 端口，并将流量转发给本地的 Docker 容器 (端口 3000)。

1. 编辑配置文件
Bash

sudo vi /etc/nginx/nginx.conf
2. 配置文件内容
找到 http { ... } 块，在其中添加或修改 server 块如下：

Nginx

    # HTTPS Server
    server {
        listen       443 ssl;
        server_name  iyouren.top www.iyouren.top;

        # 证书路径
        ssl_certificate "/etc/nginx/cert/server.pem";
        ssl_certificate_key "/etc/nginx/cert/server.key";

        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # 反向代理设置 (转发给 NestJS)
        location / {
            proxy_pass http://127.0.0.1:3000; # 对应 Docker 容器映射的端口
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    # HTTP 自动跳转 HTTPS (可选)
    server {
        listen 80;
        server_name iyouren.top www.iyouren.top;
        return 301 https://$host$request_uri;
    }
第五阶段：验证与重启

1. 检查配置语法
每次修改配置后必须执行：

Bash

sudo nginx -t
必须看到 syntax is ok 和 test is successful 才能继续。

1. 重启 Nginx
Bash

sudo systemctl restart nginx

```

### 3. (重要) CentOS SELinux 设置
如果 Nginx 启动正常但访问报错 `502 Bad Gateway`，且 Docker 容器确实在运行，可能是 SELinux 拦截了 Nginx 的网络连接。
执行以下命令开启权限：
```bash
sudo setsebool -P httpd_can_network_connect 1
```

### 4. (可选) 解决端口占用问题

如果重启失败提示 Address already in use，说明有残留进程，强制关闭后再启动：

```bash
sudo killall nginx
sudo systemctl start nginx
```

第六阶段：最终验证
打开浏览器访问：

<https://iyouren.top/api>

预期结果:

浏览器地址栏显示安全锁头（HTTPS 生效）。

页面显示 JSON 数据（如 {"message":"..."}），证明 Nginx 成功连接到了 Docker 里的 NestJS。

功能,命令
编辑配置,sudo vi /etc/nginx/nginx.conf
检查配置,sudo nginx -t
重启 Nginx,sudo systemctl restart nginx
查看 Docker 容器,sudo docker ps
查看 Nginx 状态,sudo systemctl status nginx
...
