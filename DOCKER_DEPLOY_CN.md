# Link115 Docker / Docker Compose 部署教程

本文面向普通用户，说明如何在 Linux 服务器上使用 Docker 或 Docker Compose 部署 Link115。

Link115 的作用是根据 115 网盘资源信息获取直链，并通过 HTTP 302 跳转让播放器直接访问 115/CDN 地址，减少服务器中转流量和 CPU 压力。

## 一、准备工作

### 1. 推荐环境

推荐使用以下系统：

```text
Debian 11 / Debian 12
Ubuntu 20.04 / 22.04 / 24.04
```

建议服务器配置：

```text
CPU：1 核或以上
内存：512 MB 或以上
磁盘：1 GB 可用空间以上
网络：能正常访问 115 网盘相关接口
```

### 2. 安装常用工具

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

检查 Docker 是否可用：

```bash
docker version
docker compose version
```

如果 `docker compose version` 不存在，可以尝试：

```bash
docker-compose version
```

下面的示例优先使用新版命令 `docker compose`。

## 二、创建部署目录

建议统一放在 `/opt/link115`：

```bash
mkdir -p /opt/link115
cd /opt/link115
```

创建配置和日志目录：

```bash
mkdir -p config logs
```

## 三、使用 Docker Compose 部署，推荐

### 1. 创建 docker-compose.yml

在 `/opt/link115` 目录创建文件：

```bash
nano docker-compose.yml
```

写入：

```yaml
services:
  link115:
    image: tyer199/115link:latest
    container_name: link115
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    ports:
      - "7080:7080"
    volumes:
      - ./config:/app/config
      - ./logs:/app/logs
```

说明：

```text
7080:7080 表示把容器内 7080 端口映射到服务器 7080 端口。
./config 用来保存配置。
./logs 用来保存日志。
restart: unless-stopped 表示容器异常退出或服务器重启后自动启动。
```

如果你的镜像名称不是 `tyer199/115link:latest`，请把 `image` 改成实际镜像名。

### 2. 启动服务

```bash
cd /opt/link115
docker compose up -d
```

查看容器状态：

```bash
docker ps
```

查看日志：

```bash
docker logs -f link115
```

### 3. 访问服务

浏览器访问：

```text
http://服务器IP:7080
```

如果服务器防火墙没有开放 7080，需要放行端口。

UFW 示例：

```bash
ufw allow 7080/tcp
ufw reload
```

云服务器还需要在安全组里放行 TCP 7080。

## 四、使用 docker run 部署

如果你不想使用 Docker Compose，也可以直接运行：

```bash
docker run -d \
  --name link115 \
  --restart unless-stopped \
  -e TZ=Asia/Shanghai \
  -p 7080:7080 \
  -v /opt/link115/config:/app/config \
  -v /opt/link115/logs:/app/logs \
  tyer199/115link:latest
```

查看日志：

```bash
docker logs -f link115
```

停止容器：

```bash
docker stop link115
```

删除容器：

```bash
docker rm link115
```

## 五、反向代理和域名访问

生产环境建议使用域名和 HTTPS，例如：

```text
https://link115.example.com
```

Nginx 反向代理到本机：

```text
http://127.0.0.1:7080
```

Nginx 示例：

```nginx
location / {
    proxy_pass http://127.0.0.1:7080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 60s;
    proxy_send_timeout 600s;
    proxy_read_timeout 600s;
}
```

如果使用宝塔面板：

```text
1. 新建网站，绑定域名
2. 开启 SSL
3. 网站设置里配置反向代理
4. 目标 URL 填 http://127.0.0.1:7080
```

## 六、和 Emby / STRM 配合时的注意事项

如果你把 Link115 作为 STRM 播放地址使用，STRM 里应该写播放器能访问的公网地址，例如：

```text
https://link115.example.com/api/strm/play?pickcode=xxxxx
```

不要把下面这类地址写进最终给手机、电视、播放器使用的 STRM：

```text
127.0.0.1
172.17.0.1
localhost
```

原因：

```text
127.0.0.1 / localhost 对播放器来说是播放器自己，不是你的服务器。
172.17.0.1 通常是 Docker 网关，只适合容器内部访问，手机和电视访问不到。
```

推荐方式：

```text
STRM 播放基地址填写公网域名或反代域名。
播放器访问公网域名。
如果 Emby 容器访问公网域名存在 NAT 回环问题，再用 extra_hosts 让 Emby 容器内部解析到宿主机网关。
```

示例：

```yaml
services:
  emby:
    image: amilys/embyserver:latest
    container_name: emby
    extra_hosts:
      - "link115.example.com:172.17.0.1"
```

注意：

```text
extra_hosts 里只写域名，不写 http:// 或 https://。
```

## 七、常用命令

### 1. 进入部署目录

```bash
cd /opt/link115
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

查看所有容器，包括已停止：

```bash
docker ps -a
```

### 6. 查看日志

实时日志：

```bash
docker logs -f link115
```

最近 100 行日志：

```bash
docker logs --tail=100 link115
```

带时间戳：

```bash
docker logs -f --timestamps link115
```

### 7. 进入容器

```bash
docker exec -it link115 sh
```

### 8. 查看端口监听

```bash
ss -lntp | grep 7080
```

### 9. 测试本机访问

```bash
curl -I http://127.0.0.1:7080
```

### 10. 更新镜像

```bash
cd /opt/link115
docker compose pull
docker compose up -d
```

如果使用 `docker run` 部署：

```bash
docker pull tyer199/115link:latest
docker stop link115
docker rm link115
docker run -d \
  --name link115 \
  --restart unless-stopped \
  -e TZ=Asia/Shanghai \
  -p 7080:7080 \
  -v /opt/link115/config:/app/config \
  -v /opt/link115/logs:/app/logs \
  tyer199/115link:latest
```

### 11. 删除旧镜像

查看镜像：

```bash
docker images
```

清理无用镜像：

```bash
docker image prune -f
```

清理无用容器、网络、镜像缓存：

```bash
docker system prune -f
```

谨慎使用 `docker system prune -a`，它会删除所有未被容器使用的镜像。

## 八、备份和恢复

### 1. 备份配置

```bash
mkdir -p /opt/link115/backup
tar -czf /opt/link115/backup/link115-config.$(date +%F-%H%M%S).tar.gz -C /opt/link115 config docker-compose.yml
```

### 2. 备份日志

```bash
tar -czf /opt/link115/backup/link115-logs.$(date +%F-%H%M%S).tar.gz -C /opt/link115 logs
```

### 3. 恢复配置

先停止服务：

```bash
cd /opt/link115
docker compose down
```

解压备份：

```bash
tar -xzf /opt/link115/backup/link115-config.备份时间.tar.gz -C /opt/link115
```

重新启动：

```bash
docker compose up -d
```

## 九、常见问题

### 1. 浏览器打不开 7080

检查容器是否启动：

```bash
docker ps
docker logs --tail=100 link115
```

检查端口：

```bash
ss -lntp | grep 7080
```

检查防火墙：

```bash
ufw status
```

放行端口：

```bash
ufw allow 7080/tcp
```

如果是云服务器，还要检查云厂商安全组。

### 2. 反代域名能访问，IP:7080 不能访问

先检查 Docker 端口绑定：

```bash
docker ps
ss -lntp | grep 7080
```

如果只监听在 `127.0.0.1:7080`，外部不能直接访问 IP:7080，这是正常现象。要么改成：

```yaml
ports:
  - "7080:7080"
```

要么继续只通过反向代理域名访问。

### 3. 播放器提示反代错误或超时

常见原因：

```text
1. 播放器访问的是 127.0.0.1、localhost、172.17.0.1 等内部地址。
2. 服务器防火墙或安全组没有开放端口。
3. 反向代理超时时间太短。
4. 115 Cookie 或取链参数失效。
5. 海外服务器访问 115/CDN 不稳定。
```

建议先在服务器上测试：

```bash
curl -I "http://127.0.0.1:7080/"
```

再在本地电脑或手机浏览器测试：

```text
http://服务器IP:7080
https://你的域名
```

### 4. Docker Compose 命令不可用

新版 Docker 使用：

```bash
docker compose up -d
```

旧版可能需要：

```bash
docker-compose up -d
```

如果两个都没有，重新安装 Docker：

```bash
curl -fsSL https://get.docker.com | bash
```

### 5. 容器一直重启

查看日志：

```bash
docker logs --tail=200 link115
```

检查配置目录权限：

```bash
ls -lah /opt/link115
ls -lah /opt/link115/config
ls -lah /opt/link115/logs
```

必要时修正权限：

```bash
chown -R root:root /opt/link115
chmod -R 755 /opt/link115
```

## 十、安全注意事项

1. 不要公开泄露 115 Cookie、账号、Token、配置文件。

2. 生产环境建议使用 HTTPS 域名访问，不建议长期裸露 IP 管理入口。

3. 如果服务只给本机其他容器访问，可以把端口绑定到本机：

```yaml
ports:
  - "127.0.0.1:7080:7080"
```

这样公网不能直接访问 7080，只能通过 Nginx 反代访问。

4. 如果需要公网播放器访问，STRM 里必须使用公网域名或公网 IP，不能使用 Docker 内网地址。

5. 修改配置前建议先备份：

```bash
tar -czf /opt/link115/backup-before-change.$(date +%F-%H%M%S).tar.gz -C /opt/link115 config docker-compose.yml
```

6. 不要随意执行网上复制的 `rm -rf`、`docker system prune -a` 等清理命令，避免误删数据。

## 十一、推荐部署流程汇总

最常用的一套命令如下：

```bash
apt update
apt install -y curl wget ca-certificates
curl -fsSL https://get.docker.com | bash
systemctl enable docker
systemctl start docker

mkdir -p /opt/link115/config /opt/link115/logs
cd /opt/link115
nano docker-compose.yml
docker compose up -d
docker logs -f link115
```

访问：

```text
http://服务器IP:7080
```

如果配置了反向代理：

```text
https://你的域名
```
