# 前端服务器 Nginx 从零安装、配置、验证完整实操文档（CentOS）

**适用人群**：前端开发者、无服务器运维基础

**适用系统**：CentOS 7 / 8 / 9

**文档目的**：一套流程搞定 Nginx 安装、校验、找配置、避坑，可长期复用

## 一、前置：登录远程服务器

打开终端 / FinalShell / Xshell，执行登录命令：

```bash
ssh root@你的服务器公网IP
```

输入服务器密码，回车登录即可。

## 二、CentOS 完整安装 Nginx（必走流程）

### 1\. 可能要先装 epel\-release？

CentOS 官方默认软件源 **没有 Nginx**，必须先安装 EPEL 扩展软件源，yum 才能识别并安装 Nginx，属于**必填步骤，不可省略**。

### 2\. 安装命令

```bash
# 1. 安装扩展软件源
yum install epel-release -y

# 2. 正式安装 Nginx
yum install nginx -y
```

### 3\. 启动 Nginx \+ 设置开机自启（核心）

```bash
# 启动 Nginx
systemctl start nginx

# 设置开机自启（服务器重启后自动运行）
systemctl enable nginx
```

## 三、Nginx 核心安装目录（重点）

**重要说明**：yum 安装的 Nginx 不会生成 `/data/nginx` 目录！**data 目录为空是正常现象，不是安装失败**。

Nginx 所有核心文件默认固定路径：

```Plain
# 主程序二进制文件
/usr/sbin/nginx

# 主配置文件（全局配置）
/etc/nginx/nginx.conf

# 站点子配置目录（我们前端主要用这个！）
/etc/nginx/conf.d/

# Nginx 默认静态页面目录
/usr/share/nginx/html

# 日志目录（报错、访问日志）
/var/log/nginx/
```

## 四、4 种方式验证 Nginx 安装 \&amp; 运行正常

全部执行一遍，100% 确认环境没问题

### 1\. 查看 Nginx 版本（验证安装成功）

```bash
nginx -v
```

输出版本号（如 nginx/1\.20\.1）= 安装成功

### 2\. 检查配置文件语法（零报错核心命令）

```bash
nginx -t
```

出现以下两行代表配置无问题：

```Plain
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 3\. 查看 Nginx 运行状态

```bash
systemctl status nginx
```

显示 **active \(running\)** = 正在正常运行

### 4\. 浏览器访问验证

浏览器打开：`http://服务器公网IP`

看到 Nginx 官方欢迎页面 = 服务正常、端口正常、防火墙放行正常

## 五、前端常用 Nginx 核心操作命令

```bash
# 启动 Nginx
systemctl start nginx

# 停止 Nginx
systemctl stop nginx

# 重启 Nginx（中断访问，不推荐）
systemctl restart nginx

# 平滑重载配置（改完配置必用！不中断用户访问）
systemctl reload nginx

# 检查配置语法
nginx -t

# 查看运行状态
systemctl status nginx
```

## 六、前端项目标准配置流程

专门适配 Vue/React 单页项目，解决刷新404、静态资源缓存问题

### 1\. 进入站点配置目录

```bash
cd /etc/nginx/conf.d/
```

### 2\. 创建项目配置文件

```bash
vi front-project.conf
```

### 3\. 粘贴通用前端配置

```nginx
server {
    listen 80;
    server_name 你的服务器IP;

    # 前端dist包存放目录
    root /var/www/front-dist;
    index index.html;

    # 解决单页应用刷新404
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源缓存，提升访问速度
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
        expires 7d;
    }
}
```

ESC → 输入 `:wq` 保存退出

### 4\. 创建前端文件存放目录

```bash
mkdir -p /var/www/front-dist
```

### 5\. 生效配置

```bash
nginx -t
systemctl reload nginx
```

## 七、前端本地修改Nginx配置（无需服务器在线编辑，增量上传）

**核心痛点**：直接在服务器 vi 编辑配置易输错、无备份、无法本地格式化、无法留存版本。

推荐 **本地编辑 \+ 单独上传配置文件 \+ 重载生效** 方案，只同步修改的配置文件，不改动服务器其他环境文件，安全高效。

### 1、核心原理（前端必懂）

- 我们所有**前端项目自定义配置**都存放在 `/etc/nginx/conf\.d/` 目录，默认是空的，只有我们自己创建的 `\.conf` 项目配置
- 主配置 `nginx\.conf` 一般不动，只维护 conf\.d 下的项目配置文件
- **只需本地下载对应 conf 文件、修改、重新上传覆盖即可**，属于增量更新，不影响任何环境

### 2、完整命令行操作流程（本地修改 → 命令下载/上传 → 生效）

**说明**：摒弃图形化鼠标上传方式，全程使用简单 SSH 命令，适合纯终端操作、精准增量同步，前端可固定这套流程使用

#### 第一步：【仅首次】拉取服务器 Nginx 配置到本地（两种方案：单文件 / 整文件夹）

打开**本地电脑终端**，根据需求任选一种下载方案，下载后永久留存本地，后续只修改本地文件，无需登录服务器编辑配置。

**方案一：仅下载项目配置文件（日常使用首选、轻量化）**

适合只维护前端自定义配置，无需改动 Nginx 系统默认配置的场景，下载速度快、文件干净。

```bash
# 下载单个前端项目配置文件到本地当前目录
scp root@你的服务器IP:/etc/nginx/conf.d/front-project.conf ./
```

执行后本地生成 `front\-project\.conf`，仅包含你的前端站点配置。

**方案二：下载完整 /etc/nginx 全部配置文件夹（全套备份、新手推荐）**

一次性下载 Nginx 所有配置（主配置、子配置、系统默认参数、静态配置），完整复刻服务器所有配置，方便本地查阅、对比、全套修改备份。

```bash
# 递归下载服务器完整 nginx 配置文件夹到本地
# scp -r 服务器地址:服务器路径 本地自定义保存路径
scp -r root@你的服务器IP:/etc/nginx ./
scp -r root@你的IP:/etc/nginx ~/Desktop/nginx-backup
```

参数说明：`\-r` 代表递归下载，包含所有子文件夹、配置文件，无遗漏。

- 打开新命令行
- `\./` = **当前终端所在目录**（默认）
- 改成任意**本地绝对路径** = 自定义下载位置

下载完成后本地生成 `nginx` 文件夹，完整包含：

- 主配置文件：`nginx\.conf`
- 前端项目配置：`conf\.d/` 目录下所有 \.conf 文件
- 系统默认配置、mime资源类型、代理参数等全套基础配置

打开**本地电脑终端**，执行下载命令，把服务器配置拉到本地：

```bash
# 语法：scp 服务器用户名@IP:服务器文件路径 本地保存路径
scp root@你的服务器IP:/etc/nginx/conf.d/front-project.conf ./
```

执行后，当前本地文件夹会得到 `front\-project\.conf`，永久留存，后续只改此本地文件。

#### 第二步：本地 VS Code 修改配置

本地打开文件修改（代理、缓存、路由、域名、跨域等），保存即可，无需操作服务器。

#### 第三步：【核心】本地修改好的文件，命令上传覆盖服务器

依旧在**本地电脑终端**执行上传覆盖命令：

```bash
# 本地文件覆盖服务器配置文件（精准单文件增量更新）
scp ./front-project.conf root@你的服务器IP:/etc/nginx/conf.d/
```

执行后直接覆盖服务器原有配置，无多余文件改动，安全精准。

#### 第四步：服务器端校验 \+ 平滑生效（必执行，防线上报错）

登录服务器后执行两条固定命令：

```bash
# 1. 校验配置语法是否正确（改错直接拦截，不会崩服务）
nginx -t

# 2. 平滑重载配置（不中断用户访问，线上无停机）
systemctl reload nginx
```

输出 `test is successful` 代表修改完全生效。

### 4、必备习惯：配置备份（防止改错崩溃）

如果不想整文件覆盖，可通过命令对比差异、只同步修改内容，适合多人维护、严格版本控制场景：

```bash
# 查看本地与服务器配置差异
diff /etc/nginx/conf.d/front-project.conf 本地文件路径

# 按需复制修改行，服务器端追加/修改，无需覆盖全文件
vi /etc/nginx/conf.d/front-project.conf
```

### 4、必备习惯：配置备份（防止改错崩溃）

每次修改上传前，先备份服务器原有配置，一键回滚：

```bash
# 备份原有配置（带时间戳，不覆盖旧备份）
cp /etc/nginx/conf.d/front-project.conf /etc/nginx/conf.d/front-project-bak-$(date +%Y%m%d).conf
```

### 5、常见问题解答

- **Q****：能不能只改部分配置，不上传整个文件？**
  A：可以，优先本地改完整体上传（最简单、零出错），严谨场景用 diff 对比差异、局部修改。
- **Q****：上传覆盖会不会影响 Nginx 主程序？**
  A：完全不会，仅替换自定义项目配置，不改动系统默认配置和程序。
- **Q****：改错了如何快速回滚？**
  A：直接将备份的 bak 配置文件重新覆盖，重载配置即可秒恢复。

## 八、常见问题汇总

- **/data 目录没有 nginx 文件夹？** 正常！yum 安装默认不生成，无需创建、无需处理
- **浏览器访问不通？** 去服务器安全组放行 80 端口
- **页面刷新404？** 配置中 `try\_files` 代码未添加，复制上方完整配置即可
- **为什么 cat /usr/sbin/nginx 是乱码？（cat 某个文件乱码）**
  - `/usr/sbin/nginx` 是 **二进制可执行程序**（相当于 Windows 的 exe 文件）
  - `cat`命令只能查看文本文件，无法读取程序文件，所以会出现乱码
  - **千万不要用 cat 查看 nginx 主程序**
