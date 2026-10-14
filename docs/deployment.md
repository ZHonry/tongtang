# 同堂 · 部署与运维

本文面向使用官方 Docker 镜像部署的用户。镜像支持 `linux/amd64` 与 `linux/arm64`，
内容为编译产物（不含可读源码）。

## 系统要求

- 任何能跑 Docker 的环境：NAS（TrueNAS / 群晖 / UNRAID）、x86 小主机、树莓派 4 以上
- 资源占用极低：约 150MB 内存；数据库为单文件 SQLite
- 一个或多个可访问的 Home Assistant 实例及其**长期访问令牌**（HA → 个人资料 → 安全 → 创建长期访问令牌）

## 部署

两种方式二选一，compose 文件在仓库根目录：

**一体化单容器（最简，推荐个人使用）**

```bash
docker compose -f docker-compose.allinone.yml up -d
```

**双容器（Nginx 前端 + FastAPI 后端）**

```bash
docker compose up -d
```

不想用 compose 的话，一行 docker run 也可以：

```bash
docker run -d --name tongtang --restart unless-stopped \
  -p 8080:8000 \
  -e APP_SECRET="$(openssl rand -hex 32)" \
  -e TZ=Asia/Shanghai \
  -v tongtang_data:/data \
  jeesa/tongtang:latest
```

### 环境变量

| 变量 | 说明 |
|---|---|
| `APP_SECRET` | **建议必填**，≥16 位随机串（`openssl rand -hex 32`）。会话签名密钥：留空每次重启生成一次性密钥（全员重新登录）；改动会使全员重新登录 |
| `INITIAL_ADMIN_PASSWORD` | 首次建库时管理员初始密码（仅空库生效，默认 `admin123`） |
| `HA_URL` / `HA_TOKEN` | 可选。设置后**优先于界面配置**（向导会显示"已由环境变量提供"）；留空走网页首次配置向导。修改后需 `docker compose up -d` 重建容器才生效；令牌失效时请更新此处而非向导 |
| `TZ` | 时区，默认 `Asia/Shanghai` |
| `PUID` / `PGID` | 可选。数据目录属主修正后以此身份运行（默认 10001）；设为你的 `id -u`/`id -g` 便于在宿主机直接读写数据文件 |
| `SESSION_COOKIE_NAME` | 可选。会话 Cookie 名（默认 `tongtang_session`，2.3.2 之前为 `qj_session`）。同一主机跑多套实例（不同端口）时各配一个，避免互相挤登出（浏览器 Cookie 不区分端口） |
| `DATA_DIR` | 容器内数据目录，保持默认 `/data` 即可 |

### 数据目录

默认使用命名卷 `tongtang_data`。也可以绑定宿主机目录：

```yaml
    volumes:
      - /srv/tongtang/data:/data
```

容器启动时会自动把数据目录属主修正为运行用户，**无需手动 chown**。

### 首次配置

打开 `http://服务器IP:8080` 进入首次配置向导：填 HA 地址（如 `http://192.168.x.x:8123`）、
长期访问令牌、管理员账号密码；系统自动测试连接并同步全部受支持的实体与房间。
HA 令牌只保存在服务端数据库中，前端与普通用户永远接触不到。

## 升级

当前正式版本为 **2.4.24**。一体化、web、api与Matter侧车均提供同版本标签，支持linux/amd64与linux/arm64；HomeKit侧车复用API镜像。一体化部署给以下命令增加 `-f docker-compose.allinone.yml`，静态侧车随同一套compose更新。

保留既有数据卷、APP_SECRET及网络/端口配置，升级前备份数据；请勿删除数据卷。可将compose中的镜像固定为 `:2.4.24`，避免后续latest变化。

```bash
docker compose pull && docker compose up -d
```

数据库结构变更会在启动时自动迁移，无需手工操作。升级后浏览器强刷一次
（Service Worker 会自动接管新版本）。版本变更见 [CHANGELOG](../CHANGELOG.md)。

## 备份与恢复

数据全部在 `/data` 的单个 SQLite 库中（WAL 模式，注意连同 `-wal`/`-shm` 一起备份）：

```bash
# 备份
docker compose exec api sh -c "cd /data && tar cf - *.db*" > tongtang-backup-$(date +%F).tar
# （一体化部署把 api 换成 tongtang）

# 恢复
docker compose stop api
cat tongtang-backup-xxxx.tar | docker compose run --rm -T api sh -c "cd /data && tar xf -"
docker compose start api
```

配置级备份也可用 `GET /api/admin/backup`（管理员登录后访问）。
绑定宿主机目录部署的话，直接备份该目录同样可行（建议先停容器保证一致性）。

## HTTPS（强烈建议公网访问时启用）

在前面加一层你现有的反代（Nginx Proxy Manager / Caddy / Traefik）指向映射端口：

- 必须转发 WebSocket：`/api/ws` 需要 `Upgrade/Connection` 头（NPM 勾选 WebSockets Support 即可）
- 摄像头流 `/api/cameras/*/stream` 建议关闭代理缓冲
- HTTPS 下会话 Cookie 自动带 `Secure` 标记

## 安全清单

- [ ] `APP_SECRET` 已设为随机值
- [ ] 公网访问走 HTTPS
- [ ] 管理员密码 ≥8 位且非默认
- [ ] 审计保留时长按需设置（默认 90 天）
- [ ] 登录接口自带按 IP+账号限速（15 分钟 10 次失败锁定）

## 故障排查

**症状：同一台主机跑两套实例（不同端口），登录一个另一个被挤下线**
浏览器 Cookie 只按域名隔离、不区分端口，两套实例默认共用同名会话 Cookie 互相覆盖。
给其中一套设置 `SESSION_COOKIE_NAME`（如 `tt_test_session`）即可共存。

**症状：换成宿主机目录（bind mount）后 API 起不来**
旧版镜像（< 1.1.1）对 root 属主的宿主目录无写权限。升级到最新镜像即可（启动时自动修正属主）；
或对旧镜像手动执行一次 `chown -R 10001:10001 <宿主目录>`。

**症状：API 时好时坏 / 页面转圈**
若 Docker 网络启用了 IPv6，Nginx 可能解析到后端的 v6 地址而后端只监听 v4。
本仓库 compose 已配置 `enable_ipv6: false`；自建网络时请保持。改动网络配置需 `docker compose down && up -d`。

**症状：实时推送不工作**
`docker compose logs api | grep 已订阅` 应看到每个启用实例各一行订阅日志；
无则检查 HA 地址可达性与令牌有效性（监听器每 30 秒自动对账重连）。

**症状：能源页历史为零（实时数值正常）**
能源历史走 HA 长期统计：实体需带 `state_class` 属性（如 `total_increasing`）HA 才会记录统计。
在 HA 开发者工具查看实体属性，缺失则在集成侧补上（ESPHome 需刷新固件配置）。

**症状：前端改了没生效 / 升级后界面没变**
Service Worker 缓存所致：强刷（Cmd/Ctrl+Shift+R），或开发者工具 → Application → Service Workers → Unregister。

**症状：附加 HA 实例连不上**
在部署机上直接 `curl -H "Authorization: Bearer <token>" http://<ha>:8123/api/config`
验证网络与令牌；跨网段需保证路由可达。

**症状：设备识别不对（存在传感器变开关卡等）**
见 [设备识别对照表](device-recognition.md)，按文档在 HA 侧修正 `device_class` 等属性后重新同步。
