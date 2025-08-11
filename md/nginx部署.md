### **一、环境准备**

#### 1. 安装 Nginx

bash

复制

```
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS
sudo yum install epel-release
sudo yum install nginx

# 验证安装
nginx -v
```

#### 2. 项目准备

- 构建前端项目（以 Vue/React 为例）：

  bash

  复制

  ```
  npm run build  # 生成 dist 或 build 目录
  ```

#### 3. 上传文件到服务器

- 将构建产物（如 `dist` 目录）上传至服务器目标路径，例如：

  bash

  复制

  ```
  scp -r dist/* user@server_ip:/var/www/my-frontend
  ```

------

### **二、Nginx 核心配置**

#### 1. 主配置文件结构

nginx

复制

```
# /etc/nginx/nginx.conf（全局配置）
user www-data;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    keepalive_timeout 65;
    
    # 包含子配置（推荐）
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

#### 2. 站点配置文件

nginx

复制

```
# /etc/nginx/sites-available/my-frontend.conf
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;  # 域名或IP
    
    root /var/www/my-frontend;  # 项目路径
    index index.html;
    
    # 开启Gzip压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 1k;
    gzip_comp_level 6;
    gzip_vary on;
    
    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
        expires 365d;
        add_header Cache-Control "public, max-age=31536000, immutable";
        access_log off;
    }
    
    # 前端路由处理（解决刷新404问题）
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    # 错误页面配置
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    
    # 可选的访问限制
    location /admin {
        allow 192.168.1.0/24;  # 限制IP段
        deny all;
    }
}
```

#### 3. 启用配置

bash

复制

```
# 创建符号链接到 sites-enabled
sudo ln -s /etc/nginx/sites-available/my-frontend.conf /etc/nginx/sites-enabled/

# 测试配置语法
sudo nginx -t

# 重新加载Nginx
sudo systemctl reload nginx
```

------

### **三、HTTPS 配置（SSL 证书）**

#### 1. 获取证书（以 Let's Encrypt 为例）

bash

复制

```
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx

# 申请证书（自动修改Nginx配置）
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

#### 2. 更新 Nginx 配置

nginx

复制

```
server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL 优化配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # 其他配置同上...
}

# 强制HTTP跳转HTTPS
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}
```



### 注意

##### 如果是docker部署的nginx挂载正确的目录之后，配置文件中的root需要使用容器中的路径