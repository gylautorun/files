# Nginx 前端配置手册（前端专属·完整版）

**适用人群**：前端开发、零基础服务器部署

**手册定位**：只保留前端必备配置，剔除运维冗余功能，涵盖配置工程化拆分、SPA 部署、反向代理、缓存、Gzip/Br预压缩、多站点、排错方案

**前置说明**：服务器安装、配置文件本地下载/上传流程已独立整理，本手册专注**前端业务配置核心语法与实战场景**

## 一、Nginx 核心配置基础认知

### 1\.1 核心目录规范（前端固定用法）

遵循 **主配置\+站点拆分**工程化规范，所有前端项目配置统一托管，整洁易维护

- `/etc/nginx/nginx\.conf`：Nginx 主配置文件（全局配置、公共引用）
- `/etc/nginx/conf\.d/`：前端项目站点配置目录（所有项目 \.conf 放这里）
- `/etc/nginx/common/`：自定义公共配置目录（存放复用规则，手动创建）
- `/var/www/xxx`：前端打包 dist 静态资源存放目录
- `/var/log/nginx/`：访问日志、错误日志目录（排查问题专用）

### 1\.2 前端高频核心指令释义

- **listen**：监听端口，默认 80 端口为网页默认访问端口
- **server\_name**：绑定服务器 IP 或域名，用于区分多站点
- **root**：前端静态资源 dist 根目录路径
- **index**：项目默认首页文件，前端固定为 index\.html
- **location**：路由匹配规则，处理不同请求路径
- **try\_files**：前端 SPA 核心指令，解决路由刷新 404 问题
- **expires**：静态资源浏览器缓存配置，优化加载速度
- **proxy\_pass**：反向代理，转发接口请求到后端服务
- **include**：**前端工程化核心**，引入外部公共配置，实现配置复用、解耦

## 二、Include 配置拆分（前端必学工程化用法）

### 2\.1 Include 核心作用

解决单配置文件臃肿、重复代码问题，将**通用缓存、跨域、请求头**规则抽离为公共文件，多个前端项目直接引用，改一处全局生效。

### 2\.2 系统默认加载规则（重点）

Nginx 主配置默认自带全局引用，自动加载 `conf\.d` 下所有后缀为 `\.conf` 的站点配置，无需手动引入：

```nginx
http {
    # 自动加载所有前端站点配置
    include /etc/nginx/conf.d/*.conf;
}
```

### 2\.3 实操：搭建前端公共配置库

#### 第一步：创建公共配置文件夹

```bash
mkdir -p /etc/nginx/common
```

#### 第二步：创建通用静态缓存配置 common/static\.conf

所有前端项目通用静态资源缓存规则

```nginx
# 静态资源统一缓存规则
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
    expires 7d;                # 浏览器缓存7天
    add_header Cache-Control "public";
}
```

#### 第三步：创建通用跨域配置 common/cors\.conf

```nginx
# 前端通用跨域配置
add_header Access-Control-Allow-Origin *;
add_header Access-Control-Allow-Methods GET,POST,OPTIONS,PUT,DELETE;
add_header Access-Control-Allow-Headers *;

# 处理浏览器 OPTIONS 预检请求
if ($request_method = OPTIONS) {
    return 204;
}
```

#### 第四步：项目配置引用公共规则

```nginx
server {
    listen 80;
    server_name 你的服务器IP;
    root /var/www/front-dist;
    index index.html;

    # 解决 SPA 刷新404
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 复用公共缓存规则
    include /etc/nginx/common/static.conf;
    # 复用公共跨域规则
    include /etc/nginx/common/cors.conf;
}
```

## 三、前端高频场景完整配置（可直接复制上线）

### 3\.1 基础版：Vue/React SPA 单页项目（最常用）

适配绝大多数前端项目，解决刷新404 \+ 静态资源缓存优化

```nginx
server {
    listen 80;
    server_name 你的IP/域名;
    root /var/www/front-dist;
    index index.html;

    # 核心：前端路由 history 模式刷新404解决方案
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|svg|woff2)$ {
        expires 7d;
    }
}
```

### 3\.2 进阶版：SPA \+ 接口反向代理（前后端分离必备）

前端页面由 Nginx 托管，/api 接口请求转发到后端服务，**前端代码无需改接口地址**

```nginx
server {
    listen 80;
    server_name 你的IP/域名;
    root /var/www/front-dist;
    index index.html;

    # 前端路由兜底
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 接口反向代理
    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/; # 后端接口地址
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;       # 传递真实客户端IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 引入公共缓存规则
    include /etc/nginx/common/static.conf;
}
```

### 3\.3 多站点部署（一台服务器多个前端项目）

#### 方式1：同端口、不同域名区分（推荐）

```nginx
# 项目1：主站
server {
    listen 80;
    server_name www.xxx.com;
    root /var/www/main-dist;
    index index.html;
    location / { try_files $uri $uri/ /index.html; }
    include /etc/nginx/common/static.conf;
}

# 项目2：后台管理系统
server {
    listen 80;
    server_name admin.xxx.com;
    root /var/www/admin-dist;
    index index.html;
    location / { try_files $uri $uri/ /index.html; }
    include /etc/nginx/common/static.conf;
}
```

#### 方式2：同域名、不同端口区分

```nginx
# 前台项目 8081 端口
server {
    listen 8081;
    server_name _;
    root /var/www/front-dist;
    index index.html;
    location / { try_files $uri $uri/ /index.html; }
}

# 后台项目 8082 端口
server {
    listen 8082;
    server_name _;
    root /var/www/admin-dist;
    index index.html;
    location / { try_files $uri $uri/ /index.html; }
}
```

### 3\.4 全局 Gzip \+ Brotli 预压缩（前端生产最优完整版）

配置在 `nginx\.conf` 的 http 全局块，所有站点生效。**同时支持「实时压缩」\+「前端打包预压缩」**，优先读取项目打包生成的 \.gz/\.br 文件，极致提速，是前端线上标准优化方案。

```nginx
http {
    # ===================== Gzip 压缩配置 =====================
    gzip on;
    gzip_min_length 1k;
    gzip_types text/plain text/css text/javascript application/javascript application/json application/x-javascript text/xml application/xml application/xml+rss;
    gzip_vary on;
    gzip_buffers 4 16k;
    gzip_comp_level 6;

    # 核心：开启Gzip预压缩读取
    # 优先加载前端打包好的 *.gz 文件，不实时计算压缩
    gzip_static on;

    # ===================== Brotli 高级压缩（压缩率更高） =====================
    brotli on;
    brotli_static on; # 优先读取前端打包的 *.br 预压缩文件
    brotli_min_length 1k;
    brotli_types text/plain text/css text/javascript application/javascript application/json;
    brotli_comp_level 6;
}
```

**配套前端说明**：Vue/React 项目可在打包配置开启 `gzip`、`brotli` 预打包，生成 \.gz/\.br 静态文件，Nginx 会优先加载，相比实时压缩性能提升极大。

### 3\.5 静态资源目录浏览（文件资源站专用）

适用于文档站、资源下载站，**禁止用于正式业务 SPA 项目**

```nginx
server {
    listen 80;
    server_name file.xxx.com;
    root /var/www/files;

    autoindex on;                # 开启目录浏览
    autoindex_exact_size off;    # 简化文件大小展示
    autoindex_localtime on;      # 使用本地时间
}
```

## 四、Location 匹配优先级（前端必懂）

优先级从高到低，精准控制路由匹配规则

1. **精确匹配**：`location = /` 精准匹配指定路径，优先级最高
2. **正则匹配**：`location \~\* \\\.\(png\|jpg\|js\|css\)` 匹配静态资源后缀
3. **前缀匹配**：`location /api/`匹配指定路由前缀
4. **通用匹配**：`location /` 兜底匹配，优先级最低

## 五、配置生效固定流程（零事故上线）

无论修改、上传单文件/全量配置，**必须执行以下两步**，避免语法错误导致服务宕机

```bash
# 1. 校验配置语法（有错直接提示，不会生效）
nginx -t

# 2. 平滑重载配置（不中断线上用户访问）
systemctl reload nginx
```

## 六、前端常见问题速查手册

- **页面刷新404**：缺失 `try\_files $uri $uri/ /index\.html;` 路由兜底规则
- **静态资源加载慢**：未配置 expires 缓存、未开启 Gzip/Br 预压缩优化
- **接口跨域报错**：未配置跨域请求头，或代理路径拼接错误
- **include 引入不生效**：检查文件路径、后缀必须为 \.conf、文件权限正常
- **配置上传不生效**：未执行 nginx \-t 校验、未重载配置
- **多站点冲突**：同端口下 server\_name 重复、路由规则重叠

## 七、前端 Nginx 配置最佳实践总结

1. **配置解耦**：主配置 nginx\.conf 只保留全局规则，业务站点全部放 conf\.d
2. **规则复用**：通用缓存、跨域规则抽离到 common，通过 include 全局复用
3. **本地开发**：全部配置本地修改、校验后上传，不在服务器 vi 编辑
4. **上线规范**：任何修改必做语法校验 \+ 平滑重载，杜绝暴力重启
5. **性能优化**：所有前端项目默认开启 7天静态缓存 \+ Gzip/Br 预压缩双优化
