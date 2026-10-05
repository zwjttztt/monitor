# monitor

文档：[monitor-document.pages.dev](https://monitor-document.pages.dev)，安装、配置、反向代理与主题开发都在这里。

## 特性

- 实时监控：秒级实时数据展示
- 轻量高效：Rust 语言构建，低资源占用，极简高效
- 自托管：完全掌控数据隐私，部署简单
- 通知：节点掉线、流量、到期与登录，推送到 Telegram 或自定义 Webhook

## 组成

| 仓库 | 说明 |
|---|---|
| [monitor](https://github.com/monitor-probe/monitor) | hub：后台、API、公开页宿主 |
| [agent](https://github.com/monitor-probe/agent) | Linux agent |
| [monitor-theme-default](https://github.com/monitor-probe/monitor-theme-default) | 内置默认主题 |

```
agent (Linux)  ──WebSocket / JSON-RPC 2.0──▶  hub (axum + SQLite)  ──▶  后台 + 状态页
```

## 本仓库的改动

在 `monitor-probe/monitor` 基础上做了两处改动，部署时需要注意的差异都写在这里。

| 改动 | 说明 |
|---|---|
| 到期时间精确到分钟 | 节点的到期时间从「只到天」升级为 `YYYY-MM-DD HH:MM`，续费设置里可填到分钟。库里统一存这个格式，浏览器用 `datetime-local`。**旧的纯日期值仍然可读**，按当天 00:00 处理，无需迁移。 |
| 操作栏「已续费」按钮 | 节点列表操作栏新增按钮，点击按该节点的付款周期顺延一个周期，保留时分（1 月 10 日 08:32 按月付 → 2 月 10 日 08:32）。已过期或从未设置到期时间的，从当前时刻起算；一次性付款（`once`）没有周期，按钮禁用。 |

新增的接口是 `POST /api/nodes/{id}/renew`，与 agent 无关，旧版 agent 照常工作。

## Docker 部署

### 为什么需要额外的 Dockerfile

仓库根目录的 `Dockerfile` **不能从源码构建**。它只把两个预编译的 musl 二进制
（`monitor-hub-amd64` / `monitor-hub-arm64`）复制进 `scratch`，而根 `.dockerignore`
的白名单也只放行这两个文件——它们由 `release.yml` 用 `cross` 编译后上传为
artifact。所以照原样 `docker build .` 会得到一个缺文件的镜像，或者干脆找不到
Dockerfile 上下文。

从源码构建走 `Dockerfile.source`，配套的 `Dockerfile.source.dockerignore` 只在
`-f Dockerfile.source` 时生效，两个文件互不干扰，发布流程不受影响。

### 构建并启动

```bash
git clone https://github.com/zwjttztt/monitor.git
cd monitor

# 构建。慢在 cargo：235 个依赖，且 release profile 开了 lto + codegen-units=1。
# 2 核小机器上实测 1.5 小时以上，官方 runner 约 10 分钟。Docker 层有缓存，
# 第二次只重跑改动之后的阶段。
docker build -f Dockerfile.source -t monitor-hub:local .

# 启动
docker compose up -d
```

`docker-compose.yml` 已经配好：数据在命名卷 `monitor-data`（挂到 `/data`），
端口只开在宿主机回环 `127.0.0.1:28080`，内存上限 256M。

```bash
docker compose logs -f          # 首次启动会打印一次性应急密码
docker compose ps
```

首次启动日志里那行 `Emergency password: ...` 只出现一次，登录后到「安全」里改掉。

### 换端口 / 对外暴露

改 `docker-compose.yml` 的 `ports`。想让公网直接访问面板，把
`127.0.0.1:28080:28080` 改成 `28080:28080`——但这样面板密码和 agent token
会在链路上裸奔，**更推荐配反向代理**（nginx / caddy / CF Tunnel），反代配法见
[官方文档](https://monitor-document.pages.dev/install/reverse-proxy)。反代与 hub
同机时保留 `127.0.0.1` 那行即可。

### 环境变量

| 变量 | 说明 |
|---|---|
| `TZ` | 时区，默认 `UTC`。流量与账期都以本地日期为界，跨时区看面板会对不上账。chrono's `Local` 直接读它，镜像里也带了 zoneinfo，所以改这个就够。 |
| `MONITOR_LOG` | 日志过滤，如 `monitor_hub=info,tower_http=warn`。 |

`--site`（面板对外地址）**没有对应的环境变量**，只能走命令行，在 compose 里用
`command:` 追加：

```yaml
    command:
      - /monitor-hub
      - --db
      - /data/monitor.db
      - --site
      - https://monitor.example.com
```

三种情况需要它：节点连的域名与进面板的域名不同；通过 SSH 隧道进面板（回环地址
节点够不着）；反代不发 `X-Forwarded-Proto`（这时会话 cookie 拿不到 `Secure` 标志）。
必须是 `https://` 开头的域名，填错会导致无法添加节点。

### 从旧版本升级

数据库 schema 由 hub 自己管理（`PRAGMA user_version`，当前 12），启动时在一个事务里
自动迁移，失败会整个回滚、不会停在中间状态。**不要手工改数据库文件。**
本次改动没有碰 schema（`expires_at` 本来就是 TEXT，只是内容格式更细），旧库可直接用。

```bash
docker compose down
docker build -f Dockerfile.source -t monitor-hub:local .   # 拉新代码后重建（只重跑受影响阶段）
docker compose up -d
```

卷不动，节点数据、主题、后台设置全部保留。

**降级要当心**：迁移只保证向前兼容。旧版 hub 跑在已迁移的文件上未必能正常启动，
所以回滚前先导出备份（见下），别只靠镜像 tag 回退。

### 备份

数据库是 SQLite（WAL 模式），运行中直接复制 `monitor.db` 会丢掉预写日志里还没落盘的
那部分数据——面板自己也这么提示。**首选面板内的「导出备份 / 导入备份」**（设置页
「备份」卡片）：走 SQLite 在线备份 API 导出一致快照，恢复时整体覆盖。
备份文件含节点凭证与登录密码哈希，请当密钥保管。

必须停机手工备份时，`-wal`、`-shm` 要一起复制：

```bash
docker compose down
docker run --rm -v monitor-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/monitor-$(date +%Y%m%d).tar.gz -C /data .
docker compose up -d
```

### 常见问题

| 现象 | 原因 |
|---|---|
| `folder ".../web-admin/dist" does not exist` | 面板没先构建。`Dockerfile.source` 已处理：面板阶段在前，`dist/` 复制进 Rust 阶段后才跑 cargo。 |
| 流量和到期时间在错误的时刻翻页 | 时区。镜像里带 zoneinfo，删掉它 chrono 会静默退回 UTC。确认 `TZ` 有设置。 |
| 容器起不来，`exec: "monitor-hub": permission denied` | 二进制丢了执行位。用本仓库的 `Dockerfile.source` 不会出现这个问题（stage 之间复制会保留权限）。 |
| `docker exec` 报 `exec format error` 或没有 shell | 正常。镜像 `FROM scratch`，里面没有 shell 也没有 `curl`，无法进容器排查。看 `docker compose logs`。 |
| agent 装不上、面板提示域名不对 | `--site` 没传或传错，它必须是 `https://` 开头的域名。注意它**只能走命令行**，没有环境变量。 |

