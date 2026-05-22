# CLAUDE.md

## OpenPBS Docker 部署

本目录包含将 OpenPBS 部署为 Docker 容器的所有文件，以及配套的 **OpenPBS Web 前端项目**。

架构：PBS 服务器与 Web 前端**完全分离部署**，通过 SSH 通信。

```
                     部署一：PBS 服务器                    部署二：Web 前端
  ┌─────────────────────────────────────┐    ┌──────────────────────────────────┐
  │  openpbs-server (rockylinux:8)      │    │  openpbs-web-frontend (nginx)    │
  │  ├─ pbs_server :15001               │    │  :80 → React SPA                │
  │  ├─ pbs_sched  :15003               │    │  /api/* → proxy to backend      │
  │  ├─ pbs_mom    :15002               │    │  /ws/*  → proxy to backend      │
  │  ├─ pbs_comm   :15004               │    ├──────────────────────────────────┤
  │  └─ sshd :22                        │    │  openpbs-web-backend (FastAPI)   │
  └──────────┬──────────────────────────┘    │  :8000                           │
             │                               │  └─ SSH :22 → PBS 容器          │
             │                               │     ├─ hweuser (管理操作)        │
             │                               │     └─ sudo -u <user> (作业/终端)│
             └─── deploy_pbs-network ────────┘                                  │
                      Docker Bridge Network                      │
```

## windows目录结构（本地）

### PBS 部署（本目录）

```
docs/deploy/
  ├── .dockerignore            # Docker 构建忽略文件
  ├── Dockerfile               # 多阶段 Docker 镜像构建文件
  ├── docker-compose.yml       # PBS 服务器 Docker Compose 配置
  ├── docker-entrypoint.sh     # 容器入口脚本（初始化 + 启动 + SSH + sudo），初始系统用户密码（root/root hweuser/hweuser）
  ├── gen_db_password.py       # 数据库密码文件加密生成工具
  ├── fix_db_external.py       # 修复：外部数据库模式避免启动本地 dataservice
  ├── fix_pbsd_init.py         # 修复：非交互模式 + 前台模式日志输出
  ├── debug_db_connect.py      # 调试：数据库连接详细日志（生产环境已禁用）
  ├── fix_crlf.py              # 修复：安装后脚本的 Windows CRLF 换行符
  ├── build_and_deploy.sh      # Linux/Mac 一键构建部署脚本
  ├── build_and_deploy.bat     # Windows 一键构建部署脚本
  └── README.md                # 本文件
```

### Web 前端项目（独立部署）

```
openpbs-web/
  ├── docker-compose.yml       # Web 前端 Docker Compose 配置
  ├── setup_pbs_ssh.sh         # PBS 容器 SSH 配置脚本（可选）
  ├── backend/
  │   ├── Dockerfile           # 后端镜像构建（rockylinux:8）
  │   ├── requirements.txt     # Python 依赖
  │   └── app/
  │       ├── main.py          # FastAPI 入口
  │       ├── config.py        # 配置管理
  │       ├── api/             # API 路由（jobs, nodes, queues, server, auth, websocket）
  │       ├── core/            # 核心模块（pbs_client, auth）
  │       └── schemas/         # Pydantic 数据模型
  └── frontend/
      ├── Dockerfile           # 前端镜像构建（node:20-alpine → nginx:alpine）
      ├── nginx.conf           # Nginx 配置（SPA + API 反向代理）
      ├── package.json         # React 依赖
      └── src/
          ├── App.tsx          # 主布局（Ant Design）
          ├── pages/           # 页面（Dashboard, Jobs, Nodes, Queues, Terminal）
          ├── hooks/           # 自定义 Hook（useWebSocket）
          └── types/           # TypeScript 类型定义
```
## Linux 服务器部署（192.168.40.96）

### PBS 服务 — `/data/openpbs/`

```
/data/openpbs/
  ├── docker-compose-linux.yml          # PBS 容器编排配置
  ├── pbs_wrappers/               # PBS 作业包装脚本
  │   └── pbs_job_watchdog.sh     #   通用资源看门狗
  ├── pbshome/                    # PBS 运行时数据（→ /var/spool/pbs）
  └── pgsqldata/                  # PostgreSQL 数据（→ /var/lib/postgresql/data）
  └── userhome/                  # 系统用户home目录（→ /home）
  └── accounts/                  # 系统用户账号（→ /etc/account-persist）
```

**docker-compose-linux.yml 关键配置：**

| 类别 | 项                                    | 值                                    |
|----|--------------------------------------|--------------------------------------|
| 镜像 | `openpbs-server`                     | `openpbs:latest`                     |
| 环境 | `PBS_DATA_SERVICE_HOST`              | `host.docker.internal` (本机 Postgres) |
|    | `PBS_DATA_SERVICE_PORT`              | `5432`                               |
|    | `PBS_DATA_SERVICE_USER`              | `hweuser`                            |
|    | `PBS_DB_PASSWORD`                    | `hweuser`                            |
| 端口 | 15001                                | PBS Server                           |
|    | 15002                                | PBS Mom                              |
|    | 15003                                | PBS Scheduler                        |
|    | 15004                                | PBS Comm                             |
| 挂载 | `/data/openpbs/pbshome`              | `/var/spool/pbs`                     |
|    | `./pbs_wrappers/pbs_job_watchdog.sh` | `/opt/pbs/bin/pbs_job_watchdog` (ro) |
|    | `/data/apps/atrans/script/atrans.sh` | `/opt/pbs/bin/atrans` (ro)           |
|    | `/data/apps`                         | `/data/apps` (ro)                    |
|    | `/data/3dmodel`                      | `/data/output/3dmodel`               |
|    | `/data/uploads`                      | `/data/uploads`（与 Web 后端共享）          |
|    | `/data/openpbs/accounts`             | `/etc/account-persist`               |
| 网络 | `openpbs-net`                        | bridge                               |

### Web 前端/后端 — `/data/openpbs-web/`

```
/data/openpbs-web/
  ├── docker-compose.yml          # Web 服务编排配置
  └── appconfig/                  # 应用配置（→ /app/config/）
      └── apps.yaml               #   作业提交表单配置
```

**docker-compose.yml 关键配置：**

| 服务 | 项 | 值 |
|------|----|----|
| **backend** | 镜像 | `openpbs-web-backend:latest` |
| | 端口 | `8000:8000` |
| | `PBS_WEB_PBS_SSH_HOST` | `openpbs-server` |
| | `PBS_WEB_PBS_SSH_PORT` | `22` |
| | `PBS_WEB_PBS_ADMIN_USER` | `hweuser` |
| | `PBS_WEB_PBS_ADMIN_PASSWORD` | `hweuser` |
| | `PBS_WEB_PBS_DB_HOST` | `postgres` (容器名) |
| | `PBS_WEB_APPS_CONFIG_PATH` | `/app/config/apps.yaml` |
| | `PBS_WEB_UPLOAD_DIR` | `/data/uploads` |
| | 挂载 | `/data/openpbs-web/appconfig:/app/config:ro` |
| | | `/data/uploads:/data/uploads` |
| **frontend** | 镜像 | `openpbs-web-frontend:latest` |
| | 端口 | `80:80` |
| **tomcat** | 镜像 | `tomcat:9.0-jre11` |
| | 端口 | `8080:8080` |
| | 挂载 | `/data/3dmodel:/usr/local/tomcat/webapps/models` |
| | | `/data/apps/3dviewer:/usr/local/tomcat/webapps/3dviewer` |

### 辅助容器

| 容器 | 镜像 | 数据卷 | 网络 |
|------|------|--------|------|
| `openpbs-postgres` | `postgres:15` | `/data/openpbs/pgsqldata:/var/lib/postgresql/data` | `openpbs-net` |

### 完整映射关系图

```
宿主机 (/data)                       容器内
──────────────────────────────────────────────────────────
/openpbs/pbshome/              →    /var/spool/pbs
/openpbs/pbs_wrappers/*.sh     →    /opt/pbs/bin/* (ro)
/openpbs/pgsqldata/            →    /var/lib/postgresql/data
/3dviewer/                     →    /usr/local/tomcat/webapps/models (Tomcat)
/apps/                         →    /data/apps (ro, PBS)
/3dmodel/                      →    /data/output/3dmodel (PBS)
/uploads/                      →    /data/uploads (PBS + Web Backend)
/openpbs-web/appconfig/        →    /app/config (ro, Web Backend)
/openpbs/userhome              →    /home
/openpbs/accounts              →    /etc/account-persist
```

> **注意**：修改服务器文件前先备份：`cp file.yml file.yml.bak.$(date +%Y%m%d_%H%M%S)`

## 前置条件

1. **Docker Desktop** (Windows/Mac) 或 Docker Engine (Linux)
2. **PostgreSQL** 运行在宿主机上（或使用 `openpbs-postgres` 容器），配置如下：
   - 数据库名：`pbs_datastore`
   - 用户：`hweuser`
   - 密码：`hweuser`
   - 必须安装 `postgresql-contrib` 包（提供 `hstore` 扩展）

## 快速开始-本机windows

### 第一步：部署 OpenPBS 服务器

**Windows**

```batch
cd D:\scsCloud\workspace\openpbs\docs\deploy
build_and_deploy.bat
```

**Linux/Mac**

```bash
cd /d/scsCloud/workspace/openpbs/docs/deploy
bash build_and_deploy.sh
```

部署完成后验证：

```bash
docker logs -f openpbs-server
# 看到 "OpenPBS 启动服务" 表示成功

docker exec openpbs-server qstat -B
# 显示 PBS 服务器状态
```

### 第二步：构建并启动 Web 前端

```bash
cd D:\scsCloud\workspace\openpbs\openpbs-web

# 构建并启动
docker compose up -d
```

**注意**：PBS 服务器必须先启动，因为 Web 项目使用 PBS 部署创建的 Docker 网络 `deploy_pbs-network`。

### 第三步：访问

| 地址 | 说明 |
|------|------|
| `http://localhost` | Web 前端页面（仪表盘、作业管理、节点、队列、终端） |
| `http://localhost:8000/api/docs` | 后端 API 文档（Swagger UI） |
| `http://localhost:8000/api/server/health` | API 健康检查 |

## 端口映射

### PBS 服务器端口

| 宿主机端口 | 容器端口 | 服务 |
|-----------|---------|------|
| 15001 | 15001 | PBS Server |
| 15002 | 15002 | PBS Mom |
| 15003 | 15003 | PBS Scheduler |
| 15004 | 15004 | PBS Comm |
| 22 | 22 | SSH（远程管理 + 后端命令通道） |

### Web 前端端口

| 宿主机端口 | 容器端口 | 服务 |
|-----------|---------|------|
| 80 | 80 | Nginx（React SPA + API 反向代理） |
| 8000 | 8000 | FastAPI 后端 |

## SSH 通信机制

Web 后端通过 **paramiko SSH** 连接到 PBS 容器的 sshd 服务：

```
Web Backend (FastAPI)                     PBS Container
      │                                       │
      │  SSH as hweuser@openpbs:22            │  sshd :22
      ├──────────────────────────────────────>│  ├─ 管理操作 (qstat, qmgr, pbsnodes)
      │  exec: "qstat -B"                     │  ├─ 作业操作 (qdel, qhold, qrls)
      │<──────────────────────────────────────│  └─ 文件操作 (ls, mkdir, cp)
      │                                       │
      │  SSH as hweuser@openpbs:22            │
      ├──────────────────────────────────────>│
      │  exec: "sudo -u songhui qsub ..."     │  └─ 以用户身份提交作业
      │<──────────────────────────────────────│
```

- **hweuser** 拥有免密 sudo 权限（`/etc/sudoers.d/hweuser`）
- 管理操作以 hweuser 身份直接执行
- 作业提交通过 `sudo -u <user>` 以目标用户身份执行，作业自然归属正确用户
- Web 终端通过 SSH + sudo 以登录用户身份执行命令
- 无需额外端口（复用 SSH 22），无需 .rhosts 配置

## 环境变量

### PBS 服务器环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PBS_SERVER` | `openpbs` | PBS 服务器主机名 |
| `PBS_DATA_SERVICE_HOST` | `192.168.0.109` | PostgreSQL 主机地址 |
| `PBS_DATA_SERVICE_PORT` | `5432` | PostgreSQL 端口 |
| `PBS_DATA_SERVICE_USER` | `hweuser` | 数据库用户名 |
| `PBS_DB_PASSWORD` | `hweuser` | 数据库密码 |

### Web 后端环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PBS_WEB_DEBUG` | `true` | 调试模式 |
| `PBS_WEB_WS_POLL_INTERVAL` | `3` | WebSocket 推送间隔（秒） |
| `PBS_WEB_PBS_SSH_HOST` | `openpbs-server` | PBS SSH 主机 |
| `PBS_WEB_PBS_SSH_PORT` | `22` | PBS SSH 端口 |
| `PBS_WEB_PBS_ADMIN_USER` | `hweuser` | PBS 管理用户 |
| `PBS_WEB_PBS_ADMIN_PASSWORD` | `hweuser` | PBS 管理用户密码 |
| `PBS_WEB_SECRET_KEY` | 内置默认值 | JWT 签名密钥 |
| `PBS_WEB_ACCESS_TOKEN_EXPIRE_MINUTES` | `480` | Token 过期时间 |
