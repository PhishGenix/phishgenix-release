# PhishGenix 系统生产环境部署指南

本文档面向实施与运维人员，详细记录 PhishGenix 系统在 Linux 生产服务器上的部署架构、环境准备、一键安装、服务管理与日常运维操作步骤，用于指导系统的规范发布与维护。

---

## 1. 系统部署架构概述

本系统采用 **“宿主机混合容器化”** 的黄金组合架构进行部署：

- **前端（Vue 3）**：由 **Nginx 容器**进行静态资源托管。
- **后端（Go）**：直接以 **Linux 宿主机原生进程** 运行（通过 Systemd 守护），享有极致性能与简单维护。
- **第三方依赖（MySQL、Redis、Mailpit、Postfix）**：全部作为 **Docker 容器** 运行，实现环境解耦与数据持久化。
- **网络连通性**：Nginx 容器与 Postfix 容器通过配置 `host.docker.internal` 网络解析，反向代理/投递邮件给宿主机运行的 Go 后端服务（`8080` 端口）。

---

## 2. 环境准备要求

### 2.1 Linux 生产服务器要求

- 操作系统：Rocky Linux 9+ / Ubuntu 20.04+ / Debian 11+ / AlmaLinux 等。

- 安装有 **Docker** 与 **Docker Compose V2**（能够运行 `docker compose` 命令行）。

- 安装有 **unzip**（解压工具）。

- **关闭 SELinux**：SELinux 处于 Enforcing 模式时会拦截 Docker 容器对挂载配置文件（如 Nginx、Postfix）的读取及宿主机目录的访问，导致容器启动失败或权限异常。部署前须将其关闭：
  
  ```bash
  # 临时关闭（立即生效，重启后恢复）
  sudo setenforce 0
  
  # 永久关闭（修改配置文件，重启后仍生效）
  sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
  
  # 验证当前状态应为 Disabled 或 Permissive
  getenforce
  ```
  
  > Ubuntu / Debian 默认不启用 SELinux，可跳过本步骤。

### 2.2 附：Rocky Linux 9 部署 Docker 与 Docker Compose 指南

若您的生产服务器为 Rocky Linux 9，请按照以下步骤完成容器化环境的部署：

#### 1. 移除冲突或旧版本组件

```bash
sudo dnf remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
```

#### 2. 添加 Docker 官方仓库源

安装 DNF 配置工具包并添加官方 Docker CE 稳定版仓库（Rocky Linux 9 可完美兼容使用 CentOS 兼容源）：

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 3. 安装 Docker 核心引擎及 Compose 插件

一键安装 Docker 运行引擎、命令行客户端以及官方的 Docker Compose V2 插件（即 `docker-compose-plugin`）：

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 4. 启动服务并设置开机自启

```bash
# 开启 Docker 守护进程
sudo systemctl start docker

# 开启开机自启动
sudo systemctl enable docker

# 检查运行状态是否为 active (running)
sudo systemctl status docker
```

#### 5. 验证安装版本

```bash
docker --version
docker compose version
```

---

## 3. 服务器部署流程（Linux 服务器执行）

### 3.1 创建系统服务账号

部署前需先在服务器上手动创建专用的系统服务账号 `phishgenix`，用于运行 Go 后端进程并持有数据目录所有权：

```bash
# 创建系统非登录账号（无 home 目录、禁止交互登录）
sudo useradd phishgenix
```

### 3.2 上传并解压

将本地生成的部署包上传到服务器的 `/home/phishgenix` 目录下。**注意选择与服务器 CPU 架构匹配的包**（`uname -m` 输出 `x86_64` 用 `phishgenix-release-amd64-<版本>.zip`，`aarch64` 用 `phishgenix-release-arm64-<版本>.zip`；`<版本>` 为打包时 `VERSION` 的值，如 `v1.6.0`）：

```bash
# 进入部署根目录
cd /home/phishgenix

# 查看本机架构，确认该用哪个包
uname -m

# 解压安装包（按实际架构二选一，<版本> 替换为实际包名中的版本号）
unzip phishgenix-release-amd64-<版本>.zip     # x86_64 服务器
unzip phishgenix-release-arm64-<版本>.zip     # ARM64 服务器
```

> [!IMPORTANT]
> `install.sh` 会读取 `server/phishgenix-server` 的 ELF 头校验架构与宿主机是否一致，架构不匹配会直接报错退出，避免部署完成后才出现 `exec format error`。

### 3.3 导入离线 Docker 镜像（内网/隔离环境必选，联网环境可跳过）

若生产服务器处于**无互联网访问的纯内网或隔离环境**，`install.sh` 执行 `docker compose up -d` 时将无法从公网拉取 MySQL、Redis、Nginx、Postfix、Mailpit 等中间件镜像，导致容器启动失败。此时需在运行一键配置脚本之前，预先导入离线镜像。

**① 准备离线镜像包**：在一台能联网且已部署 PhishGenix 的机器上，使用备份脚本导出镜像（详见[数据备份与恢复](docs/backup-restore.md)）：

```bash
/home/phishgenix/deploy/backup/phishgenix-backup.sh --export-images
```

导出后，备份目录（如 `backup_20260629_140000/`）下会生成 `images/` 子目录（含各中间件镜像的 `.tar` 归档）与 `image_metadata.txt` 元数据文件。将该目录整体上传至生产服务器（例如 `/tmp/restore/backup_20260629_140000`）。

**② 在生产服务器上执行只导入镜像的恢复命令**：

```bash
sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000 --images-only
```

脚本会读取 `images/` 下的所有镜像归档并载入，再依据 `image_metadata.txt` 自动校准镜像名称与标签，完成后即退出，不执行任何数据恢复动作。

> [!NOTE]
> 联网环境下可跳过本步骤，`install.sh` 会在执行 `docker compose up -d` 时自动拉取所需镜像。

### 3.4 运行一键配置脚本

以 `sudo` 权限执行解压出来的 `install.sh` 脚本，自动初始化并**一键拉起所有组件运行**：

```bash
cd /home/phishgenix
sudo ./install.sh
```

> 如果这台机器要对外发布产品官网（即访问 `/` 打开主页，而不是控制台），改用
> `sudo ./install.sh --with-site`。不带参数时脚本**保持现状**：新装实例的 `/` 仍是控制台
> 入口，已发布过官网的实例继续发布（升级不会顺手把它关掉）—— 详见 3.7。
> **此脚本会自动帮您完成以下动作：**

1. 校验系统账号 `phishgenix` 是否存在（须已按 3.1 手动创建，否则报错退出）。
2. 为 Go 运行文件 `phishgenix-server` 添加可执行权限。
3. 创建持久化数据目录 `/home/phishgenix/data/uploads` 与日志目录 `/home/phishgenix/server/logs`。
4. 递归将整个 `/home/phishgenix` 的所有者变更为 `phishgenix:phishgenix` 账号。
5. 将 `deploy/phishgenix.service` 自动复制到系统的服务目录中，重载配置并将其设为**开机自启动**。
6. **自动实例化生产配置**：检测到 `deploy/docker-compose.yml` 不存在时，基于模板生成并注入随机强密码，避免升级覆盖风险。
7. **全自动拉起服务**：自动执行 `docker compose up -d` 启动所有容器，并自动运行宿主机 Go 服务，最后**在屏幕上直观输出管理员初始密码及登录引导**。

### 3.5 配置与校验生产环境变量

一键配置脚本 `install.sh` 已经在初始化时**自动帮您复制了配置文件模板并生成了安全的随机高强度密码与 JWT 密钥**，并已同步更新至 Docker 容器的编排文件中。

如果您没有特殊的定制需求，**您可以直接跳过此步骤，无需手动创建和配置**。

若需要进行手动校验或自定义修改，可以执行以下命令：

```bash
cat /home/phishgenix/server/.env
```

关键配置项及逻辑说明如下：

- `APP_ENV=prod`：生产环境标志。
- `JWT_SECRET`：系统初始化时已自动生成 64 位的高强度随机密钥，用于 Token 安全加密。
- `MYSQL_PASSWORD`：系统已自动生成 12 位随机高强度数据库密码（且自动与 `deploy/docker-compose.yml` 中的 MySQL 容器密码保持同步）。
- `FILE_STORAGE_PATH=/home/phishgenix/data/uploads`：文件上传持久化目录，已由脚本自动创建并赋予 `phishgenix` 账号权限。
- `BOUNCE_CALLBACK_SECRET`：退信/弹回通知接口的安全签名共享密钥。系统初始化时已自动随机生成 48 位强密钥，用于对 Postfix 投递进来的退信回调进行 HMAC-SHA256 签名合法性鉴权。
- `BOUNCE_CALLBACK_ALLOWED_IPS`：免签退信回调客户端的内网 IP/CIDR 允许列表（以逗号分隔）。默认设置为 `127.0.0.1,172.16.0.0/12,192.168.0.0/16,10.0.0.0/8`。凡是来自允许网段内的退信回调（如本地运行的 Postfix 服务）均可免签名认证直接放行。

### 3.6 校验服务状态与手动启停（可选）

> [!NOTE]
> **以下步骤已由 `install.sh` 脚本在初始化时全自动执行完毕。**
> 如果您在后续维护中需要手动校验状态、或者手动重启服务，可以执行以下指令：

- **手动管理 Docker 容器组件（MySQL、Nginx、Redis、Postfix 等）**：
  
  ```bash
  cd /home/phishgenix/deploy
  
  # 启动所有容器
  docker compose up -d
  # 停止所有容器
  docker compose down
  # 重启特定容器 (如 nginx)
  docker compose restart nginx
  ```

- **手动管理 Go 后端进程服务**：
  
  ```bash
  # 启动后端
  sudo systemctl start phishgenix
  # 停止后端
  sudo systemctl stop phishgenix
  # 重启后端
  sudo systemctl restart phishgenix
  # 查看运行状态是否为 active (running)
  sudo systemctl status phishgenix
  ```

部署完成后，通过浏览器访问服务器的 `80` 端口（或绑定的域名），即可进入系统：

- 默认超级管理员账号：`superadmin`，密码：`superpassword`

### 3.7 可选：把官网发布到根路径（`--with-site`）

发布包内自带 `site/`（产品官网单页，纯静态）。**它默认不启用**：客户实例访问 `/` 仍然是控制台。
只有需要对外展示站点（例如官方演示机）时，才在安装/升级时显式打开。

开关是**三态**的：不带参数不会改动现状，只有显式 `--no-site` 才会把官网收回去。

```bash
cd /home/phishgenix
sudo ./install.sh --with-site      # 发布：/ 打开官网
sudo ./install.sh --no-site        # 收回：/ 回到控制台（只有这个参数会关闭官网）
sudo ./install.sh                  # 不带参数：保持现状（升级 / 重跑时的推荐写法）
# 等价的环境变量写法（无人值守/批量初始化）：
sudo ENABLE_MARKETING_SITE=1 ./install.sh
sudo ENABLE_MARKETING_SITE=0 ./install.sh
```

| 命令            | 当前已发布                               | 当前未发布 |
| ------------- | ----------------------------------- | ----- |
| `--with-site` | 保持发布                                | 发布    |
| `--no-site`   | **收回（下线官网）**                        | 无变化   |
| 不带参数          | **保持发布**（nginx.conf 与 override 都不写） | 无变化   |

> 覆盖安装 / 升级时不要为了「保险」顺手加 `--no-site`——那等于主动下线官网。不带参数就是最安全的写法。脚本判断「当前是否已发布」的依据是 `deploy/nginx.conf` 里的 `# >>> marketing-site >>>` 标记块，不看 `site/` 目录是否存在（发行包恒带 `site/`）。

启用后行为差异：

| 访问路径                   | 未发布官网                  | 已发布（`--with-site`） |
| ---------------------- | ---------------------- | ------------------ |
| `/`                    | 控制台（重定向到 `/dashboard`） | 官网主页               |
| `/login`               | 登录页                    | 登录页（官网 CTA 指向这里）   |
| `/dashboard`           | 控制台首页                  | 控制台首页              |
| `/training/...`、`/open`、`/click`、`/landing` | 演练页面 | 演练页面 |

---

## 4. 日志与日常运维指令

> 此处仅列出部署后最常用的几项。

### 4.1 查看宿主机后端日志

系统使用 Systemd 管理 Go 后端，其标准输出会被 journald 收集，并且程序会自动在 `/home/phishgenix/server/logs/` 目录下按天写入分类的日志文件。

- **查看系统守护日志**：
  
  ```bash
  # 实时查看最新的 100 行日志
  sudo journalctl -u phishgenix -n 100 -f
  ```

- **查看本地物理日志文件**：
  
  ```bash
  # 例如查看邮件收发系统日志
  tail -f /home/phishgenix/server/logs/smtp.log.*
  ```

### 4.2 查看 Docker 容器日志

- **查看 Nginx 反向代理日志**：
  
  ```bash
  docker logs -f phish_nginx
  ```

- **查看数据库/邮件投递日志**：
  
  ```bash
  docker logs -f phish_mysql
  docker logs -f phish_postfix
  ```

---

# 🇬🇧 English Documentation

# PhishGenix Production Deployment Guide

This document is written for implementation and operations personnel. It records the deployment architecture, environment preparation, one-click installation, service management, and routine operations of PhishGenix on Linux production servers, and serves as the standard reference for releasing and maintaining the system.

---

## 1. Deployment Architecture Overview

The system is deployed with a **hybrid host + container** architecture:

- **Frontend (Vue 3)**: static assets are served by an **Nginx container**.
- **Backend (Go)**: runs as a **native process on the Linux host** (supervised by systemd), which gives maximum performance and simple maintenance.
- **Third-party dependencies (MySQL, Redis, Mailpit, Postfix)**: all run as **Docker containers**, decoupling the environment and persisting data.
- **Network connectivity**: the Nginx and Postfix containers reach the host through `host.docker.internal`, so they can reverse-proxy and deliver mail to the Go backend running on host port `8080`.

---

## 2. Environment Preparation

### 2.1 Linux Production Server Requirements

- Operating system: Rocky Linux 9+ / Ubuntu 20.04+ / Debian 11+ / AlmaLinux, etc.

- **Docker** and **Docker Compose V2** installed (the `docker compose` command must work).

- **unzip** installed.

- **SELinux disabled**: when SELinux runs in Enforcing mode it blocks containers from reading mounted configuration files (Nginx, Postfix) and from accessing host directories, which leads to container start failures or permission errors. Disable it before deployment:

  ```bash
  # Temporary (takes effect immediately, reverts after reboot)
  sudo setenforce 0

  # Permanent (edits the config file, survives reboot)
  sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

  # The current state should now be Disabled or Permissive
  getenforce
  ```

  > SELinux is not enabled by default on Ubuntu / Debian, so this step can be skipped there.

### 2.2 Appendix: Installing Docker and Docker Compose on Rocky Linux 9

If your production server runs Rocky Linux 9, follow these steps to prepare the container runtime:

#### 1. Remove conflicting or legacy packages

```bash
sudo dnf remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
```

#### 2. Add the official Docker repository

Install the DNF plugin package and add the official Docker CE stable repository (Rocky Linux 9 is fully compatible with the CentOS repository):

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 3. Install Docker Engine and the Compose plugin

This installs the Docker runtime, the CLI client, and the official Docker Compose V2 plugin in one command:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 4. Start the service and enable it at boot

```bash
# Start the Docker daemon
sudo systemctl start docker

# Enable it at boot
sudo systemctl enable docker

# Verify the state is active (running)
sudo systemctl status docker
```

#### 5. Verify the installed versions

```bash
docker --version
docker compose version
```

---

## 3. Server Deployment Procedure (run on the Linux server)

### 3.1 Create the system service account

Before deployment, create the dedicated system account `phishgenix` manually. It owns the data directory and runs the Go backend process:

```bash
# Create a non-login system account (no home directory, interactive login disabled)
sudo useradd phishgenix
```

### 3.2 Upload and extract

Upload the deployment package to `/home/phishgenix` on the server. **Make sure the package matches the server CPU architecture** (`uname -m` returns `x86_64` for `phishgenix-release-amd64-<version>.zip` and `aarch64` for `phishgenix-release-arm64-<version>.zip`; `<version>` is the `VERSION` value at packaging time, e.g. `v1.6.0`):

```bash
# Enter the deployment root directory
cd /home/phishgenix

# Check the local architecture to pick the right package
uname -m

# Extract the package (choose one according to the architecture, replace <version> with the real version)
unzip phishgenix-release-amd64-<version>.zip     # x86_64 servers
unzip phishgenix-release-arm64-<version>.zip     # ARM64 servers
```

> [!IMPORTANT]
> `install.sh` reads the ELF header of `server/phishgenix-server` and verifies it matches the host architecture. A mismatch aborts the script immediately, so you never end up with `exec format error` only after the deployment is finished.

### 3.3 Import offline Docker images (required for intranet / air-gapped environments, optional when online)

If the production server has **no internet access (pure intranet or air-gapped environment)**, `install.sh` cannot pull the MySQL, Redis, Nginx, Postfix, and Mailpit images from public registries during `docker compose up -d`, and the containers fail to start. In that case, import the offline images before running the one-click script.

**① Prepare the offline image bundle**: on a machine that has internet access and a working PhishGenix deployment, export the images with the backup script (see [Backup & Restore](docs/backup-restore.md)):

```bash
/home/phishgenix/deploy/backup/phishgenix-backup.sh --export-images
```

The backup directory (e.g. `backup_20260629_140000/`) then contains an `images/` subdirectory (one `.tar` archive per middleware image) and an `image_metadata.txt` metadata file. Upload that directory to the production server (e.g. `/tmp/restore/backup_20260629_140000`).

**② Run the image-only restore command on the production server**:

```bash
sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000 --images-only
```

The script loads every image archive under `images/`, aligns image names and tags according to `image_metadata.txt`, and then exits without restoring any data.

> [!NOTE]
> This step can be skipped when the server has internet access; `install.sh` pulls the required images automatically during `docker compose up -d`.

### 3.4 Run the one-click configuration script

Run the extracted `install.sh` with `sudo`. It initializes the system and **starts every component in one go**:

```bash
cd /home/phishgenix
sudo ./install.sh
```

> If this machine must also publish the product website (i.e. `/` opens the homepage instead of the console), use
> `sudo ./install.sh --with-site` instead. Without any flag the script **keeps the current state**: a fresh instance
> keeps `/` as the console entry, and an instance that already publishes the website keeps publishing it (an upgrade
> will not silently turn it off) — see 3.7.
> **The script performs the following automatically:**

1. Verify that the system account `phishgenix` exists (it must be created manually per 3.1, otherwise the script exits with an error).
2. Grant execute permission on the Go binary `phishgenix-server`.
3. Create the persistent data directory `/home/phishgenix/data/uploads` and the log directory `/home/phishgenix/server/logs`.
4. Recursively change the owner of `/home/phishgenix` to `phishgenix:phishgenix`.
5. Copy `deploy/phishgenix.service` into the system service directory, reload systemd, and enable it **at boot**.
6. **Instantiate the production configuration automatically**: if `deploy/docker-compose.yml` is missing, generate it from the template and inject random strong passwords, avoiding the risk of overwriting it during upgrades.
7. **Start all services automatically**: run `docker compose up -d`, start the host Go service, and finally **print the initial administrator password and login guidance on screen**.

### 3.5 Verify the production environment variables

The one-click script `install.sh` has **already copied the configuration template and generated secure random high-strength passwords and JWT secrets** during initialization, and has synchronized them into the Docker orchestration file.

If you have no special customization requirements, **you can skip this step entirely — no manual creation or configuration is needed**.

To verify manually or to customize, run:

```bash
cat /home/phishgenix/server/.env
```

Key configuration items and their behavior:

- `APP_ENV=prod`: production environment flag.
- `JWT_SECRET`: a 64-character high-strength random key generated at initialization, used to sign and encrypt tokens.
- `MYSQL_PASSWORD`: a 12-character random high-strength database password generated automatically (and kept in sync with the MySQL container password in `deploy/docker-compose.yml`).
- `FILE_STORAGE_PATH=/home/phishgenix/data/uploads`: the persistent upload directory, created by the script and owned by `phishgenix`.
- `BOUNCE_CALLBACK_SECRET`: the shared secret used to sign bounce/delivery notification callbacks. A 48-character strong key is generated at initialization and used to validate the HMAC-SHA256 signature of bounce callbacks posted by Postfix.
- `BOUNCE_CALLBACK_ALLOWED_IPS`: a comma-separated allow list of intranet IPs/CIDRs whose bounce callbacks are accepted without a signature. The default is `127.0.0.1,172.16.0.0/12,192.168.0.0/16,10.0.0.0/8`. Callbacks from these ranges (such as the locally running Postfix) are accepted directly.

### 3.6 Check service status and start/stop manually (optional)

> [!NOTE]
> **The following steps have already been executed automatically by `install.sh` during initialization.**
> If you need to check the status or restart services manually during later maintenance, use these commands:

- **Manage the Docker containers (MySQL, Nginx, Redis, Postfix, etc.) manually**:

  ```bash
  cd /home/phishgenix/deploy

  # Start all containers
  docker compose up -d
  # Stop all containers
  docker compose down
  # Restart a specific container (e.g. nginx)
  docker compose restart nginx
  ```

- **Manage the Go backend service manually**:

  ```bash
  # Start the backend
  sudo systemctl start phishgenix
  # Stop the backend
  sudo systemctl stop phishgenix
  # Restart the backend
  sudo systemctl restart phishgenix
  # Check that the state is active (running)
  sudo systemctl status phishgenix
  ```

After deployment, open port `80` (or the bound domain) in a browser to access the system:

- Default super administrator account: `superadmin`, password: `superpassword`

### 3.7 Optional: publish the website at the root path (`--with-site`)

The release package bundles `site/` (a static single-page product website). **It is disabled by default**: on customer instances `/` still opens the console.
Enable it explicitly during install/upgrade only when the site must be publicly visible (for example an official demo machine).

The switch is **tri-state**: no flag leaves the current state untouched, and only an explicit `--no-site` retracts the website.

```bash
cd /home/phishgenix
sudo ./install.sh --with-site      # Publish: / opens the website
sudo ./install.sh --no-site        # Retract: / returns to the console (only this flag disables the site)
sudo ./install.sh                  # No flag: keep the current state (recommended for upgrades / re-runs)
# Equivalent environment variable form (unattended / batch initialization):
sudo ENABLE_MARKETING_SITE=1 ./install.sh
sudo ENABLE_MARKETING_SITE=0 ./install.sh
```

| Command       | Currently published                | Currently not published |
| ------------- | ---------------------------------- | ----------------------- |
| `--with-site` | Stays published                    | Publishes               |
| `--no-site`   | **Retracts (takes the site down)** | No change               |
| No flag       | **Stays published** (neither `nginx.conf` nor the override is written) | No change |

> Do not add `--no-site` "just to be safe" during a reinstall or upgrade — that actively takes the website offline. Passing no flag is the safest option. The script decides whether the site is currently published by looking for the `# >>> marketing-site >>>` marker block in `deploy/nginx.conf`, not by checking whether `site/` exists (the release package always ships it).

Behavior differences once enabled:

| Path                    | Site not published                     | Published (`--with-site`)          |
| ----------------------- | -------------------------------------- | ---------------------------------- |
| `/`                     | Console (redirects to `/dashboard`)    | Website homepage                   |
| `/login`                | Login page                             | Login page (website CTAs point here) |
| `/dashboard`            | Console home                           | Console home                       |
| `/training/...`, `/open`, `/click`, `/landing` | Campaign pages | Campaign pages |

---

## 4. Logs and Routine Operations

> Only the most frequently used commands after deployment are listed here.

### 4.1 View host backend logs

The Go backend is managed by systemd, so its standard output is collected by journald. The program also writes categorized log files daily under `/home/phishgenix/server/logs/`.

- **View the systemd journal**:

  ```bash
  # Follow the latest 100 log lines in real time
  sudo journalctl -u phishgenix -n 100 -f
  ```

- **View the local physical log files**:

  ```bash
  # For example, the mail delivery log
  tail -f /home/phishgenix/server/logs/smtp.log.*
  ```

### 4.2 View Docker container logs

- **View the Nginx reverse proxy logs**:

  ```bash
  docker logs -f phish_nginx
  ```

- **View the database / mail delivery logs**:

  ```bash
  docker logs -f phish_mysql
  docker logs -f phish_postfix
  ```
