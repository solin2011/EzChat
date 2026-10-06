# EzChat（易信）· 服务端部署指南

> 项目：易信 / EzChat　团队：EzSoft（氢易软件）　服务端：`server/`（Go，module `ezchat.im/server`）
> 目标系统：Ubuntu 22.04 LTS / 24.04 LTS（x86_64 或 arm64）
> 基线硬件：2 vCPU / 2 GB 内存 / 香港节点（低配优化见第八节）
> 本文随版本迭代，如有改动请同步更新。

---

## 目录

1. [适用范围与前置条件](#一适用范围与前置条件)
2. [环境准备](#二环境准备)
3. [二进制部署](#三二进制部署)
4. [config.yaml 配置说明](#四configyaml-配置说明)
5. [Docker 部署](#五docker-部署)
6. [Nginx 反向代理](#六nginx-反向代理)
7. [域名解析与证书](#七域名解析与证书)
8. [2C2G 香港低配优化建议](#八2c2g-香港低配优化建议)
9. [日志与备份](#九日志与备份)
10. [升级与回滚流程](#十升级与回滚流程)
11. [常见问题排查](#十一常见问题排查)
12. [部署检查清单](#十二部署检查清单)

---

## 一、适用范围与前置条件

### 1.1 适用场景

* 单机部署 EzChat 服务端（Go 二进制或 Docker 容器）+ Nginx 反向代理；
* 同时承载 API（`api.`）、WebSocket 长连接（`ws.`）、媒体上传下载（`file.`）、
  开放平台（`open.`）、管理端（`admin.`）、审核端（`review.`）、
  文档站（`docs.`）、状态页（`status.`）、下载站（`dl.`）、邮件（`mail.`）。
* **根域名 `ezchat.im` 保持不动**：本指南与仓库里的配置文件都不修改根域名的解析与站点，
  它继续由现有主站负责。

### 1.2 前置条件

| 项目 | 要求 |
|---|---|
| 系统 | Ubuntu 22.04 LTS 或 24.04 LTS（全新安装或干净环境） |
| 架构 | `x86_64`（amd64）或 `aarch64`（arm64） |
| 权限 | 具备 `sudo` 权限的账号 |
| 域名 | 已注册 `ezchat.im`，且能修改 DNS 解析 |
| 出口 | 能访问 GitHub（下载构建产物）、`letsencrypt.org`（签发证书） |
| 端口 | 对外开放 80、443；SSH 端口按需收敛。**服务端自身监听 8080，不对公网开放** |
| 依赖 | 无。服务端使用 SQLite（单文件数据库），无需额外数据库服务 |

### 1.3 部署方式怎么选

| 方式 | 适合 | 优点 | 缺点 |
|---|---|---|---|
| **二进制 + systemd**（推荐） | 生产环境、低配服务器 | 无运行时依赖、内存开销最小、与 systemd 安全加固配合最好、升级回滚最简单 | 需要手工放置文件 |
| **Docker Compose** | 希望环境隔离、快速迁移、多机一致 | 依赖封装在镜像内、编排（含 Nginx/Certbot）一键起停 | 多一层运行时开销（2C2G 上更明显）、卷与权限需额外注意 |

> 2C2G 的香港小机器上，**优先选二进制 + systemd**。

---

## 二、环境准备

> 以下命令除特别说明外均在**服务器**上以 `sudo` 执行。

### 2.1 系统更新与时区

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install curl wget ca-certificates gnupg lsb-release \
                    jq unzip zip tar sqlite3 rsync logrotate ufw

# 业务时间戳统一按 GMT+8（Asia/Shanghai）对齐
sudo timedatectl set-timezone Asia/Shanghai
timedatectl status
```

### 2.2 创建专用系统用户与目录

服务端**不以 root 运行**。创建不可登录的系统用户 `ezchat`，并建立统一目录布局：

```bash
# 系统用户（无 shell、无家目录）
sudo addgroup --system ezchat
sudo adduser  --system --ingroup ezchat --no-create-home \
              --home /opt/ezchat --shell /usr/sbin/nologin ezchat

# 目录布局
sudo mkdir -p /opt/ezchat/{bin,configs,data,uploads,logs,backups,tmp}
sudo mkdir -p /opt/ezchat/www/{dl,docs,status,admin,review,mail}

# 属主：程序目录只读属于 root，数据目录属于 ezchat
sudo chown -R root:ezchat   /opt/ezchat/bin
sudo chown -R ezchat:ezchat /opt/ezchat/{configs,data,uploads,logs,backups,tmp}
sudo chown -R root:root     /opt/ezchat/www

# 权限：数据与配置不对其他用户开放
sudo chmod 0755 /opt/ezchat /opt/ezchat/bin
sudo chmod 0750 /opt/ezchat/configs
sudo chmod 0700 /opt/ezchat/{data,uploads,logs,backups,tmp}
```

> ⚠️ `deploy/systemd/ezchat-server.service` 里用 `ProtectSystem=strict` 把整个文件系统设为只读，
> 只有 `ReadWritePaths` 列出的目录可写。**如果你把数据目录改到 `/opt/ezchat` 之外
> （例如挂载了一块独立数据盘 `/data/ezchat`），必须同步修改单元文件的 `ReadWritePaths`**，
> 否则服务会因为 `Read-only file system` 启动失败。

### 2.3 系统参数调优

长连接服务需要大量文件描述符与更大的网络队列。写入 sysctl：

```bash
sudo tee /etc/sysctl.d/99-ezchat.conf >/dev/null <<'EOF'
# ---- 文件描述符 ----
fs.file-max = 2097152
fs.nr_open  = 2097152

# ---- 连接队列与端口范围 ----
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_local_port_range = 10240 65000

# ---- TIME_WAIT 复用（长连接服务大量短连接握手时收益明显）----
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_max_tw_buckets = 262144

# ---- TCP 缓冲区 ----
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# ---- 保活：及早发现死连接，释放 fd ----
net.ipv4.tcp_keepalive_time   = 300
net.ipv4.tcp_keepalive_intvl  = 30
net.ipv4.tcp_keepalive_probes = 5

# ---- 低配机器防 SYN Flood ----
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 3
EOF

sudo sysctl --system
```

`/etc/security/limits.conf` 放宽单用户句柄上限：

```bash
sudo tee /etc/security/limits.d/99-ezchat.conf >/dev/null <<'EOF'
ezchat  soft  nofile  1048576
ezchat  hard  nofile  1048576
ezchat  soft  nproc   65535
ezchat  hard  nproc   65535
EOF
```

> 单元文件里已经写了 `LimitNOFILE=1048576`，systemd 服务以单元文件的值为准；
> 这里的 `limits.conf` 是为了你在服务器上手工以 `ezchat` 用户调试时也生效。

### 2.4 防火墙

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp      comment 'SSH'
sudo ufw allow 80/tcp      comment 'HTTP (ACME + redirect)'
sudo ufw allow 443/tcp     comment 'HTTPS'
# 8080 只给本机 Nginx 用，绝不对外开放
sudo ufw deny 8080/tcp
sudo ufw enable
sudo ufw status verbose
```

> ⚠️ 云服务器（阿里云/腾讯云/华为云/AWS/Azure）还有一层**安全组**，需要在控制台
> 单独放行 80/443。香港节点通常不需要备案，但大陆访问质量与线路（CN2 GIA 等）相关。

### 2.5 可选：fail2ban 防爆破

```bash
sudo apt -y install fail2ban
sudo tee /etc/fail2ban/jail.d/ezchat.conf >/dev/null <<'EOF'
[sshd]
enabled  = true
maxretry = 5
bantime  = 3600
findtime = 600
EOF
sudo systemctl enable --now fail2ban
```

> 注意：服务端自身还有 `security` 分节的限流与黑名单（见第四节），
> fail2ban 只兜 SSH 与 Nginx 日志层面的爆破。

---

## 三、二进制部署

### 3.1 获取产物

服务端二进制来自 GitHub Actions 的 `build-server.yml`（手动触发）。
产物命名规则：

| 目标 | Artifact 名 | 内部文件 |
|---|---|---|
| Linux x86_64 | `ezchat-server-<版本>-linux-amd64` | `ezchat-server` + `ezchat-server.sha256` |
| Linux arm64 | `ezchat-server-<版本>-linux-arm64` | `ezchat-server` + `ezchat-server.sha256` |
| 全平台校验和 | `ezchat-server-<版本>-SHA256SUMS` | `SHA256SUMS.txt` |

下载（`gh` CLI 示例）：

```bash
# 在本地或服务器上执行
gh run list --workflow=build-server.yml --limit 5
gh run download <run-id> -n ezchat-server-1.0.0-linux-amd64 -D ./ezchat-1.0.0
```

### 3.2 上传并校验（**必做**）

```bash
# 在本地：上传到服务器
scp -r ./ezchat-1.0.0/* <user>@<服务器IP>:/tmp/ezchat-1.0.0/

# 在服务器：校验 sha256（下载到的 .sha256 文件内容形如
#   <hash>  ezchat-server
# 若文件里的路径与本地不一致，用下面的等价命令手工比对）
cd /tmp/ezchat-1.0.0
sha256sum -c ezchat-server.sha256

# 或者用 Release 里的统一 SHA256SUMS.txt
# grep ' ezchat-server$' SHA256SUMS.txt | sha256sum -c -

# 查看二进制信息
file ezchat-server
./ezchat-server --version 2>/dev/null || true     # 若支持 --version 会打印注入的版本号
```

> **不要跳过校验。** 二进制是从网络上传输过来的，`sha256sum -c` 能确认文件没有损坏或被篡改。

### 3.3 安装文件

```bash
sudo install -m 0755 -o root  -g root   /tmp/ezchat-1.0.0/ezchat-server /opt/ezchat/bin/ezchat-server

# 配置文件：从仓库样例复制（样例文件是 server/configs/config.example.yaml）
# 部署机上没有仓库时，先把样例和二进制一起 scp 到 /tmp
sudo install -m 0640 -o ezchat -g ezchat /tmp/config.example.yaml /opt/ezchat/configs/config.yaml
sudoedit /opt/ezchat/configs/config.yaml       # 按第四节逐项填写

# 备份一份当前版本，便于回滚
sudo cp /opt/ezchat/bin/ezchat-server /opt/ezchat/bin/ezchat-server.bak
ls -l /opt/ezchat/bin/
```

> **运行期读取的文件名就是 `config.yaml`**。
> 仓库里的样例是 `server/configs/config.example.yaml`，部署时复制并重命名为 `config.yaml`。

### 3.4 先用前台方式试跑（重要）

在装 systemd 之前，先手工确认配置能被正确读取、端口能正常绑定：

```bash
sudo -u ezchat env TZ=Asia/Shanghai \
  /opt/ezchat/bin/ezchat-server --config /opt/ezchat/configs/config.yaml
```

看到启动日志且没有报错后，`Ctrl+C` 停掉，再进入下一步。

> ⚠️ **需要你确认的一点**：本指南与 `deploy/systemd/ezchat-server.service` 都假定服务端支持
> `--config <路径>` 参数。如果实际实现用的是 `-c` / `-conf` / 环境变量，
> 或者只读取「工作目录下的 `config.yaml`」，请把命令与单元文件里的 `ExecStart`
> 改成实际形式（例如仅写 `/opt/ezchat/bin/ezchat-server`，
> 并把 `WorkingDirectory` 调整到 `config.yaml` 所在目录）。
>
> 补充：样例配置说明里写明**任何字段都可用 `EZCHAT_<SECTION>_<FIELD>` 环境变量覆盖**
> （例如 `EZCHAT_SERVER_PORT=8080`、`EZCHAT_AUTH_JWT_SECRET=xxx`）。
> 因此即便不方便改配置文件，也可以用 systemd 的 `Environment=` / `EnvironmentFile=`
> 或 Docker 的 `environment:` 覆盖关键项。**但不要用环境变量传递含特殊字符的密钥时忘记转义。**

### 3.5 安装 systemd 单元并启动

单元文件由仓库提供：`deploy/systemd/ezchat-server.service`。

```bash
sudo install -m 0644 deploy/systemd/ezchat-server.service \
     /etc/systemd/system/ezchat-server.service

# 若你的数据目录不在 /opt/ezchat 下，先改 ReadWritePaths：
sudoedit /etc/systemd/system/ezchat-server.service

sudo systemctl daemon-reload
sudo systemctl enable --now ezchat-server

# 状态与日志
sudo systemctl status ezchat-server --no-pager
sudo journalctl -u ezchat-server -f
sudo journalctl -u ezchat-server --since "10 min ago" --no-pager
```

看到 `active (running)` 即启动成功。验证监听：

```bash
ss -lntp | grep 8080
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/healthz || true
```

### 3.6 单元文件里的安全加固项说明

`deploy/systemd/ezchat-server.service` 已经包含了生产可用的加固配置，逐项含义：

| 配置项 | 作用 |
|---|---|
| `NoNewPrivileges=true` | 禁止进程及其子进程通过 setuid/setgid 提权 |
| `CapabilityBoundingSet=` / `AmbientCapabilities=` | 丢弃全部 Linux capabilities。**因此服务端不能直接绑定 80/443**，必须由 Nginx 反代（这正是我们的架构） |
| `PrivateTmp=true` | 给服务一份独立的 `/tmp`，与其他进程隔离，避免符号链接攻击 |
| `PrivateDevices=true` | 不暴露物理设备节点 |
| `ProtectSystem=strict` | 整个文件系统对服务只读 |
| `ProtectHome=true` | `/home`、`/root`、`/run/user` 不可见 |
| `ReadWritePaths=...` | **唯一**允许写入的目录白名单。数据目录换了位置就必须改这里 |
| `ProtectKernelTunables/Modules/Logs`、`ProtectControlGroups`、`ProtectClock`、`ProtectHostname` | 禁止触碰内核参数、模块、日志、cgroup、时钟、主机名 |
| `ProtectProc=invisible` + `ProcSubset=pid` | `/proc` 只看到自己的进程，防止探测宿主 |
| `SystemCallFilter=@system-service` + `SystemCallErrorNumber=EPERM` | 只允许常规服务类系统调用，越权返回 EPERM |
| `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX` | 只允许 TCP/UDP/Unix socket |
| `RestrictNamespaces` / `RestrictRealtime` / `RestrictSUIDSGID` / `LockPersonality` / `RemoveIPC` | 常规纵深防御 |
| `MemoryDenyWriteExecute=true` | 禁止可写可执行内存（防 JIT 型攻击）。**若启用了 cgo 或 plugin 需改成 `false`** |
| `UMask=0077` | 新建文件默认不对同组/其他用户开放 |
| `Restart=always` + `RestartSec=3` | 崩溃后自动拉起 |
| `StartLimitIntervalSec=60` + `StartLimitBurst=5` | 60 秒内最多重启 5 次，避免疯狂重启刷爆日志 |
| `LimitNOFILE=1048576` | 长连接（WebSocket）需要大量 fd |
| `MemoryHigh=1200M` / `MemoryMax=1500M` / `MemorySwapMax=512M` / `CPUQuota=180%` | 资源上限，防止单个服务拖垮 2C2G 小机器 |
| `OOMScoreAdjust=-200` | 内存紧张时优先牺牲其他进程，保住聊天服务 |
| `KillMode=mixed` + `TimeoutStopSec=30` | 先 SIGTERM 优雅退出（把 SQLite WAL 落盘、关闭长连接），超时再 SIGKILL |

修改单元文件后：

```bash
sudo systemctl daemon-reload && sudo systemctl restart ezchat-server
sudo systemd-analyze security ezchat-server     # 查看加固评分（越小越好）
```

### 3.7 放置 Web 静态产物（可选）

用户端 `/ 管理端 / 审核端的 Web 产物（`ezchat-app-<版本>-web.zip` 等）解压到：

```bash
sudo mkdir -p /opt/ezchat/www/{dl,docs,status,admin,review,mail}
sudo unzip ezchat-app-1.0.0-web.zip -d /opt/ezchat/www/app
sudo chown -R root:root /opt/ezchat/www
```

对应的 Nginx `root` 路径见 `deploy/nginx/ezchat.conf`（标注为「★ 需替换」的地方）。

---

## 四、`config.yaml` 配置说明

> 服务端**运行期读取的文件名是 `config.yaml`**，样例为 `server/configs/config.example.yaml`。
> 下面是按已约定分节整理的说明。真正的字段全集以
> `server/configs/config.example.yaml` 为准 —— 下面这份 YAML 示例用于
> 说明每个分节要配置什么，**字段名请与样例文件逐字核对后再落盘**。

### 4.1 分节总览

> **重要：本项目的配置分两层**（见 ADR D-019）——
>
> * **启动期配置**：就是本文这一节的 `config.yaml`，只放监听地址、路径、密钥、连接池、外部服务地址。
>   改完需要重启服务。
> * **运行期设置**：注册开关、维护模式、**保留天数（文本/媒体/表情包）**、限流阈值、
>   外链黑白名单、邮件模板等，存放在**数据库的 `system_settings` 表**中，
>   由 **EzChat Admin Desktop 实时修改，不需要重启**，**不写在 `config.yaml` 里**。
>
> `config.yaml` 里与运行期设置重叠的字段（如 `retention.*`、`security.rate_limit_enabled`、
> `features.*`）只是**启动期兜底值**，运行期以数据库中的值为准。
>
> 另外：**任何字段都可以用环境变量覆盖**，规则是 `EZCHAT_<SECTION>_<FIELD>`，例如
> `EZCHAT_SERVER_PORT=8080`、`EZCHAT_AUTH_JWT_SECRET=xxx`、`EZCHAT_LOGGING_LEVEL=debug`。
> 这让 systemd / Docker 部署时可以在不改文件的情况下覆盖敏感项。

| 分节 | 管什么 | 关键点 |
|---|---|---|
| `server` | 监听地址 `host`、`port`、对外基址 `public_base_url`、`mode`、优雅关闭超时、是否信任反代头、未鉴权并发上限 | `host` 前置 Nginx 时应改 `127.0.0.1`；`public_base_url` 必须与 Nginx 一致（用于生成二维码/邮件链接） |
| `domains` | 域名矩阵：`root` / api / ws / file / open / admin / review / docs / status / dl / mail，外加 `cors_origins` | 客户端从 `/v1/public/config` 拉取这些地址；服务端据此生成 CORS 白名单 |
| `database` | SQLite `path`、只读连接池 `max_read_conns`、`busy_timeout_ms`、`wal_autocheckpoint`、`cache_size_kb`、`write_queue_size`、迁移目录与 `auto_migrate` | 数据目录必须在本机 SSD 上，**禁止放 NFS/SMB**（WAL 会失效甚至损坏库） |
| `storage` | 媒体根目录 `root`、`temp_root`、`backup_root`、`backend`、**单文件上限 `max_file_size_mb`**、分片阈值/大小、签名直链有效期、`s3` 段 | `root` 要与 systemd `ReadWritePaths`、Nginx `alias` 三处一致；`max_file_size_mb` 要与 Nginx `client_max_body_size` 一致 |
| `retention` | 启动期兜底：`enabled`、`cleanup_interval_minutes`、`cleanup_batch_size`、**磁盘保护 `min_free_disk_mb`**、`disk_guard_action`（stop/readonly/aggressive_cleanup）、`max_run_seconds` | **文本 180 天 / 媒体 14 天 / 表情包永久**这些天数在数据库的运行期设置里，不在此文件 |
| `auth` | `jwt_secret`、`jwt_issuer`、`access_token_minutes`、`refresh_token_days`、**单点登录槽位 `slots`**、扫码登录票据有效期、Argon2id 参数 | `jwt_secret` 必须换成随机值；`slots` 决定每种端各允许一个活跃会话 |
| `security` | `rate_limit_enabled`、`ip_requests_per_minute`、**`settings_encryption_key`**、`log_request_body`、`max_request_body_bytes`、管理端 IP 白名单 | `settings_encryption_key` 用于加密数据库里的敏感运行期设置（SMTP 授权码等），必须是 32 字节 base64；**丢失将无法解密已存储的密钥** |
| `mail` | `prefer_database_config`、`enabled`、`host`/`port`、`security`（ssl/starttls/none）、`username`/`password`、`from_name`/`from_address`/`reply_to`、`skip_verify`、`timeout_seconds`、`rate_per_hour` | 推荐 `prefer_database_config: true`，由管理端在后台配置并加密入库；香港机器常封 25 端口，用 465/587 |
| `captcha` | `enabled`、`provider`（builtin / geetest / turnstile / none）、`geetest_id`/`geetest_key`、`turnstile_*`、内置验证码有效期与难度 | 登录/注册/找回密码建议开启 |
| `features` | 启动期兜底的功能开关：`user_qr_enabled`、`group_qr_enabled`、`scan_login_enabled`、`sticker_enabled`、`sticker_store_enabled`、`bot_platform_enabled`、`file_helper_enabled`、`link_card_enabled`、`voice_message_enabled` | 运行期以数据库 `features.*` 为准 |
| `logging` | `level`、`format`（text/json）、`output`（stdout/file）、`file_path`、`max_size_mb`、`max_backups`、`access_log`、`access_log_skip` | 生产用 `level: info`、`output: stdout`，交给 journald 收口 |
| `admin` | 首次启动引导管理员 `bootstrap`（`enabled`/`username`/`password`/`display_name`/`ez_id`）、`session_idle_minutes`、`allow_login_user_app` | ⚠️ 默认口令只是引导用，**首次登录后必须立刻改掉**，见 4.4 |
| `push` | 可选推送：`enabled`、`providers`（fcm/apns/hms/mipush/oppo/vivo）、`fcm.credentials_file`、`apns.*` | 未配置时只依赖长连接，不影响基本可用性 |
| `websocket` | `max_connections`（2C2G 建议 ≤5000）、`heartbeat_seconds`、`auth_timeout_seconds`、`send_buffer`、`max_frames_per_second` | 心跳要小于 Nginx 的 `proxy_read_timeout`；慢客户端靠 `send_buffer` 断开 |

### 4.2 分节结构示意（早期草案 · **非权威，请勿照抄字段名**）

> ⚠️ 下面这段 YAML 只用于说明「每个分节大致管什么」，**其中的字段名是早期设计草案**，
> 与仓库中真实的 `server/configs/config.example.yaml` 并不完全一致。
> **请不要直接复制使用**，请改用下一节 4.3 的骨架，并以
> `server/configs/config.example.yaml` 为唯一权威来源。

```yaml
# =============================================================================
# EzChat 服务端运行期配置
# 安装位置：/opt/ezchat/configs/config.yaml   权限：0640 属主：ezchat:ezchat
# =============================================================================

# ---------- 服务端监听与对外信息 ----------
server:
  listen: "127.0.0.1"          # 只监听回环，由 Nginx 反向代理；不要写 0.0.0.0
  port: 8080
  public_domain: "ezchat.im"
  # 超时（秒）：长连接由 ws 单独控制，这里主要约束普通 HTTP
  read_timeout: 30
  write_timeout: 60
  idle_timeout: 120
  # 单次请求体上限，必须与 Nginx 的 client_max_body_size 保持一致
  max_upload_size: "100m"

# ---------- 对外域名矩阵（根域名 ezchat.im 保持不动）----------
domains:
  api:    "https://api.ezchat.im"
  ws:     "wss://ws.ezchat.im"
  file:   "https://file.ezchat.im"
  open:   "https://open.ezchat.im"
  admin:  "https://admin.ezchat.im"
  review: "https://review.ezchat.im"
  docs:   "https://docs.ezchat.im"
  status: "https://status.ezchat.im"
  dl:     "https://dl.ezchat.im"
  mail:   "mail.ezchat.im"

# ---------- SQLite ----------
database:
  driver: "sqlite"
  path: "/opt/ezchat/data/ezchat.db"
  # 读连接池大小；写入由 SQLite 串行化，靠 busy_timeout 等待而不是报错
  max_open_conns: 8
  max_idle_conns: 4
  conn_max_lifetime: "1h"
  # 关键：遇到锁时等待而不是立刻失败（毫秒）
  busy_timeout: 5000
  # 建议同时开启 WAL 与外键
  journal_mode: "WAL"
  synchronous: "NORMAL"
  foreign_keys: true

# ---------- 媒体存储 ----------
storage:
  root: "/opt/ezchat/uploads"
  max_file_size: "100m"
  # 媒体过期策略（与 retention 配合）
  expire_policy: "follow_retention"

# ---------- 保留与清理 ----------
retention:
  text_days: 180              # 文本消息保留天数
  media_days: 14              # 图片 / 语音 / 文件保留天数
  emoji_forever: true         # 表情包永久保留
  cleanup_interval: "6h"      # 清理任务周期
  cleanup_cron: "0 4 * * *"   # 定时清理（低峰期）
  # 磁盘不足停机阈值：剩余空间低于该值时进入保护模式/停止写入
  disk_min_free: "5g"
  disk_stop_write_percent: 95

# ---------- 鉴权 ----------
auth:
  # ⚠️ 必须替换：openssl rand -base64 48
  jwt_secret: "__CHANGE_ME_openssl_rand_base64_48__"
  jwt_issuer: "ezchat.im"
  access_token_ttl: "15m"
  refresh_token_ttl: "720h"
  # 单点登录策略：踢掉旧设备 / 允许多设备并存
  single_session: false
  max_devices_per_user: 5

# ---------- 安全与限流 ----------
security:
  rate_limit:
    enabled: true
    per_ip_rps: 30            # 与 Nginx 的 ezchat_api 限流区对齐
    burst: 60
  blacklist:
    enabled: true
    duration: "1h"            # 触发限流后临时拉黑的时长
    max_violations: 10
  cors:
    enabled: true
    allowed_origins:
      - "https://admin.ezchat.im"
      - "https://review.ezchat.im"
    allowed_methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_credentials: true
  # 外链黑白名单默认值
  link_filter:
    mode: "blacklist"
    blacklist: []
    whitelist: []
  # 状态 API 防机器人
  status_api:
    require_token: true
    token_ttl: "5m"

# ---------- 邮件 ----------
mail:
  enabled: true
  smtp_host: "smtp.example.com"
  smtp_port: 465
  username: "no-reply@ezchat.im"
  # ⚠️ 必须替换：SMTP 授权码（不是邮箱登录密码）
  password: "__CHANGE_ME__"
  from_name: "EzChat 易信团队"
  from_address: "no-reply@ezchat.im"
  use_ssl: true               # 465 端口一般用 SSL
  use_starttls: false         # 587 端口一般用 STARTTLS
  template_html: "templates/mail/default.html"
  template_text: "templates/mail/default.txt"

# ---------- 验证码 ----------
captcha:
  enabled: true
  provider: "geetest"         # 方式：极验
  geetest_id: "__CHANGE_ME__"
  geetest_key: "__CHANGE_ME__"
  scenes: ["login", "register", "reset_password"]

# ---------- 功能总开关 ----------
features:
  register_enabled: true
  group_chat_enabled: true
  file_transfer_enabled: true
  voice_message_enabled: true
  moments_enabled: false           # 不做朋友圈
  video_call_enabled: false        # 不做视频通话
  voice_to_text_enabled: false     # 不做语音转文字
  admin_patrol_enabled: true       # 管理员静默巡查
  message_migration_enabled: true  # 聊天记录迁移到电脑

# ---------- 日志 ----------
logging:
  level: "info"                    # debug / info / warn / error
  format: "json"                   # 便于 journalctl + jq 检索
  output: "stdout"                 # 交给 systemd/journald 统一收口
  access_log: true
  # 慢请求阈值，超过则打印告警
  slow_request_ms: 1000
```

### 4.3 可直接使用的 `config.yaml` 骨架（字段名与样例文件一致）

下面是**按 `server/configs/config.example.yaml` 的真实字段名**整理的生产骨架，
按 2C2G + Nginx 反代 + `/opt/ezchat` 目录布局填写。
部署时请以样例文件为准逐字核对（样例文件里还有更详细的注释）。

```yaml
# =============================================================================
# EzChat 服务端运行期配置
# 安装位置：/opt/ezchat/configs/config.yaml   权限：0640  属主：ezchat:ezchat
# 任何字段都可用环境变量 EZCHAT_<SECTION>_<FIELD> 覆盖
# =============================================================================

server:
  host: "127.0.0.1"                        # 前置 Nginx 反代，只监听回环
  port: 8080
  public_base_url: "https://api.ezchat.im" # 必须与 Nginx / domains 一致
  mode: "release"                          # release | debug
  shutdown_timeout_seconds: 20
  trust_proxy_headers: true                # 前置 Nginx 时为 true
  max_public_concurrency: 200

domains:
  root:   "https://ezchat.im"              # 根域名保持不动，仅登记
  api:    "https://api.ezchat.im"
  ws:     "wss://ws.ezchat.im"
  file:   "https://file.ezchat.im"
  open:   "https://open.ezchat.im"
  admin:  "https://admin.ezchat.im"
  review: "https://review.ezchat.im"
  docs:   "https://docs.ezchat.im"
  status: "https://status.ezchat.im"
  dl:     "https://dl.ezchat.im"
  mail:   "ezchat.im"
  cors_origins:
    - "http://localhost:8081"
    - "http://127.0.0.1:8081"

database:
  path: "/opt/ezchat/data/ezchat.db"       # 必须在本地 SSD，禁止 NFS/SMB
  max_read_conns: 8
  busy_timeout_ms: 5000                    # 关键：并发写时等待而不是报 locked
  wal_autocheckpoint: 4000
  cache_size_kb: -20000                    # 约 20MB 页缓存
  write_queue_size: 1024
  migrations_dir: ""                       # 留空使用内置 embed 迁移
  auto_migrate: true

storage:
  root: "/opt/ezchat/uploads"              # 与 systemd ReadWritePaths、Nginx alias 一致
  temp_root: "/opt/ezchat/tmp"
  backup_root: "/opt/ezchat/backups"
  backend: "local"                         # local | s3（s3 为预留）
  max_file_size_mb: 100                    # 与 Nginx client_max_body_size 100m 一致
  multipart_threshold_mb: 20
  multipart_chunk_mb: 5
  signed_url_ttl_seconds: 900
  s3:
    endpoint: ""
    region: ""
    bucket: ""
    access_key: ""
    secret_key: ""
    use_ssl: true

# 启动期兜底；文本/媒体/表情包的具体天数在数据库的运行期设置里（管理端可改）
retention:
  enabled: true
  cleanup_interval_minutes: 30
  cleanup_batch_size: 2000
  min_free_disk_mb: 512                    # 可用空间低于此值触发 disk_guard_action
  disk_guard_action: "stop"                # stop | readonly | aggressive_cleanup
  max_run_seconds: 120

auth:
  jwt_secret: "__CHANGE_ME__openssl rand -base64 48__"
  jwt_issuer: "ezchat.im"
  access_token_minutes: 15
  refresh_token_days: 30
  slots:                                   # 每个槽位只允许一个活跃会话
    - "android"
    - "desktop"
    - "web"
    - "admin"
    - "review"
  qr_login_ttl_seconds: 180
  argon2:
    memory_kib: 65536
    iterations: 3
    parallelism: 2
    salt_length: 16
    key_length: 32

security:
  rate_limit_enabled: true
  ip_requests_per_minute: 600              # 与 Nginx 的 ezchat_api 限流区配合
  # ⚠️ 用于加密数据库中的敏感运行期设置；必须 32 字节 base64，且必须妥善备份
  settings_encryption_key: "__CHANGE_ME__openssl rand -base64 32__"
  log_request_body: false
  max_request_body_bytes: 2097152
  admin_ip_whitelist_enabled: false
  admin_ip_whitelist: []
  # 说明：外链黑白名单默认值属于「运行期设置」，在数据库里由管理端维护

mail:
  prefer_database_config: true             # 推荐：由管理端后台配置并加密入库
  enabled: false
  host: "smtp.163.com"
  port: 465
  security: "ssl"                          # ssl | starttls | none
  username: ""
  password: ""
  from_name: "易信 EzChat"
  from_address: "no-reply@ezchat.im"
  reply_to: ""
  skip_verify: false
  timeout_seconds: 15
  rate_per_hour: 200

captcha:
  enabled: true
  provider: "builtin"                      # builtin | geetest | turnstile | none
  geetest_id: ""
  geetest_key: ""
  turnstile_site_key: ""
  turnstile_secret: ""
  ttl_seconds: 180
  difficulty: 2

features:
  user_qr_enabled: true
  group_qr_enabled: true
  scan_login_enabled: true
  sticker_enabled: true
  sticker_store_enabled: true
  bot_platform_enabled: true
  file_helper_enabled: true
  link_card_enabled: true
  voice_message_enabled: true

logging:
  level: "info"                            # debug | info | warn | error
  format: "json"                           # text | json（json 便于 jq 过滤）
  output: "stdout"                         # stdout | file
  file_path: "/opt/ezchat/logs/ezchat.log"
  max_size_mb: 64
  max_backups: 7
  access_log: true
  access_log_skip: ["/healthz", "/readyz", "/v1/public/status"]

admin:
  bootstrap:
    enabled: true
    username: "admin"
    # ⚠️ 仅用于首次引导，登录后必须立即修改；也可以在初始化完成后改为 enabled: false
    password: "__CHANGE_ME__"
    display_name: "超级管理员"
    ez_id: "ezchat_admin"
  session_idle_minutes: 120
  allow_login_user_app: true

push:
  enabled: false
  providers: []                            # fcm | apns | hms | mipush | oppo | vivo
  fcm:
    credentials_file: ""
  apns:
    key_file: ""
    key_id: ""
    team_id: ""
    topic: "im.ezchat.app"

websocket:
  max_connections: 5000                    # 2C2G 建议 5000 以内
  heartbeat_seconds: 25                    # 必须小于 Nginx proxy_read_timeout
  auth_timeout_seconds: 10
  send_buffer: 256                         # 慢客户端超过即断开
  max_frames_per_second: 30
```

### 4.4 配置检查要点

* `server.host`: 前置 Nginx 时应为 `127.0.0.1`（或容器网络内地址），**不要** `0.0.0.0`。
* 上传体积的三处一致性：`storage.max_file_size_mb`（服务端）=
  Nginx `client_max_body_size`（如 `100m`）。分片上传还要看
  `storage.multipart_threshold_mb` / `multipart_chunk_mb` 是否被 Nginx 放行。
* `storage.root` 必须落在 systemd 的 `ReadWritePaths` 白名单里，并且与 Nginx
  `location /__internal_media/ { alias ... }` 指向同一个目录。
  默认样例用的是相对路径 `./data/media`，部署到 `/opt/ezchat` 时请改成绝对路径。
* `database.path` 所在目录必须可写（同样受 `ProtectSystem=strict` 约束）；
  **不要**把数据目录放到 NFS/SMB 上，否则 WAL 失效、有损坏风险。
* `auth.jwt_secret` 每个环境必须不同；更换它会让所有已签发的 token 立即失效。
* `security.settings_encryption_key` 用于加密数据库里的敏感运行期设置（SMTP 授权码等）。
  **必须和备份一起妥善保存**：一旦丢失，已加密入库的密钥将无法解密，只能重新配置。
* ⚠️ `admin.bootstrap.password` 只是首次引导用的默认口令。
  **首次登录后必须立刻修改**，并在管理员创建完成后把 `admin.bootstrap.enabled` 改为 `false`，
  避免每次重启都尝试用默认口令初始化。
* `server.public_base_url` 与 `domains.*` 必须与 `deploy/nginx/ezchat.conf` 里的
  `server_name` / 证书域名一致，否则二维码、邮件链接会指向错误地址。
* 配置文件权限 `0640`、属主 `ezchat:ezchat`，**绝不提交进 git**
  （`.gitignore` 已忽略 `config.local.yaml`、`config.local.yml`、`.env*` 等）。

---

## 五、Docker 部署

相关文件：

| 文件 | 作用 |
|---|---|
| `deploy/docker/Dockerfile.server` | 多阶段构建的服务端镜像（alpine + 非 root 用户 + 健康检查） |
| `deploy/docker/docker-compose.yml` | 服务端 + 可选 Nginx / Certbot 的编排 |
| `deploy/docker/.env.example` | 环境变量样例（复制为 `.env` 使用，不含真实密钥） |

### 5.1 安装 Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"     # 重新登录后生效
docker --version && docker compose version
```

### 5.2 准备配置与环境变量

```bash
cd deploy/docker
cp .env.example .env
sudoedit .env                      # 至少改 EZCHAT_VERSION 与各个目录路径

# 配置文件（容器内读 /app/configs/config.yaml）
mkdir -p configs data uploads logs backups
cp ../../server/configs/config.example.yaml configs/config.yaml
sudoedit configs/config.yaml       # 按第四节修改
```

> 容器里读的仍然是挂载进去的 `config.yaml`；`.env` 只负责容器运行参数
> （时区、`GOGC`、`GOMEMLIMIT`、资源上限、目录路径等）。
> `database.path` 建议写 `/app/data/ezchat.db`，`storage.root` 写 `/app/uploads`，
> 因为 compose 会把宿主机目录挂到这两个路径上。

### 5.3 构建与启动

```bash
# 方式一：本地构建镜像（构建上下文必须是仓库根）
DOCKER_BUILDKIT=1 docker build \
  -f deploy/docker/Dockerfile.server \
  -t ezchat/ezchat-server:1.0.0 \
  --build-arg VERSION=1.0.0 \
  ../..

# 方式二：直接跑 compose（会在需要时自动构建）
cd deploy/docker
docker compose up -d                     # 只起服务端
docker compose ps
docker compose logs -f ezchat-server
```

需要 Nginx / Certbot 时：

```bash
docker compose --profile with-nginx up -d
docker compose --profile with-nginx --profile with-certbot up -d
```

> `deploy/docker/docker-compose.yml` 里的 `nginx` 服务使用 `network_mode: host`，
> 这样容器内的 `127.0.0.1:8080` 就是宿主机，**同一份 `deploy/nginx/ezchat.conf`
> 在「Nginx 装在宿主机」和「Nginx 跑在容器里」两种场景下都通用**。
> 如果你更想用 bridge 网络：删掉 `network_mode: host`、恢复 `networks: [ezchat-net]`，
> 并把 `ezchat.conf` 里所有 `127.0.0.1:8080` 改成 `ezchat-server:8080`。

### 5.4 常用运维命令

```bash
docker compose ps                              # 状态
docker compose logs -f --tail=200 ezchat-server
docker compose restart ezchat-server
docker compose exec ezchat-server sh           # 进容器排查
docker stats --no-stream                       # 资源占用
docker compose config                          # 校验编排文件与变量替换结果
docker compose down                            # 停并删容器（命名卷与宿主目录保留）
```

### 5.5 容器注意事项

* 镜像内**以非 root 用户 `ezchat`（uid/gid 10001）运行**，因此挂载的宿主目录
  必须让 uid 10001 可写：
  ```bash
  sudo chown -R 10001:10001 /opt/ezchat/{configs,data,uploads,logs,backups}
  ```
* 时区通过 `TZ=Asia/Shanghai` 注入，保证业务时间戳为 GMT+8。
* 健康检查请求 `http://127.0.0.1:8080/healthz`。
  服务端提供 `/healthz`、`/readyz` 与 `/v1/public/status`
  （见 `server/configs/config.example.yaml` 中 `logging.access_log_skip` 的默认值），
  因此 Dockerfile 与 compose 里的健康检查路径可以直接用。
  若将来改动，请同步修改 `deploy/docker/Dockerfile.server` 与
  `deploy/docker/docker-compose.yml` 的 `healthcheck`。
* 日志用 `json-file` 驱动并限制 `max-size=10m`、`max-file=5`，避免小磁盘被日志写满。
* 资源限制写在 `deploy.resources.limits`（compose v2 会映射成 `--cpus` / `--memory`）。
* `config.yaml` 里含有 JWT 密钥与 SMTP 授权码，**只以只读方式挂载**，不要烘焙进镜像；
  镜像构建使用仓库根作为上下文时，请通过 `.dockerignore` 排除 `server/data/`、
  `server/uploads/`、`*.jks`、`.env` 等敏感路径。

---

## 六、Nginx 反向代理

### 6.1 安装与放置

```bash
sudo apt -y install nginx
sudo nginx -v

# 安装站点配置（模板来自仓库）
sudo install -m 0644 deploy/nginx/ezchat.conf /etc/nginx/conf.d/ezchat.conf
sudo mkdir -p /var/www/certbot
sudo mkdir -p /opt/ezchat/www/{dl,docs,status,admin,review,mail}

sudo nginx -t
sudo systemctl reload nginx
```

> 也可以放到 `/etc/nginx/sites-available/ezchat.conf` 并软链到 `sites-enabled/`。
> Ubuntu 的默认 `nginx.conf` 会同时 include `conf.d/*.conf` 与 `sites-enabled/*`，
> **二者选其一**，不要重复放置导致 `conflicting server name`。

### 6.2 模板里已经覆盖的内容

`deploy/nginx/ezchat.conf` 按域名矩阵提供了完整的 server 块，
所有需要按实际情况替换的地方都用**「★ 需替换」**标注：

| 子域名 | 作用 | 关键配置 |
|---|---|---|
| `api.ezchat.im` | 业务 API | `client_max_body_size 100m`、`limit_req zone=ezchat_api`、`/healthz` 只允许内网 |
| `ws.ezchat.im` | WebSocket 长连接 | `proxy_http_version 1.1` + `Upgrade`/`Connection` 头、`proxy_read_timeout 3600s`、`proxy_buffering off` |
| `file.ezchat.im` | 媒体上传 / 鉴权下载 | 上传与下载分开限流、`proxy_request_buffering off`、`X-Accel-Redirect` 内部目录 `location /__internal_media/` |
| `open.ezchat.im` | 开放平台 | 更严格的限流（`burst=120`）、独立日志 |
| `admin.ezchat.im` | 管理端 | **IP 白名单 `allow/deny`** + 可选 Basic Auth、安全响应头、静态站点 + `/api/` `/ws/` 反代 |
| `review.ezchat.im` | 审核端 | 同管理端（审核端能看到全部会话，属高敏感入口） |
| `docs.ezchat.im` | 文档静态站 | `try_files`、静态资源缓存 7 天 |
| `status.ezchat.im` | 状态页 / 状态 API | 严格限流 `zone=ezchat_status`、`Cache-Control: no-store` |
| `dl.ezchat.im` | 产物下载站 | `autoindex off`、按扩展名长缓存、`sendfile on` |
| `mail.ezchat.im` | 邮件说明页 / webmail | 默认说明页；有自建 webmail 时打开注释的反代段 |
| **`ezchat.im`（根域名）** | **保持不动** | 模板**不定义**根域名的 server 块，避免与现有主站冲突 |

### 6.3 关键配置逐项说明

**（1）WebSocket 升级**

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

location / {
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_buffering off;
    proxy_read_timeout 3600s;   # 长连接必须放宽，否则会被 Nginx 主动断开
    proxy_send_timeout 3600s;
    proxy_pass http://ezchat_ws_backend;
}
```

客户端心跳间隔必须**小于** `proxy_read_timeout`。

**（2）上传体积限制**

```nginx
client_max_body_size 100m;      # 必须与 config.yaml 的 storage 单文件上限一致
```

不一致时的典型现象：客户端上传大文件返回 **413 Request Entity Too Large**。

**（3）超时设置**

| 指令 | API | WebSocket | 文件 |
|---|---|---|---|
| `proxy_connect_timeout` | 5s | 5s | 5s |
| `proxy_send_timeout` | 60s | 3600s | 300s |
| `proxy_read_timeout` | 60s | 3600s | 300s |
| `client_body_timeout` | 120s | — | 300s |

**（4）Gzip**

模板在 http 上下文统一开启 `gzip`，并包含 `application/wasm`
（Flutter Web 产物里有 `.wasm`）。媒体文件本身已压缩，不重复压缩。

**（5）真实 IP 传递**

```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

服务端的 `security` 限流与黑名单依赖这些头拿到客户端真实 IP。
若前面还有 CDN，请按模板注释启用 `set_real_ip_from` + `real_ip_header`。

**（6）鉴权下载（`file.` 子域名的推荐做法）**

服务端校验权限后返回 `X-Accel-Redirect: /__internal_media/<相对路径>`，
Nginx 用 `sendfile` 直接从磁盘读，**不占用 Go 进程的 CPU 与内存**：

```nginx
location /__internal_media/ {
    internal;                                  # 只允许内部重定向访问
    alias /var/lib/ezchat/uploads/;            # ★ 需替换为真实媒体根目录
    sendfile on;
    tcp_nopush on;
    add_header X-Content-Type-Options nosniff;
}
```

> `alias` 的路径必须与 `config.yaml` 的 `storage.root` 指向同一个目录。

**（7）管理端 / 审核端的双重保护**

```nginx
allow 127.0.0.1;
allow 10.0.0.0/8;
allow 192.168.0.0/16;
deny  all;
# auth_basic           "EzChat Admin";
# auth_basic_user_file /etc/nginx/.htpasswd-ezchat-admin;
```

```bash
sudo apt -y install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd-ezchat-admin <用户名>
```

> 管理端权限极大（可静默巡查全部会话、可插话、可发站内信），
> **强烈建议不要把管理端直接暴露在公网**，优先走 WireGuard / 内网 / 跳板机。

### 6.4 校验与重载

```bash
sudo nginx -t                      # 语法检查，改完必须先跑这个
sudo systemctl reload nginx        # 平滑重载，不断连接
sudo tail -f /var/log/nginx/ezchat-api.error.log
```

---

## 七、域名解析与证书

### 7.1 DNS 记录清单

根域名 `ezchat.im` **保持不动**，只为下列子域名新增记录：

| 主机记录 | 类型 | 值 | 用途 |
|---|---|---|---|
| `api` | A / AAAA | 服务器 IPv4 / IPv6 | 业务 API |
| `ws` | A / AAAA | 同上（可加 CNAME 指向 `api`） | WebSocket |
| `file` | A / AAAA | 同上 | 媒体上传下载 |
| `open` | A / AAAA | 同上 | 开放平台 |
| `admin` | A / AAAA | 同上（建议只在内网解析或加白名单） | 管理端 |
| `review` | A / AAAA | 同上 | 审核端 |
| `docs` | A / AAAA | 同上 | 文档站 |
| `status` | A / AAAA | 同上 | 状态页 |
| `dl` | A / AAAA | 同上 | 下载站 |
| `mail` | A / AAAA + **MX** | 同上 + 邮件服务商 | 邮件发信/说明页 |

验证解析：

```bash
for h in api ws file open admin review docs status dl mail; do
  printf '%-10s -> ' "$h"; dig +short "$h.ezchat.im" | tr '\n' ' '; echo
done
```

> 证书签发**强依赖 DNS 已生效**。解析没生效就签，会失败并可能消耗 Let's Encrypt 的速率额度。

### 7.2 用 certbot 签发证书

```bash
sudo apt -y install certbot python3-certbot-nginx

# 先用 staging 演练一遍（不消耗正式额度，证书不被浏览器信任但流程一致）
sudo certbot --nginx --dry-run \
  -d api.ezchat.im -d ws.ezchat.im -d file.ezchat.im -d open.ezchat.im \
  -d admin.ezchat.im -d review.ezchat.im -d docs.ezchat.im \
  -d status.ezchat.im -d dl.ezchat.im -d mail.ezchat.im

# 正式签发（--redirect 会自动把 80 跳转到 443）
sudo certbot --nginx \
  -d api.ezchat.im -d ws.ezchat.im -d file.ezchat.im -d open.ezchat.im \
  -d admin.ezchat.im -d review.ezchat.im -d docs.ezchat.im \
  -d status.ezchat.im -d dl.ezchat.im -d mail.ezchat.im \
  --agree-tos -m admin@ezchat.im --redirect
```

certbot 的 nginx 插件会：

1. 用 HTTP-01 挑战（`/.well-known/acme-challenge/`，模板里已配好 `root /var/www/certbot`）验证域名归属；
2. 自动为每个 `server_name` 生成 443 server 块；
3. 写入 `ssl_certificate` / `ssl_certificate_key` 与 `include /etc/letsencrypt/options-ssl-nginx.conf`；
4. 按 `--redirect` 把 80 端口跳转到 443。

因此模板里各段的 `# return 301 https://$host$request_uri;` **不需要手动取消注释**。

**通配符证书（DNS-01）**：如果子域名会频繁增减，可以签 `*.ezchat.im`：

```bash
sudo apt -y install python3-certbot-dns-cloudflare   # 以 Cloudflare 为例
# /etc/letsencrypt/dns-cloudflare.ini: dns_cloudflare_api_token = <token>
sudo chmod 600 /etc/letsencrypt/dns-cloudflare.ini
sudo certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/dns-cloudflare.ini \
  -d 'ezchat.im' -d '*.ezchat.im'
```

> 通配符证书会覆盖 `*.ezchat.im`（含 `api.`/`ws.` 等），但**不覆盖根域名本身**，
> 所以要同时 `-d ezchat.im -d '*.ezchat.im'`。请确认根域名的解析与现有主站是否允许。

### 7.3 自动续期

```bash
# certbot 安装时会自带 systemd timer
systemctl list-timers | grep certbot
systemctl status certbot.timer

# 演练续期（不消耗额度，建议每月跑一次）
sudo certbot renew --dry-run

# 手动续期
sudo certbot renew --quiet

# 续期后自动重载 Nginx（certbot 的 nginx 插件默认会做；
# 若是 certonly 方式，需要手工加 hook）
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null <<'EOF'
#!/bin/sh
systemctl reload nginx
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

证书路径（`deploy/nginx/ezchat.conf` 末尾的 HTTPS 示例里用到的）：

```
/etc/letsencrypt/live/<域名>/fullchain.pem
/etc/letsencrypt/live/<域名>/privkey.pem
/etc/letsencrypt/options-ssl-nginx.conf
/etc/letsencrypt/ssl-dhparams.pem
```

监控到期：

```bash
sudo certbot certificates
echo | openssl s_client -servername api.ezchat.im -connect api.ezchat.im:443 2>/dev/null \
  | openssl x509 -noout -dates
```

---

## 八、2C2G 香港低配优化建议

2 vCPU / 2 GB 内存跑 IM 服务 + SQLite + Nginx 是够用的，但必须做取舍。
以下按收益排序。

### 8.1 加 SWAP（第一优先）

2G 内存在消息高峰或执行清理任务时很容易触发 OOM。加 2~4 GB SWAP 作为缓冲：

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 降低换出倾向：只有真正吃紧才用 SWAP，避免正常情况下的磁盘抖动
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-ezchat-swap.conf
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.d/99-ezchat-swap.conf
sudo sysctl --system
free -h
```

> 云厂商的云盘 IOPS 有限，SWAP 只能当保险丝，不能当内存用。

### 8.2 SQLite 调优

```yaml
database:
  path: "/opt/ezchat/data/ezchat.db"
  busy_timeout_ms: 8000     # 关键：并发写时等待 8 秒再报错，避免 "database is locked"
  wal_autocheckpoint: 4000  # WAL 自动检查点阈值（页数）；调小可让 WAL 更早回收
  cache_size_kb: -20000     # 约 20MB 页缓存；内存紧张时可调小到 -8000
  max_read_conns: 8         # 2C 机器不要开太大，SQLite 写入本身由单写队列串行化
  write_queue_size: 1024    # 写队列长度；满时写入方阻塞（背压），不要设得过大
```

> 日志模式与同步级别由服务端内部固定为 WAL + 合理的 synchronous 取值
> （`config.yaml` 里没有 `journal_mode` / `synchronous` 这两个字段，
> 不要照抄网上 SQLite 调优文章往里加，会被忽略或导致解析失败）。
> 如果确实需要调整，请改 `internal/store` 里的实现并重新构建。

配套运维：

```bash
# 定期整理碎片、回收空间（低峰期执行；会短暂持锁）
sudo -u ezchat sqlite3 /opt/ezchat/data/ezchat.db 'PRAGMA wal_checkpoint(TRUNCATE);'
sudo -u ezchat sqlite3 /opt/ezchat/data/ezchat.db 'VACUUM;'

# 观察数据库大小与 WAL 大小
ls -lh /opt/ezchat/data/
```

> WAL 模式下 `-wal` 文件可能比较大，这是正常的；
> 重启或用 `wal_checkpoint(TRUNCATE)` 会收缩。

### 8.3 Go 服务运行时

```bash
# /opt/ezchat/.env（单元文件里用 EnvironmentFile 读取）
cat <<'EOF' | sudo tee /opt/ezchat/.env
TZ=Asia/Shanghai
# GC 触发比例：内存紧张时用 CPU 换内存，50~80 是合理区间
GOGC=80
# 软内存上限：明显低于 systemd 的 MemoryMax=1500M，让 GC 先动手而不是被 OOM Kill
GOMEMLIMIT=1024MiB
EOF
sudo chown ezchat:ezchat /opt/ezchat/.env
sudo chmod 0640 /opt/ezchat/.env
sudo systemctl restart ezchat-server
```

同时单元文件里已设 `MemoryHigh=1200M` / `MemoryMax=1500M`，
形成「GC 先回收 → cgroup 限流 → 最后才 OOM」的阶梯。

### 8.4 Nginx 调优

```nginx
# /etc/nginx/nginx.conf 的顶层
worker_processes auto;          # 2 核 → 2 个 worker
worker_rlimit_nofile 65535;
events {
    worker_connections 16384;   # 单机万级长连接的基线
    multi_accept on;
    use epoll;
}
http {
    keepalive_timeout 65;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    # 缓存文件描述符与元数据，降低小文件压力
    open_file_cache          max=10000 inactive=30s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;
}
```

### 8.5 保留策略放在低峰期

`config.yaml` 的 `retention.cleanup_interval_minutes` 建议不要低于 30 分钟，
`cleanup_batch_size` 与 `max_run_seconds` 控制单轮清理的量与耗时，避免拖垮服务；
备份、`VACUUM` 之类会争抢磁盘 IO 的操作请放到凌晨低峰期。
文本 / 媒体 / 表情包的具体保留天数属于**运行期设置**（数据库 `system_settings` 的
`retention.*`），由 EzChat Admin Desktop 在后台修改，改完立即生效、不需要重启。

### 8.6 磁盘保护

```yaml
retention:
  min_free_disk_mb: 2048            # 可用空间低于 2GB 触发下面的动作
  disk_guard_action: "readonly"     # stop | readonly | aggressive_cleanup
  cleanup_batch_size: 2000
  max_run_seconds: 120
```

> `disk_guard_action` 的三个取值含义：`stop` = 整套系统强制停止运行（最保守，
> 宁可停服也不损坏数据库）；`readonly` = 进入只读，用户还能看历史消息；
> `aggressive_cleanup` = 激进清理媒体/过期数据来腾空间。
> 生产建议先用 `readonly`，并配合磁盘告警，人工介入扩容或清理。

配合监控：

```bash
# 简单的磁盘告警脚本（可放 crontab，每 10 分钟一次）
df -h / | awk 'NR==2 {gsub("%","",$5); if ($5+0 > 90) print "EzChat 磁盘告警: " $5 "%"}'
```

### 8.7 其它

* **关闭不需要的服务**：`snapd`、`ModemManager`、`bluetooth`、`cups` 等
  （`sudo systemctl disable --now snapd` 等）。
* **日志轮转**（见第九节）：Nginx 访问日志在 IM 场景下增长极快。
* **香港节点的现实问题**：大陆访问质量取决于线路（CN2 GIA / BGP 优于普通国际线路），
  晚高峰可能丢包；若主要用户在大陆，建议评估是否需要备案并放在境内。
* **时间同步**：`systemd-timesyncd` 或 `chrony` 必须正常运行，
  否则 JWT 有效期校验、消息时间戳都会异常。
* **不要在 2C2G 上同时跑**：全套 Docker（含 Nginx/Certbot）+ 编译任务 + 备份压缩。
  编译请在 GitHub Actions 上做。

---

## 九、日志与备份

### 9.1 日志

**应用日志**：`config.yaml` 的 `logging` 分节。生产建议 `level: info`、`format: json`、
`output: stdout`，由 systemd 统一收口到 journald。

```bash
# 实时查看
sudo journalctl -u ezchat-server -f
# 最近 200 行
sudo journalctl -u ezchat-server -n 200 --no-pager
# 按时间
sudo journalctl -u ezchat-server --since "2025-01-01 00:00:00" --until "2025-01-01 12:00:00"
# 只看错误
sudo journalctl -u ezchat-server -p err --no-pager
# JSON 格式 + jq 过滤（logging.format: json 时特别好用）
sudo journalctl -u ezchat-server -o cat | jq -c 'select(.level=="error")'
```

**journald 持久化与限额**（默认只存在内存，重启就丢）：

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald

# 限制 journal 总占用，避免小磁盘被写满
sudo sed -i 's/^#SystemMaxUse=.*/SystemMaxUse=512M/' /etc/systemd/journald.conf
sudo sed -i 's/^#SystemMaxFileSize=.*/SystemMaxFileSize=64M/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```

**Nginx 日志轮转**（`/etc/logrotate.d/ezchat-nginx`）：

```conf
/var/log/nginx/ezchat-*.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        [ -f /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid) || true
    endscript
}
```

> Ubuntu 的 nginx 包自带 `/etc/logrotate.d/nginx`，如果已经覆盖了
> `/var/log/nginx/*.log`，把上面这份留着也可（重复 rotate 不会出错），
> 或者直接依赖系统自带的那份。

### 9.2 备份

**要备份的三样东西**：

1. SQLite 数据库（`database.path`，以及同目录的 `-wal` / `-shm`）
2. 媒体文件目录（`storage.root`）
3. 配置文件（`/opt/ezchat/configs/config.yaml` 与 `/opt/ezchat/.env`）

**（1）SQLite 在线备份（不要直接 `cp` 数据库文件）**

`cp` 一个正在写入的 SQLite 文件可能拿到损坏的副本。用 SQLite 自带的备份命令：

```bash
sudo mkdir -p /opt/ezchat/backups
sudo -u ezchat sqlite3 /opt/ezchat/data/ezchat.db \
  ".backup '/opt/ezchat/backups/ezchat-$(date +%F-%H%M).db'"

# 或（SQLite 3.27+，会做碎片整理，适合冷备份）
sudo -u ezchat sqlite3 /opt/ezchat/data/ezchat.db \
  "VACUUM INTO '/opt/ezchat/backups/ezchat-$(date +%F-%H%M).db'"
```

**（2）备份脚本**（`/opt/ezchat/bin/backup.sh`）：

```bash
#!/usr/bin/env bash
# EzChat 每日备份：SQLite 在线备份 + 媒体目录增量同步 + 配置
set -euo pipefail

DATA_DIR=/opt/ezchat/data
UP_DIR=/opt/ezchat/uploads
CFG_DIR=/opt/ezchat/configs
BK_DIR=/opt/ezchat/backups
KEEP_DAYS=7
STAMP="$(date +%F-%H%M)"

mkdir -p "$BK_DIR/db" "$BK_DIR/config"

# 1) 数据库在线备份
sqlite3 "$DATA_DIR/ezchat.db" ".backup '$BK_DIR/db/ezchat-$STAMP.db'"
gzip -f "$BK_DIR/db/ezchat-$STAMP.db"

# 2) 配置（含 .env，注意权限）
cp -a "$CFG_DIR/config.yaml" "$BK_DIR/config/config-$STAMP.yaml"
[ -f /opt/ezchat/.env ] && cp -a /opt/ezchat/.env "$BK_DIR/config/env-$STAMP"

# 3) 媒体目录增量同步到备份区（硬链接方式省空间；也可换成 rsync 到远程）
rsync -a --delete \
  --link-dest="$BK_DIR/uploads-current" \
  "$UP_DIR/" "$BK_DIR/uploads-$STAMP/"
rm -f "$BK_DIR/uploads-current"
ln -s "$BK_DIR/uploads-$STAMP" "$BK_DIR/uploads-current"

# 4) 清理过期备份
find "$BK_DIR/db"     -name 'ezchat-*.db.gz' -mtime +$KEEP_DAYS -delete
find "$BK_DIR/config" -name 'config-*.yaml'  -mtime +$KEEP_DAYS -delete
find "$BK_DIR" -maxdepth 1 -name 'uploads-*' -type d -mtime +$KEEP_DAYS -exec rm -rf {} +

echo "[$(date -Is)] backup done: $STAMP"
```

```bash
sudo install -m 0755 -o root -g root /dev/stdin /opt/ezchat/bin/backup.sh <<'EOF'
...（把上面的内容粘贴进来）...
EOF
# 或者：把上面的内容写成文件后 install
# sudo install -m 0755 backup.sh /opt/ezchat/bin/backup.sh
sudo chown root:ezchat /opt/ezchat/bin/backup.sh

# crontab：每天 03:30 备份
echo '30 3 * * * /opt/ezchat/bin/backup.sh >> /opt/ezchat/logs/backup.log 2>&1' | sudo tee /etc/cron.d/ezchat-backup
sudo chmod 0644 /etc/cron.d/ezchat-backup
```

**（3）异地备份**（强烈建议：本机磁盘坏了本地备份一起没）

```bash
# 用 rclone 同步到对象存储（S3 / 阿里云 OSS / 腾讯云 COS / OneDrive 等）
sudo apt -y install rclone
rclone config                       # 按向导配置一个 remote，例如名为 oss
rclone sync /opt/ezchat/backups oss:ezchat-backup/hk-01 --transfers 4 --checkers 8
```

加进 crontab：`0 5 * * * rclone sync /opt/ezchat/backups oss:ezchat-backup/hk-01 --quiet`

**（4）恢复演练（每季度至少做一次）**

```bash
# 在测试机上验证备份可用
gunzip -c /opt/ezchat/backups/db/ezchat-2025-01-01-0330.db.gz > /tmp/restore.db
sqlite3 /tmp/restore.db 'PRAGMA integrity_check;'      # 期望输出 ok
sqlite3 /tmp/restore.db 'SELECT count(*) FROM sqlite_master;'
```

> **没演练过的备份等于没有备份。**

---

## 十、升级与回滚流程

### 10.1 升级前（每次都做）

```bash
# 1) 记录当前版本
/opt/ezchat/bin/ezchat-server --version 2>/dev/null || echo "（服务端不支持 --version，请查看日志/版本文件）"

# 2) 备份数据库与配置（见第九节）
sudo /opt/ezchat/bin/backup.sh

# 3) 额外留一份「上一个可用的二进制」
sudo cp -a /opt/ezchat/bin/ezchat-server /opt/ezchat/bin/ezchat-server.prev
ls -l /opt/ezchat/bin/

# 4) 下载新版本产物并校验
cd /tmp && sha256sum -c ezchat-server.sha256
```

### 10.2 原地升级

```bash
# 1) 停服务（systemd 会等优雅退出，最多 30 秒）
sudo systemctl stop ezchat-server

# 2) 替换二进制
sudo install -m 0755 -o root -g root /tmp/ezchat-1.1.0/ezchat-server \
     /opt/ezchat/bin/ezchat-server

# 3) 如果新版本配置格式有变化，先用样例对比差异再合并
sudo diff -u /opt/ezchat/configs/config.yaml /tmp/config.example.yaml || true

# 4) 启动并观察
sudo systemctl start ezchat-server
sudo systemctl status ezchat-server --no-pager
sudo journalctl -u ezchat-server -n 100 --no-pager

# 5) 验证
curl -sS -o /dev/null -w '@api %{http_code}\n'   http://127.0.0.1:8080/healthz || true
curl -sS -o /dev/null -w 'api. %{http_code}\n'   https://api.ezchat.im/healthz  || true
curl -sS -o /dev/null -w 'docs. %{http_code}\n'  https://docs.ezchat.im/        || true
```

### 10.3 蓝绿（零停机，推荐用于高峰期）

利用两套二进制 + 一次 Nginx 切换：

```bash
# 1) 新版本放到另一个路径
sudo install -m 0755 -o root -g root /tmp/ezchat-1.1.0/ezchat-server \
     /opt/ezchat/bin/ezchat-server-1.1.0

# 2) 用临时端口起一个新实例（配置里 server.port 改为 8081，
#    数据库仍指向同一个库；注意 SQLite 多进程写入要靠 busy_timeout）
sudo -u ezchat /opt/ezchat/bin/ezchat-server-1.1.0 \
     --config /opt/ezchat/configs/config-8081.yaml &

# 3) 确认新实例健康后，改 Nginx upstream 端口并 reload
sudo sed -i 's/server 127.0.0.1:8080/server 127.0.0.1:8081/' /etc/nginx/conf.d/ezchat.conf
sudo nginx -t && sudo systemctl reload nginx

# 4) 观察 5~10 分钟无异常后，停掉旧实例并改回端口管理方式
```

> ⚠️ 蓝绿方案下两个实例会同时写同一个 SQLite 库。
> 如果服务端没有做跨进程写入协调，**不要用蓝绿，用 10.2 的原地升级**，
> 或者把 `database.busy_timeout` 调大并接受短暂的重试延迟。

### 10.4 回滚

```bash
# 1) 停服务
sudo systemctl stop ezchat-server

# 2) 换回上一个可用二进制
sudo install -m 0755 -o root -g root \
     /opt/ezchat/bin/ezchat-server.prev /opt/ezchat/bin/ezchat-server

# 3) 如果新版本改过配置，换回旧配置备份
sudo cp -a /opt/ezchat/backups/config/config-<上一个时间戳>.yaml \
           /opt/ezchat/configs/config.yaml

# 4) 如果数据库已经被新版本迁移过且不兼容，回滚数据（⚠️ 会丢失升级后的数据）
sudo systemctl stop ezchat-server
sudo -u ezchat cp -a /opt/ezchat/data/ezchat.db /opt/ezchat/data/ezchat.db.bad
gunzip -c /opt/ezchat/backups/db/ezchat-<升级前时间戳>.db.gz \
  | sudo -u ezchat tee /opt/ezchat/data/ezchat.db >/dev/null
sudo rm -f /opt/ezchat/data/ezchat.db-wal /opt/ezchat/data/ezchat.db-shm

# 5) 启动
sudo systemctl start ezchat-server
sudo journalctl -u ezchat-server -n 100 --no-pager
```

**关于数据库迁移的处置建议**：

* 升级前**必须**备份数据库；回滚数据库等于放弃升级期间产生的新数据；
* 如果新版本引入了不可逆的 schema 变更，建议先在测试机用生产数据的副本验证升级与回滚路径；
* 生产数据量较大时，回滚前先确认 `retention` 清理任务没有在回滚窗口内删掉老数据。

### 10.5 版本与 tag 的对应关系

发布由 GitHub Actions 的 `release-all.yml` 完成（手动触发），
所有端使用**同一个版本号**并打**同一个 tag**（例如 `v1.1.0`）。
服务端二进制里的版本号也是链接期注入的同一个值，
因此「Release tag ↔ 服务端 `--version` 输出 ↔ 线上运行版本」三者可对齐。

对应关系速查：

```bash
# 服务器上确认当前跑的版本
sudo journalctl -u ezchat-server | grep -i version | tail -5
# GitHub 上确认该版本对应的产物
gh release view v1.1.0
gh release download v1.1.0 -p 'ezchat-server-1.1.0-linux-amd64*'
```

---

## 十一、常见问题排查

### 11.1 服务起不来

**现象**：`systemctl status ezchat-server` 显示 `failed`，`journalctl` 有 `Permission denied`。

| 原因 | 处理 |
|---|---|
| `ProtectSystem=strict` 下写不了数据目录 | 把数据目录加入 `ReadWritePaths`，`sudo systemctl daemon-reload && sudo systemctl restart ezchat-server` |
| 目录属主不对 | `sudo chown -R ezchat:ezchat /opt/ezchat/{data,uploads,logs,backups,tmp}` |
| 配置文件不可读 | `sudo chown ezchat:ezchat /opt/ezchat/configs/config.yaml && sudo chmod 0640 ...` |
| SELinux/AppArmor（Ubuntu 一般只有 AppArmor） | `sudo aa-status`；必要时为二进制添加 profile 或 `sudo aa-complain` |

**现象**：`journalctl` 里有 `bind: address already in use`。

```bash
sudo ss -lntp | grep 8080        # 看谁占了
sudo lsof -i :8080
sudo systemctl stop <占用进程>   # 或改 config.yaml 的 server.port
```

**现象**：`unit file ... contains unknown lvalue` 或启动即失败。

```bash
sudo systemd-analyze verify /etc/systemd/system/ezchat-server.service
```

### 11.2 SQLite `database is locked`

**原因**：并发写入冲突；WAL 未启用；`busy_timeout` 太小；有长事务占着写锁。

**处理**：

```yaml
database:
  busy_timeout_ms: 8000     # 先加大到 8000~15000
  max_read_conns: 8         # 不要开太大，写入由单写队列串行化
  write_queue_size: 1024    # 队列太小会让写入方频繁阻塞
```

```bash
# 看是否还有别的进程占着库
sudo lsof /opt/ezchat/data/ezchat.db
# 手工 checkpoint 收缩 WAL
sudo -u ezchat sqlite3 /opt/ezchat/data/ezchat.db 'PRAGMA wal_checkpoint(TRUNCATE);'
```

> 如果备份脚本或 `VACUUM` 正好在跑，也会短暂持锁 —— 把清理与备份都放到凌晨。

### 11.3 WebSocket 连不上 / 502 / 频繁掉线

| 现象 | 原因 | 处理 |
|---|---|---|
| 101 之后 60 秒断开 | `proxy_read_timeout` 太小 | 改成 `3600s`（模板已改） |
| 502 Bad Gateway | 上游挂了或端口不对 | `ss -lntp \| grep 8080`；核对 `upstream` 地址 |
| 连接建立不了、握手失败 | 缺少 `Upgrade`/`Connection` 头 | 确认 `map $http_upgrade $connection_upgrade` 与两个 `proxy_set_header`（模板已配） |
| 消息延迟高 | `proxy_buffering on` | 设 `proxy_buffering off`（模板已配） |
| 大量连接后新连接失败 | fd 或 `worker_connections` 不够 | 提高 `LimitNOFILE` 与 `worker_connections`，检查 `ss -s` |
| 混合内容报错 | 页面是 https，长连接用了 ws | 客户端应使用 `wss://`（`domains.ws` 配成 `wss://`） |

### 11.4 上传返回 413

**原因**：Nginx `client_max_body_size` 小于实际上传大小。

```nginx
client_max_body_size 100m;    # deploy/nginx/ezchat.conf
```

同步修改 `config.yaml` 的 `storage.max_file_size`，然后：

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 11.5 证书续期失败

```bash
sudo certbot renew --dry-run -v
```

| 报错 | 原因 | 处理 |
|---|---|---|
| `Challenge failed` / 404 | `/.well-known/acme-challenge/` 无法访问 | 确认模板里的该 location 存在、`/var/www/certbot` 权限正确（`www-data` 可读）、DNS 已生效 |
| `Too many requests` | 触发了 Let's Encrypt 速率限制 | 等一周，或先用 `--server https://acme-staging-v02.api.letsencrypt.org/directory` 调试 |
| `Could not bind to port 80` | 80 被占或防火墙拦住 | 90 天续期只需 80 可达；检查 `ufw` 与云安全组 |
| 续期成功但浏览器仍提示过期 | 没有 reload nginx | 加 `renewal-hooks/deploy/reload-nginx.sh` |

### 11.6 磁盘写满

```bash
df -h; du -sh /opt/ezchat/* /var/log/* /var/lib/docker 2>/dev/null | sort -h
```

常见元凶与处理：

| 元凶 | 处理 |
|---|---|
| `/var/log/nginx` 访问日志 | 配置 logrotate（第九节），或关闭 access_log |
| journald 日志 | `SystemMaxUse=512M`（第九节） |
| Docker 日志与镜像 | `docker system prune -af --volumes`（谨慎）、`logging` 限制大小 |
| 媒体目录 `uploads/` | 调小 `retention.media_days`；确认清理任务在跑 |
| SQLite `-wal` 膨胀 | `PRAGMA wal_checkpoint(TRUNCATE)` |
| 备份文件堆积 | 备份脚本的保留天数（`KEEP_DAYS`）与异地同步是否正常 |

> `config.yaml` 的 `retention.disk_stop_write_percent` 是最后一道防线：
> 磁盘到阈值会停止写入以保护数据库不损坏。看到它触发要立刻扩容或清理。

### 11.7 服务被 OOM Killer 杀掉

```bash
sudo journalctl -k | grep -i 'killed process'
sudo systemctl show ezchat-server -p MemoryCurrent -p MemoryMax
```

**处理**（按顺序）：

1. 加 SWAP（8.1）；
2. 调 `GOMEMLIMIT` 低于 `MemoryMax`（8.3），让 Go 先 GC；
3. 降 `GOGC` 到 50~80；
4. 降低 `database.max_open_conns`；
5. 缩短 `retention.text_days` / `media_days`，降低内存里的索引规模；
6. 仍不够就升配（2C2G → 2C4G 通常就能解决）。

### 11.8 时间不对 / 时间戳偏差

```bash
timedatectl                       # 看 Time zone 与 NTP synchronized
sudo systemctl status systemd-timesyncd
sudo timedatectl set-timezone Asia/Shanghai
```

业务时间戳按 GMT+8 约定，容器部署时要确认 `TZ=Asia/Shanghai` 已注入：

```bash
docker compose exec ezchat-server date
```

### 11.9 管理端 / 审核端打不开（403）

大概率是 `deploy/nginx/ezchat.conf` 里的 IP 白名单把你自己拦了。
先用手机热点或换网络验证，再把你的出口 IP 加进 `allow`：

```bash
curl -s https://ifconfig.me; echo          # 查你的出口 IP
sudo nginx -t && sudo systemctl reload nginx
```

### 11.10 日志里出现大量限流/黑名单记录

说明真的有异常流量或阈值太严。检查：

```yaml
security:
  rate_limit:
    per_ip_rps: 30        # 与服务端实际业务模型对齐
    burst: 60
  blacklist:
    duration: "1h"
    max_violations: 10
```

先看 Nginx 侧的限流日志确认是不是单个 IP 在刷，
再决定是调阈值还是把该 IP 段加入黑名单。

---

## 十二、部署检查清单

### 环境

- [ ] Ubuntu 22.04/24.04 已更新，时区为 `Asia/Shanghai`，NTP 同步正常
- [ ] 已创建 `ezchat` 系统用户与 `/opt/ezchat/{bin,configs,data,uploads,logs,backups,tmp}` 目录
- [ ] 目录属主与权限正确（配置 0640、数据 0700）
- [ ] `/etc/sysctl.d/99-ezchat.conf` 已生效，`fs.file-max`、`somaxconn` 已调整
- [ ] `ufw` 已开启，仅放行 22/80/443，**8080 未对公网开放**
- [ ] 云安全组同步放行 80/443
- [ ] SWAP 已配置（建议 2~4 GB，`vm.swappiness=10`）

### 应用

- [ ] 服务端二进制已下载、`sha256sum -c` 校验通过
- [ ] 二进制安装到 `/opt/ezchat/bin/ezchat-server`，权限 0755、属主 `root:root`
- [ ] `config.yaml` 已从 `server/configs/config.example.yaml` 复制并按第四节逐项填写
- [ ] `auth.jwt_secret` 已用 `openssl rand -base64 48` 替换
- [ ] `security.settings_encryption_key` 已用 `openssl rand -base64 32` 生成并**已随备份保存**
- [ ] `mail.password`（SMTP 授权码）已填写，`mail.security`（ssl/starttls/none）与端口匹配
- [ ] `server.host` 为 `127.0.0.1`，`server.port` 为 8080，`server.public_base_url` 与 Nginx 一致
- [ ] `storage.max_file_size_mb`（默认 100）与 Nginx `client_max_body_size`（`100m`）一致
- [ ] `storage.root` / `temp_root` / `backup_root` 与 systemd `ReadWritePaths`、Nginx `alias` 一致
- [ ] `database.path` 指向本地 SSD（非 NFS/SMB），`busy_timeout_ms` ≥ 5000
- [ ] `admin.bootstrap.password` 已改成强口令；管理员创建完成后已把 `bootstrap.enabled` 改为 `false`
- [ ] `websocket.heartbeat_seconds` 小于 Nginx 的 `proxy_read_timeout`
- [ ] `retention` 启动期兜底值确认（文本 180 天 / 媒体 14 天 / 表情包永久在数据库运行期设置中）
- [ ] 前台试跑成功，无报错

### systemd

- [ ] `deploy/systemd/ezchat-server.service` 已安装到 `/etc/systemd/system/`
- [ ] `ExecStart` 的参数与实际实现一致（`--config` 的正确性已确认）
- [ ] `ReadWritePaths` 与真实数据目录一致
- [ ] `systemctl enable --now ezchat-server` 成功，状态 `active (running)`
- [ ] `systemd-analyze security ezchat-server` 评分符合预期
- [ ] `/opt/ezchat/.env` 里的 `GOGC` / `GOMEMLIMIT` 已按内存设置

### Nginx 与证书

- [ ] `deploy/nginx/ezchat.conf` 已安装，所有「★ 需替换」处已改完
- [ ] 上游地址、`client_max_body_size`、静态站 `root` 路径均正确
- [ ] 管理端 / 审核端的 IP 白名单已按实际出口 IP 填写（或已启用 Basic Auth）
- [ ] 根域名 `ezchat.im` 的站点**未被本配置影响**
- [ ] `sudo nginx -t` 通过，`reload` 成功
- [ ] 10 个子域名 DNS 均已生效
- [ ] `certbot --nginx` 签发成功，80 → 443 跳转生效
- [ ] `systemctl list-timers | grep certbot` 存在，`certbot renew --dry-run` 通过
- [ ] 续期后 reload nginx 的 hook 已配置

### 可观测与数据安全

- [ ] journald 已持久化并设置 `SystemMaxUse`
- [ ] Nginx 日志轮转已配置
- [ ] `/opt/ezchat/bin/backup.sh` 已安装，crontab 已配置
- [ ] **恢复演练完成**，`PRAGMA integrity_check` 输出 `ok`
- [ ] 异地备份（rclone 等）已配置并验证过一次
- [ ] 磁盘告警已配置

### 上线前最后一步

- [ ] 从客户端真机验证：注册/登录、发文字、发图片、发文件、长连接收消息、断线重连
- [ ] 从管理端验证：管理员登录、巡查、站内信、监控面板
- [ ] 从审核端验证：审核相关流程
- [ ] 记录本次上线版本号与 tag（`vX.Y.Z`），与 GitHub Release 对齐
