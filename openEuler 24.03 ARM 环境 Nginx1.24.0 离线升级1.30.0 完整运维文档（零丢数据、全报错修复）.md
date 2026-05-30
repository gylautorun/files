# openEuler 24\.03 ARM 环境 Nginx1\.24\.0 离线升级1\.30\.0 完整运维文档

## （零丢数据、全报错修复）

## 一、升级环境与前置说明

### 1\.1 环境信息

- 操作系统：openEuler 24\.03 LTS（aarch64 ARM架构）
- 旧版本：Nginx 1\.24\.0
- 目标版本：Nginx 1\.30\.0（官方稳定版）
- 业务场景：部署前端静态项目（irs\-portal），需保留所有上传文件、站点配置

### 1\.2 升级核心限制（全程踩坑根源）

- 服务器外网源超时、443端口无法连通，所有 YUM/DNF 在线源（CentOS源、Oepkgs源）全部失效，**无法在线安装升级**
- openEuler 系统不兼容 CentOS 官方 Nginx 源，强行使用会报 404、元数据下载失败
- 仅支持**源码离线编译升级**，唯一可行方案
- 核心要求：不卸载旧业务、不删除前端上传资源、不丢失原有站点配置

### 1\.3 升级前置备份（必执行，保障数据安全）

全程备份配置与静态资源，杜绝数据丢失

```bash
# 备份Nginx全部配置文件
cp -r /etc/nginx /etc/nginx_bak_1.24.0
# 备份前端静态资源文件
cp -r /usr/share/nginx/html /usr/share/nginx/html_bak
```

## 二、环境预处理：清理失效源、修复依赖报错

### 2\.1 删除所有失效外网源

清理超时、404的无效源，避免后续命令卡死报错

```bash
rm -f /etc/yum.repos.d/nginx.repo
rm -f /etc/yum.repos.d/oepkgs.repo
dnf clean all
```

### 2\.2 禁用失效Oepkgs源

```bash
dnf config-manager --disable oepkgs
```

### 2\.3 安装本地编译依赖（系统默认源）

跳过外网源，使用系统自带源安装编译所需依赖，解决依赖缺失问题

```bash
dnf install -y gcc make zlib-devel pcre-devel openssl-devel
```

## 三、Nginx1\.30\.0 离线源码编译安装

### 3\.1 下载官方源码包（无404、可正常下载）

放弃失效阿里云镜像，使用Nginx官方包地址

```bash
cd /usr/local
wget http://nginx.org/download/nginx-1.30.0.tar.gz
tar -zxvf nginx-1.30.0.tar.gz
cd nginx-1.30.0
```

### 3\.2 合规编译配置（修复非法参数报错）

删除无效参数 `\-\-with\-http\_gzip\_module`，替换为官方合法参数，适配新旧版本兼容，完全对齐系统原有Nginx路径

```bash
./configure --prefix=/usr/share/nginx \
--sbin-path=/usr/sbin/nginx \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/run/nginx.lock \
--http-client-body-temp-path=/var/cache/nginx/client_temp \
--http-proxy-temp-path=/var/cache/nginx/proxy_temp \
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
--http-scgi-temp-path=/var/cache/nginx/scgi_temp \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_gzip_static_module
```

### 3\.3 编译与版本替换（无损升级核心）

仅编译不安装，备份旧版本二进制文件，覆盖替换为新版，不改动任何业务配置

```bash
# 编译源码
make
# 备份旧1.24.0程序
cp /usr/sbin/nginx /usr/sbin/nginx_1.24.0_bak
# 替换为1.30.0新版程序
cp objs/nginx /usr/sbin/nginx
chmod +x /usr/sbin/nginx
```

## 四、编译后缺失文件终极修复（解决启动全报错）

源码编译仅生成程序，不会自动生成配置文件、临时目录、systemd服务，需手动补全

### 4\.1 补全系统必备目录

```bash
mkdir -p /etc/nginx
mkdir -p /etc/nginx/conf.d
mkdir -p /var/cache/nginx/{client_temp,proxy_temp,fastcgi_temp,uwsgi_temp,scgi_temp}
```

### 4\.2 生成主配置文件 nginx\.conf

```bash
cat > /etc/nginx/nginx.conf <<EOF
user root;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '\$remote_addr - \$remote_user [\$time_local] "\$request" '
                      '\$status \$body_bytes_sent "\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;

    include /etc/nginx/conf.d/*.conf;
}
EOF
```

### 4\.3 生成mime类型文件（页面解析必备）

```bash
cat > /etc/nginx/mime.types <<EOF
types {
    text/html                             html htm shtml;
    text/css                              css;
    text/xml                              xml;
    image/gif                             gif;
    image/jpeg                             jpg jpeg;
    image/png                             png;
    application/javascript                js;
    application/json                      json;
}
EOF
```

### 4\.4 恢复原有业务站点配置

```bash
cp -r /etc/nginx_bak_1.24.0/conf.d/* /etc/nginx/conf.d/
```

### 4\.5 修复配置语法报错（核心致命问题）

解决 `\&\#34;user\&\#34; directive is not allowed here` 报错：子配置文件禁止全局指令

```bash
# 清理子配置文件中非法的全局参数
sed -i '1,5d' /etc/nginx/conf.d/dms.conf
sed -i '/^user /d' /etc/nginx/conf.d/*.conf
sed -i '/^worker_processes/d' /etc/nginx/conf.d/*.conf
sed -i '/^events {/d' /etc/nginx/conf.d/*.conf
```

### 4\.6 修复前端路由报错（根治URL拼写错误）

修正历史错误：try\_files 误写 irs\-admin，统一为项目真实路径 irs\-portal

最终正确站点配置：

```nginx
server {
    listen       8006;
    location /irs-portal {
        alias /usr/share/nginx/html/irs-portal/;
        index index.html;
        try_files $uri $uri/ /irs-portal/index.html;
    }
}
```

### 4\.7 配置systemd系统服务（解决Unit不存在）

```bash
cat > /usr/lib/systemd/system/nginx.service <<EOF
[Unit]
Description=nginx web service
Documentation=http://nginx.org/en/docs/
After=network.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

## 五、最终校验与服务启动

```bash
# 刷新系统服务缓存
systemctl daemon-reload
# 校验配置语法（必须成功）
nginx -t
# 启动并设置开机自启
systemctl start nginx
systemctl enable nginx
# 查看运行状态
systemctl status nginx
# 验证升级版本
nginx -v
```

## 六、全程报错汇总与解决方案

| 报错现象                              | 根因                                 | 解决方案                                           |
| ------------------------------------- | ------------------------------------ | -------------------------------------------------- |
| Oepkgs源443超时、元数据下载失败       | 服务器外网不通，在线源全部失效       | 删除失效源，禁用oepkgs，改用离线编译               |
| \-\-with\-http\_gzip\_module 非法参数 | Nginx无此参数，参数名称错误          | 替换为合法参数\-\-with\-http\_gzip\_static\_module |
| nginx: command not found              | 编译后未将二进制文件放入系统环境变量 | 手动拷贝 objs/nginx 到 /usr/sbin/ 并授权           |
| nginx\.conf文件不存在                 | 源码编译不生成默认配置文件           | 手动创建主配置、mime类型文件                       |
| nginx\.service: Unit not found        | 无系统服务托管文件                   | 手动编写systemd服务文件并重载                      |
| user directive is not allowed here    | 子配置文件写入全局指令               | 清理conf\.d下所有非法全局参数                      |
| 前端URL拼写错误                       | try\_files兜底路径写错为irs\-admin   | 修正为真实项目路径irs\-portal                      |

## 七、升级成功验证标准

- 版本验证：`nginx \-v` 输出 nginx/1\.30\.0
- 配置校验：`nginx \-t` 提示 test is successful
- 服务状态：systemctl 可正常启动、重启Nginx，服务运行正常
- 业务访问：`http://192\.168\.9\.158:8006/irs\-portal` 无URL报错，页面正常加载
- 数据完整性：所有前端上传文件、自定义站点配置完全保留

## 八、核心总结

1\. 本次升级全程采用**离线源码编译**，完美适配 openEuler ARM 外网不通的特殊环境，是唯一可行方案；

2\. 所有报错均为**环境适配、语法配置问题**，与Nginx版本本身无关，升级后彻底兼容新版校验规则；

3\. 全程无损业务，未删除任何用户上传资源与站点配置，升级后业务完全恢复正常；

4\. 彻底解决旧版本1\.24\.0规则兼容缺陷，新版1\.30\.0校验更严格、运行更稳定。

> （注：文档部分内容可能由 AI 生成）
