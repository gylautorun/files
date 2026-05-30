# Nginx 前端多项目部署运维手册

**适配场景**：交接存量环境、38个前端SPA项目统一部署

**固定目录规范**：所有前端项目统一存放 **/usr/share/nginx/html/项目名**

**核心上线原则**：**更新前端静态文件，无需重启/重载Nginx；仅修改conf配置才需要重载**

## 一、Nginx 基础启停/重载命令

所有配置修改、服务维护专用，前端代码上线无需执行

```bash
# 启动Nginx
systemctl start nginx

# 设置开机自启
systemctl enable nginx

# 平滑重载（修改conf配置必执行，不中断线上服务）
nginx -t
systemctl reload nginx

# 查看运行状态
systemctl status nginx
```

## 二、项目部署固定目录结构

沿用原有交接环境，不改动系统目录，适配多项目统一管理

```Plain
/usr/share/nginx/html/A    # 项目A
/usr/share/nginx/html/B    # 项目B
/usr/share/nginx/html/C    # 项目C
... 所有前端项目统一存放此处
```

**结构要求**：解压后一级目录必须直接是 index\.html、css、js 等静态文件，Nginx可直接识别访问。

## 三、项目上传方案（真实专属实操）

### 3\.1 全量上传（scp 适合首次部署、大批量更新）

**核心特性**：覆盖已有文件、合并目录、无重复文件，简单粗暴不易出错

专属固定工作路径：本地打包静态文件目录 `\~/Desktop/nginx\-backup/build/nginx/html`

```bash
# 进入本地打包目录
cd ~/Desktop/nginx-backup/build/nginx/html

# 全量上传覆盖服务端所有前端项目
scp -r ./ root@192.168.9.158:/usr/share/nginx/html/
```

**执行结果**：

- 本地新文件 **直接覆盖** 服务端旧文件
- 同名文件夹自动合并，不会生成重复项目
- 无需重启Nginx，页面刷新即可生效

### 3\.2 增量上传（rsync 适合日常迭代更新，速度最快）

**核心特性**：仅上传「新增/修改文件」，跳过未变更文件，支持断点续传，替代scp全量上传，适合单个项目日常迭代上线

```bash
# 基础增量同步（推荐默认使用，安全不删服务端文件）
rsync -avzP ./本地项目文件夹/ root@192.168.9.158:/usr/share/nginx/html/目标项目文件夹/

# 严格镜像同步（谨慎！本地删除的文件，服务端同步删除）
rsync -avzP --delete ./本地项目文件夹/ root@192.168.9.158:/usr/share/nginx/html/目标项目文件夹/
```

#### 参数释义

- `\-a`：归档模式，保留文件权限、目录结构
- `\-v`：显示上传详情
- `\-z`：传输压缩，节省带宽、提速
- `\-P`：显示进度、支持断点续传
- `\-\-delete`：镜像删除，谨慎使用

**路径避坑**：文件夹末尾 `/` 必须加，代表同步文件夹内所有内容，而非上传整个文件夹

## 四、服务端文件本地备份下载（专属实操）

日常维护必备：拉取服务端完整 Nginx 配置\+所有前端项目文件到本地备份，用于本地修改、版本留存、故障回滚，固定实操命令如下

专属本地备份路径：`\~/Desktop/shuzhi\_code/nginx\-backup/build`

```bash
# 下载服务端完整 nginx 目录（配置+所有前端项目）到本地备份目录
scp -r root@192.168.9.158:/usr/share/nginx ~/Desktop/shuzhi_code/nginx-backup/build
```

**下载核心作用**：

- 完整同步服务端 Nginx 所有配置、html 前端项目文件到本地
- 本地修改配置、更新前端代码，避免直接在服务器改文件导致出错
- 留存线上完整备份，支持随时版本对比、故障回滚

## 五、Zip压缩包全自动上线方案（多项目最优解）

最优上线流程：本地打包dist为**项目名\.zip** → 上传服务端html目录 → 自动解压覆盖对应项目 → 自动删除压缩包，全程无需手动操作、无需重启Nginx

### 5\.1 全自动解压脚本（适配全部项目）

脚本路径：`/usr/share/nginx/html/auto\_unzip\.sh`

```bash
#!/bin/bash
DIR="/usr/share/nginx/html"
cd $DIR

# 遍历所有zip包，自动匹配项目名解压覆盖、清理压缩包
for zip_file in *.zip; do
  [ -f "$zip_file" ] || continue
  PROJECT_NAME="${zip_file%.zip}"
  unzip -o "$zip_file" -d "$PROJECT_NAME"
  rm -f "$zip_file"
  echo "✅ 已完成：$zip_file → 解压至 $PROJECT_NAME，已删除压缩包"
done

echo "🎉 所有压缩包自动部署完成！"
```

* Nginx `unzip -o`： 单纯 `unzip -o` ≠ 完整替换 `unzip -o` 逻辑：

  * ✅ 同名文件覆盖
  * ❌ 旧版本多余文件、废弃文件、残留目录不删除

```bash
#!/bin/bash
# Nginx前端项目自动部署脚本：清空旧目录 - 全新解压 - 清理压缩包
DIR="/usr/share/nginx/html"
cd $DIR

# 遍历当前目录下所有zip压缩包
for zip_file in *.zip; do
  # 判断是否为有效文件，跳过空匹配
  [ -f "$zip_file" ] || continue
  
  # 获取项目目录名（去除.zip后缀）
  PROJECT_NAME="${zip_file%.zip}"
  
  # 1. 先移除已存在的旧项目目录（彻底清空旧文件）
  if [ -d "$PROJECT_NAME" ]; then
    rm -rf "$PROJECT_NAME"
    echo "🗑️  已清理旧项目目录：$PROJECT_NAME"
  fi
  
  # 2. 全新解压压缩包到对应项目目录
  unzip -o "$zip_file" -d "$PROJECT_NAME"
  
  # 3. 解压完成后删除原压缩包
  rm -f "$zip_file"
  
  echo "✅ 部署完成：$zip_file → 全新解压至 $PROJECT_NAME，压缩包已清理"
done

echo -e "\n🎉 所有前端压缩包部署完成！旧文件已清空，新项目已全覆盖！"
```


### 5\.2 赋予脚本执行权限（仅首次配置）

```bash
chmod +x /usr/share/nginx/html/auto_unzip.sh
```

### 5\.3 手动触发执行

```bash
/usr/share/nginx/html/auto_unzip.sh
```

### 5\.4 定时自动执行（每10秒全自动检测部署）

无需手动执行，上传zip包后10秒内自动完成全流程部署

#### 编辑定时任务

```bash
crontab -e
```

#### 粘贴定时任务配置

```bash
* * * * * /usr/share/nginx/html/auto_unzip.sh
* * * * * sleep 10 && /usr/share/nginx/html/auto_unzip.sh
* * * * * sleep 20 && /usr/share/nginx/html/auto_unzip.sh
* * * * * sleep 30 && /usr/share/nginx/html/auto_unzip.sh
* * * * * sleep 40 && /usr/share/nginx/html/auto_unzip.sh
* * * * * sleep 50 && /usr/share/nginx/html/auto_unzip.sh
```

#### 查看已配置定时任务

```bash
crontab -l
```

### 5\.5 完整Zip上线标准流程

1. 本地打包前端dist文件，压缩为 `项目名\.zip`（A\.zip、B\.zip 对应对应项目）
2. 上传压缩包至服务器 `/usr/share/nginx/html/` 目录
3. 定时任务自动检测压缩包，自动解压覆盖对应项目文件夹
4. 解压完成后自动删除zip压缩包，无文件残留
5. 浏览器刷新页面即可展示最新版本，**无需操作Nginx**

## 六、服务端文件/文件夹删除命令

**高危提醒**：rm 删除文件/文件夹后无法恢复，仅删除无用项目、冗余文件

```bash
# 进入项目根目录
cd /usr/share/nginx/html/

# 1. 删除单个文件（zip包、日志等）
rm -f 文件名.zip

# 2. 删除整个项目文件夹（彻底删除对应项目）
rm -rf 项目名

# 3. 批量清空当前目录所有zip压缩包
rm -f *.zip
```

## 七、核心上线规则（必记避坑）

- **仅更新前端静态文件**：上传/解压覆盖即可，**无需重启、重载Nginx**
- **修改Nginx配置文件（\.conf）**：必须执行 `nginx \-t \&amp;\&amp; systemctl reload nginx` 校验并平滑重载
- **scp全量上传**：覆盖旧文件、合并目录，无重复冗余文件，适合批量部署
- **rsync增量上传**：仅更新变更文件，节省带宽，日常迭代首选
- **Zip自动化部署**：多项目上线最快、最干净的方式，全程无人值守

## 八、日常上线场景快速选择

- 首次批量部署38个项目：**scp全量上传**
- 单个项目日常迭代更新：**rsync增量上传 / Zip包自动部署**
- 清理旧项目/垃圾压缩包：**rm \-rf / rm \-f 精准删除**
- 修改域名/端口/路由等配置：**修改conf配置 \+ 重载Nginx**

## 九、专属常用命令速查表（一键复制）

**固定信息**：服务器IP `192\.168\.9\.158`、服务端项目根目录 `/usr/share/nginx/html`

**本地固定路径**：打包目录 `\~/Desktop/nginx\-backup/build/nginx/html`、备份目录 `\~/Desktop/shuzhi\_code/nginx\-backup/build`

### 9\.1 Nginx 服务基础命令

```bash
# 启动服务
systemctl start nginx
# 开机自启
systemctl enable nginx
# 校验配置+平滑重载（改配置必用）
nginx -t && systemctl reload nginx
# 查看运行状态
systemctl status nginx
```

### 9\.2 项目全量上线命令

```bash
# 进入本地打包目录，全量覆盖上传所有项目
cd ~/Desktop/nginx-backup/build/nginx/html
scp -r ./ root@192.168.9.158:/usr/share/nginx/html/
```

### 9\.3 项目增量上线命令

```bash
# 安全增量更新（仅更新变更文件，不删服务端文件）
rsync -avzP ./本地项目文件夹/ root@192.168.9.158:/usr/share/nginx/html/目标项目文件夹/

# 严格镜像更新（本地删除文件，服务端同步删除，谨慎使用）
rsync -avzP --delete ./本地项目文件夹/ root@192.168.9.158:/usr/share/nginx/html/目标项目文件夹/
```

### 9\.4 服务端文件本地备份命令

```bash
# 完整下载服务端nginx所有配置+项目到本地备份目录
scp -r root@192.168.9.158:/usr/share/nginx ~/Desktop/shuzhi_code/nginx-backup/build
```

### 9\.5 Zip自动部署相关命令

```bash
# 赋予解压脚本执行权限（仅首次配置）
chmod +x /usr/share/nginx/html/auto_unzip.sh

# 手动触发zip自动解压+删包
/usr/share/nginx/html/auto_unzip.sh

# 查看定时任务
crontab -l
# 编辑定时任务
crontab -e
```

### 9\.6 服务端文件/文件夹删除命令

```bash
# 进入项目根目录
cd /usr/share/nginx/html/

# 删除单个zip文件
rm -f 文件名.zip

# 删除整个项目文件夹
rm -rf 项目名

# 批量清空所有zip包
rm -f *.zip
```

### 9\.7 速查核心规则

- 更新前端静态资源、zip部署：**无需重启/重载Nginx**
- 修改Nginx配置文件：**必须校验并平滑重载**
- scp：全量覆盖合并，适合批量部署
- rsync：增量更新，适合日常迭代
- 自动部署：上传zip包，10秒内自动解压删包上线
