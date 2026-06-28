# 115ED Docker / Docker Compose 部署教程

本文面向普通用户，说明如何使用 Docker 或 Docker Compose 部署 115ED，并完成 Emby、115 Cookie、STRM 路径映射、播放基地址、反向代理和常用维护操作。

115ED 主要用途：

```text
1. 连接 Emby
2. 使用 115 Cookie 读取网盘目录
3. 按路径映射生成 STRM 文件
4. 提供 /api/strm/play?pickcode=xxx 播放入口
5. 可选接收 Emby Webhook 和发送 Telegram 通知
```

## 一、部署前准备

### 1. 推荐系统

推荐使用：

```text
Debian 11 / Debian 12
Ubuntu 20.04 / 22.04 / 24.04
```

建议配置：

```text
CPU：1 核或以上
内存：512 MB 或以上
磁盘：预留足够空间保存 STRM、日志和配置
网络：能正常访问 115 网盘、Emby 和授权接口
```

### 2. 安装基础工具

```bash
apt update
apt install -y curl wget vim nano ca-certificates gnupg lsb-release
```

### 3. 安装 Docker

```bash
curl -fsSL https://get.docker.com | bash
```

启动 Docker：

```bash
systemctl enable docker
systemctl start docker
```

检查 Docker：

```bash
docker version
docker compose version
```

如果 `docker compose version` 不存在，旧环境可能使用：

```bash
docker-compose version
```

## 二、创建部署目录

建议放在 `/opt/115ed`：

```bash
mkdir -p /opt/115ed
cd /opt/115ed
```

创建配置、日志、STRM 目录：

```bash
mkdir -p logs
mkdir -p media/strm
```

目录说明：

```text
/opt/115ed/config.json       115ED 主配置
/opt/115ed/115ed.env         容器环境变量
/opt/115ed/logs              日志目录
/opt/115ed/media/strm        STRM 文件目录
```

## 三、创建配置文件

### 1. 创建 115ed.env

```bash
cd /opt/115ed
nano 115ed.env
```

写入：

```env
TZ=Asia/Shanghai
```

### 2. 创建 config.json

```bash
nano config.json
```

写入基础配置：

```json
{
  "emby_url": "",
  "emby_api_key": "",
  "cookie_115": "",
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
  "listen_port": ":6090",
  "strm_play_base_url": "",
  "request_delay": 1000,
  "admin_user": "admin",
  "admin_pass": "请改成强密码",
  "license_key": "",
  "license_server_url": "https://115.mdjp.eu.org/api/v1/check-heartbeat",
  "tg_bot_token": "",
  "tg_chat_id": "",
  "tg_api_url": "https://api.telegram.org",
  "notify_types": [
    "add",
    "delete",
    "play_start",
    "play_stop"
  ],
  "sync_schedule": "0 2 * * *",
  "sync_schedule_enabled": false,
  "sync_schedule_mode": "incremental",
  "mappings": []
}
```

首次部署至少需要关注这些字段：

```text
admin_user              后台登录账号
admin_pass              后台登录密码，必须修改
emby_url                Emby 地址，例如 http://emby:8096 或 http://服务器IP:8096
emby_api_key            Emby API Key
cookie_115              115 Cookie
strm_play_base_url      STRM 播放基地址，建议填播放器可访问的公网域名
license_key             授权码，没有就先留空
license_server_url      授权接口地址，按实际提供的地址填写
mappings                路径映射，也可以进后台页面添加
```

注意：

```text
config.json 里的敏感信息不要发给别人。
修改 config.json 后需要重启容器，或者在后台页面保存配置。
```

## 四、Docker Compose 部署，推荐

### 1. 创建 docker-compose.yml

```bash
cd /opt/115ed
nano docker-compose.yml
```

写入：

```yaml
services:
  115ed:
    image: tyer199/115ed:latest
    container_name: 115ed
    restart: unless-stopped
    env_file:
      - ./115ed.env
    environment:
      115ED_CONFIG: /app/config.json
      TZ: Asia/Shanghai
    ports:
      - "6090:6090"
    volumes:
      - ./config.json:/app/config.json
      - ./logs:/app/logs
      - ./media/strm:/opt/115kf/media/strm
```

如果你的镜像名不是 `tyer199/115ed:latest`，请改成实际镜像名。

### 2. 启动

```bash
cd /opt/115ed
docker compose up -d
```

如果你的环境使用旧版 Compose：

```bash
docker-compose up -d
```

### 3. 查看容器状态

```bash
docker ps
```

正常应能看到：

```text
115ed   Up ...   0.0.0.0:6090->6090/tcp
```

### 4. 查看日志

```bash
docker logs -f 115ed
```

### 5. 访问后台

浏览器打开：

```text
http://服务器IP:6090
```

或：

```text
http://服务器IP:6090/ui
```

默认账号密码来自 `config.json`：

```text
admin_user
admin_pass
```

## 五、docker run 部署

如果不用 Docker Compose，也可以使用 `docker run`：

```bash
docker run -d \
  --name 115ed \
  --restart unless-stopped \
  -e 115ED_CONFIG=/app/config.json \
  -e TZ=Asia/Shanghai \
  -p 6090:6090 \
  -v /opt/115ed/config.json:/app/config.json \
  -v /opt/115ed/logs:/app/logs \
  -v /opt/115ed/media/strm:/opt/115kf/media/strm \
  tyer199/115ed:latest
```

查看日志：

```bash
docker logs -f 115ed
```

停止：

```bash
docker stop 115ed
```

删除容器：

```bash
docker rm 115ed
```

## 六、后台基础配置说明

进入后台后，建议按顺序填写：

```text
1. Emby 地址
2. Emby ApiKey
3. 115 Cookie
4. STRM 播放基地址
5. 路径映射
6. 授权码
7. Telegram 通知，可选
8. 定时策略，可选
```

### 1. Emby 地址

如果 Emby 和 115ED 在同一个 Docker Compose 网络中，可以填容器名：

```text
http://emby:8096
```

如果 Emby 在宿主机或其他机器上：

```text
http://服务器IP:8096
```

### 2. Emby ApiKey

在 Emby 后台创建 API Key。

常见路径：

```text
Emby 管理后台 -> 高级 / API Keys
```

### 3. 115 Cookie

填写有效的 115 Cookie。

注意：

```text
Cookie 失效后需要重新填写。
不要把 Cookie 发给别人。
不要在公共群、截图、日志里暴露 Cookie。
```

### 4. STRM 播放基地址

这个字段非常重要。

它应该填写播放器可以访问的公网地址，例如：

```text
https://115ed.example.com
```

或者：

```text
http://服务器IP:6090
```

不要填：

```text
127.0.0.1
localhost
127.17.0.1
172.17.0.1
```

原因：

```text
127.0.0.1 / localhost 对手机、电视、播放器来说是它自己，不是你的 115ED。
172.17.0.1 是 Docker 网关，通常只有容器内部能访问。
STRM 文件最终是给播放器读的，所以要填播放器能访问的地址。
```

如果 Emby 容器访问公网域名超时，可以给 Emby 服务加 `extra_hosts`，让 Emby 容器内部把域名解析到宿主机 Docker 网关：

```yaml
services:
  emby:
    extra_hosts:
      - "115ed.example.com:172.17.0.1"
```

注意：

```text
extra_hosts 只写域名，不写 http:// 或 https://。
STRM 播放基地址仍然写公网域名。
```

### 5. 路径映射

路径映射用于告诉 115ED：

```text
115 云端 CID -> 本地 STRM 保存目录
```

示例：

```text
CID：2564183201492762537
本地路径：/opt/115kf/media/strm
```

如果使用本文 docker-compose.yml，本地路径建议填容器内路径：

```text
/opt/115kf/media/strm
```

因为 compose 里已经挂载：

```yaml
- ./media/strm:/opt/115kf/media/strm
```

宿主机实际目录是：

```text
/opt/115ed/media/strm
```

Emby 媒体库也需要能访问同一批 STRM 文件。

## 七、和 Emby 配合

### 1. Emby 媒体库路径

如果 Emby 也是 Docker 部署，建议让 Emby 挂载同一个宿主机目录。

示例：

```yaml
services:
  emby:
    image: amilys/embyserver:latest
    container_name: emby
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - ./emby/config:/config
      - /opt/115ed/media/strm:/strm
```

Emby 媒体库添加：

```text
/strm
```

115ED 路径映射填写：

```text
/opt/115kf/media/strm
```

这两个路径可以不同，但必须指向宿主机同一批 STRM 文件。

### 2. Emby 扫描

115ED 生成 STRM 后，需要 Emby 扫描媒体库。

可以在 Emby 后台手动扫描，也可以配置 Webhook 或定时任务。

## 八、Emby Webhook，可选

Webhook 地址：

```text
http://服务器IP:6090/api/emby/webhook
```

如果使用域名：

```text
https://115ed.example.com/api/emby/webhook
```

Emby 通知类型选择：

```text
Webhook
```

数据格式：

```text
默认 multipart/form-data
```

建议勾选：

```text
新片入库
媒体删除
播放开始
播放停止
```

## 九、反向代理和 HTTPS

生产环境建议使用域名和 HTTPS，例如：

```text
https://115ed.example.com
```

Nginx 反向代理到：

```text
http://127.0.0.1:6090
```

Nginx 示例：

```nginx
location / {
    proxy_pass http://127.0.0.1:6090;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 60s;
    proxy_send_timeout 600s;
    proxy_read_timeout 600s;
}
```

宝塔面板：

```text
1. 新建网站并绑定域名
2. 申请 SSL
3. 添加反向代理
4. 目标 URL 填 http://127.0.0.1:6090
```

## 十、常用命令

### 1. 进入目录

```bash
cd /opt/115ed
```

### 2. 启动

```bash
docker compose up -d
```

### 3. 停止

```bash
docker compose down
```

### 4. 重启

```bash
docker compose restart
```

### 5. 查看容器

```bash
docker ps
```

查看全部容器：

```bash
docker ps -a
```

### 6. 查看日志

实时日志：

```bash
docker logs -f 115ed
```

最近 100 行：

```bash
docker logs --tail=100 115ed
```

带时间：

```bash
docker logs -f --timestamps 115ed
```

### 7. 进入容器

```bash
docker exec -it 115ed sh
```

### 8. 查看端口

```bash
ss -lntp | grep 6090
```

### 9. 本机测试访问

```bash
curl -I http://127.0.0.1:6090
```

### 10. 容器内测试 Emby

```bash
docker exec -it 115ed sh -c 'wget -S -O- --timeout=5 http://emby:8096/System/Info/Public'
```

如果 Emby 不在同一个 Docker 网络，把地址换成你的 Emby 地址。

### 11. 更新镜像

```bash
cd /opt/115ed
docker compose pull
docker compose up -d
```

如果使用 `docker run`：

```bash
docker pull tyer199/115ed:latest
docker stop 115ed
docker rm 115ed
docker run -d \
  --name 115ed \
  --restart unless-stopped \
  -e 115ED_CONFIG=/app/config.json \
  -e TZ=Asia/Shanghai \
  -p 6090:6090 \
  -v /opt/115ed/config.json:/app/config.json \
  -v /opt/115ed/logs:/app/logs \
  -v /opt/115ed/media/strm:/opt/115kf/media/strm \
  tyer199/115ed:latest
```

### 12. 备份配置

```bash
mkdir -p /opt/115ed/backup
cp /opt/115ed/config.json /opt/115ed/backup/config.json.$(date +%F-%H%M%S)
cp /opt/115ed/115ed.env /opt/115ed/backup/115ed.env.$(date +%F-%H%M%S)
```

### 13. 备份全部部署目录

```bash
tar -czf /opt/115ed-backup.$(date +%F-%H%M%S).tar.gz -C /opt 115ed
```

### 14. 查看 STRM 文件数量

```bash
find /opt/115ed/media/strm -type f -name "*.strm" | wc -l
```

### 15. 查看空 STRM 文件

```bash
find /opt/115ed/media/strm -type f -name "*.strm" -size 0 -print
```

### 16. 检查某个 STRM 内容

```bash
cat "/opt/115ed/media/strm/影片名/影片名.strm"
```

## 十一、常见问题

### 1. 后台打不开

检查容器：

```bash
docker ps
docker logs --tail=100 115ed
```

检查端口：

```bash
ss -lntp | grep 6090
```

检查防火墙：

```bash
ufw status
ufw allow 6090/tcp
```

云服务器还需要检查安全组。

### 2. IP:6090 不能访问，但域名可以访问

可能端口只绑定到了 `127.0.0.1`，或者防火墙没有开放公网端口。

如果你只打算通过反向代理域名访问，这是可以接受的。

如果需要公网直接访问，compose 端口应为：

```yaml
ports:
  - "6090:6090"
```

### 3. 播放器无法播放

重点检查 STRM 内容：

```bash
find /opt/115ed/media/strm -name "*.strm" | head
cat "某个.strm"
```

STRM 里应该是播放器可访问的公网地址，例如：

```text
https://115ed.example.com/api/strm/play?pickcode=xxxxx
```

不要是：

```text
172.17.0.1
127.0.0.1
localhost
```

### 4. Emby 获取 PlaybackInfo 超时

可能是 Emby 容器访问 STRM 播放基地址失败。

如果 STRM 播放基地址是公网域名，但 Emby 容器访问这个域名会走公网回环，可以在 Emby compose 加：

```yaml
extra_hosts:
  - "115ed.example.com:172.17.0.1"
```

然后重启 Emby：

```bash
docker compose restart emby
```

### 5. 生成 STRM 数量和 Emby 入库数量不一致

常见原因：

```text
1. Emby 还没有完成媒体库扫描
2. 存在重复文件名，后生成的 STRM 覆盖了前一个
3. STRM 文件为空或路径错误
4. Emby 媒体库没有挂载到同一个目录
```

检查 STRM 数量：

```bash
find /opt/115ed/media/strm -type f -name "*.strm" | wc -l
```

检查空文件：

```bash
find /opt/115ed/media/strm -type f -name "*.strm" -size 0 -print
```

### 6. 115 Cookie 失效

表现：

```text
同步失败
取链失败
环境检测失败
播放失败
```

处理：

```text
重新获取 115 Cookie，在后台保存基础配置，然后重新同步。
```

### 7. 路径映射新增后没有保存

新增映射后一定要点击：

```text
保存映射关系
```

否则只是前端草稿，不会写入 `config.json`。

### 8. 修改配置后刷新页面

未保存的前端输入可能仍显示为草稿，并提示“有未保存变更”。

如果确认不要这些草稿，手动改回原值或重新登录后按实际页面状态处理。

## 十二、安全注意事项

1. `config.json` 包含 Emby ApiKey、115 Cookie、Telegram Token、授权码，不要公开。

2. 后台管理员密码必须修改，不要使用默认密码。

3. 生产环境建议使用 HTTPS 域名访问后台。

4. 如果只通过 Nginx 反代访问，可以把端口绑定到本机：

```yaml
ports:
  - "127.0.0.1:6090:6090"
```

5. 如果播放器需要直接访问 `6090`，就不能只绑定 `127.0.0.1`。

6. 不要把 `172.17.0.1` 写进给播放器使用的 STRM。

7. 更新镜像前先备份 `config.json`。

8. 不要随便执行 `docker system prune -a`，它可能删除你后续还要用的镜像。

## 十三、推荐最小部署流程

```bash
apt update
apt install -y curl wget ca-certificates
curl -fsSL https://get.docker.com | bash
systemctl enable docker
systemctl start docker

mkdir -p /opt/115ed/logs /opt/115ed/media/strm
cd /opt/115ed
nano 115ed.env
nano config.json
nano docker-compose.yml
docker compose up -d
docker logs -f 115ed
```

访问：

```text
http://服务器IP:6090
```

然后在后台完成：

```text
1. 填 Emby 地址和 ApiKey
2. 填 115 Cookie
3. 填 STRM 播放基地址
4. 添加路径映射
5. 保存配置
6. 执行全量同步
7. Emby 扫描媒体库
8. 测试播放
```
