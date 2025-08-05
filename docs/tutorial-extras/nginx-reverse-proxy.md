# 云服务器配置 Nginx 反向代理完整教程

## 概述

Nginx 反向代理是现代 Web 应用架构中的重要组件，它可以：
- 负载均衡多个后端服务
- 提供 SSL 终端
- 缓存静态内容
- 隐藏后端服务器细节
- 提高安全性和性能

本教程将详细介绍如何在云服务器上配置 Nginx 反向代理。

## 前置条件

- 一台运行 Ubuntu 20.04/22.04 或 CentOS 7/8 的云服务器
- 具有 sudo 权限的用户账户
- 一个或多个运行中的后端应用（如 Node.js、Django、Spring Boot 等）
- 域名（可选，用于 SSL 配置）

## 第一步：安装 Nginx

### Ubuntu/Debian 系统

```bash
# 更新包管理器
sudo apt update

# 安装 Nginx
sudo apt install nginx -y

# 启动并启用 Nginx 服务
sudo systemctl start nginx
sudo systemctl enable nginx

# 检查 Nginx 状态
sudo systemctl status nginx
```

### CentOS/RHEL 系统

```bash
# 安装 EPEL 源
sudo yum install epel-release -y

# 安装 Nginx
sudo yum install nginx -y

# 启动并启用 Nginx 服务
sudo systemctl start nginx
sudo systemctl enable nginx

# 检查 Nginx 状态
sudo systemctl status nginx
```

## 第二步：配置防火墙

```bash
# Ubuntu (UFW)
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable

# CentOS (firewalld)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## 第三步：基本的反向代理配置

### 创建配置文件

```bash
# 创建新的站点配置文件
sudo nano /etc/nginx/sites-available/reverse-proxy
```

### 基本反向代理配置示例

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    # 日志配置
    access_log /var/log/nginx/reverse-access.log;
    error_log /var/log/nginx/reverse-error.log;

    # 反向代理到后端服务
    location / {
        proxy_pass http://127.0.0.1:3000;  # 后端服务地址
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # API 路由代理示例
    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 静态文件直接提供
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 启用配置

```bash
# 创建软链接启用站点
sudo ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/

# 删除默认配置（可选）
sudo rm /etc/nginx/sites-enabled/default

# 测试配置
sudo nginx -t

# 重新加载 Nginx
sudo systemctl reload nginx
```

## 第四步：负载均衡配置

如果你有多个后端服务器，可以配置负载均衡：

```nginx
# 在 http 块中定义上游服务器
upstream backend {
    least_conn;  # 负载均衡算法：least_conn, ip_hash, hash, random
    server 127.0.0.1:3000 weight=3;
    server 127.0.0.1:3001 weight=2;
    server 127.0.0.1:3002 weight=1;
    server 127.0.0.1:3003 backup;  # 备用服务器
}

server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 健康检查
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    }
}
```

## 第五步：SSL/HTTPS 配置

### 使用 Let's Encrypt 免费证书

```bash
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx -y  # Ubuntu
# 或
sudo yum install certbot python3-certbot-nginx -y  # CentOS

# 获取 SSL 证书
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# 设置自动续期
sudo crontab -e
# 添加以下行：
# 0 12 * * * /usr/bin/certbot renew --quiet
```

### 手动 SSL 配置

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    return 301 https://$server_name$request_uri;  # 重定向到 HTTPS
}

server {
    listen 443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    # SSL 配置
    ssl_certificate /path/to/your/certificate.crt;
    ssl_certificate_key /path/to/your/private.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # 安全头部
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 第六步：性能优化

### 启用 Gzip 压缩

```nginx
# 在 http 块中添加
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_proxied any;
gzip_comp_level 6;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/json
    application/javascript
    application/xml+rss
    application/atom+xml
    image/svg+xml;
```

### 缓存配置

```nginx
# 静态文件缓存
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# 代理缓存
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g 
                 inactive=60m use_temp_path=off;

location / {
    proxy_cache my_cache;
    proxy_cache_revalidate on;
    proxy_cache_min_uses 3;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_background_update on;
    proxy_cache_lock on;
    
    proxy_pass http://backend;
}
```

### 速率限制

```nginx
# 在 http 块中定义速率限制区域
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

## 第七步：监控和日志

### 自定义日志格式

```nginx
# 在 http 块中定义日志格式
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'rt=$request_time uct="$upstream_connect_time" '
                'uht="$upstream_header_time" urt="$upstream_response_time"';

server {
    access_log /var/log/nginx/access.log main;
}
```

### 状态监控

```nginx
# 启用状态页面
location /nginx_status {
    stub_status on;
    allow 127.0.0.1;
    deny all;
}
```

## 第八步：常见问题和解决方案

### 1. 502 Bad Gateway 错误

```bash
# 检查后端服务是否运行
sudo netstat -tlnp | grep :3000

# 检查 SELinux 设置（CentOS）
sudo setsebool -P httpd_can_network_connect 1

# 检查防火墙规则
sudo iptables -L
```

### 2. 配置文件语法错误

```bash
# 测试配置文件语法
sudo nginx -t

# 查看详细错误信息
sudo nginx -T
```

### 3. 权限问题

```bash
# 检查 Nginx 用户
ps aux | grep nginx

# 修改文件权限
sudo chown -R nginx:nginx /var/www/
sudo chmod -R 755 /var/www/
```

### 4. 性能调优

```nginx
# 在 nginx.conf 的 events 块中
worker_processes auto;
worker_connections 1024;

# 在 http 块中
keepalive_timeout 65;
client_max_body_size 50M;
```

## 第九步：备份和维护

### 配置文件备份

```bash
# 创建备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
tar -czf /backup/nginx_config_$DATE.tar.gz /etc/nginx/
```

### 定期维护

```bash
# 清理日志文件
sudo logrotate /etc/logrotate.d/nginx

# 检查配置并重新加载
sudo nginx -t && sudo systemctl reload nginx
```

## 总结

通过以上步骤，你已经成功配置了一个功能完整的 Nginx 反向代理服务器。记住：

1. 定期更新 Nginx 和系统
2. 监控服务器性能和日志
3. 备份重要的配置文件
4. 测试灾难恢复流程

根据你的具体需求，可能还需要进一步的定制配置。建议在生产环境中进行充分的测试，确保配置满足性能和安全要求。