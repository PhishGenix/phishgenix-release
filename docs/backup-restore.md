# PhishGenix 备份与恢复操作指南

本文档旨在指导系统运维管理员对 PhishGenix 系统的核心配置、数据库数据及上传的用户文件进行定时自动化备份与灾难恢复。
系统灾备策略采用 **定时目录化冷备 + 手动一键脚本恢复**，备份的保存根目录默认为 `/home/phishgenix/backup`。

---

## 1. 灾备架构与策略

- **数据丢失量 (RPO)**：小于 24 小时（取决于定时备份频率，通常每日备份一次）。
- **服务恢复时间 (RTO)**：15 分钟至 1 小时（主要取决于新服务器初始化时间及域名 DNS 解析生效时长）。
- **备份范围与结构**：
  每次备份都会在备份根目录下生成一个专属的**时间戳子目录**（如 `backup_20260629_140000/`），其下保存的文件均使用**固定命名**，从而方便管理和恢复：
  - `phishgenix_data.tar.gz`：包含 Go 服务配置文件 `.env` 以及用户上传目录 `/home/phishgenix/data/uploads`。
  - `phishgenix_db.sql.gz`：动态提取 `.env` 凭据导出的 MySQL 数据库压缩归档。
  - `image_metadata.txt` (可选)：记录镜像的 Image ID 与标签映射关系的元数据文件。
  - `images/` (可选目录)：仅当开启导出参数时生成，内含运行所需的 5 个中间件 Docker 镜像的独立 `.tar` 归档包。通过逐个导出为单包镜像，完美绕过了容器引擎在单次导出多镜像时的底层 Bug。

---

## 2. 自动化备份说明 (`phishgenix-backup.sh`)

备份脚本位于部署包的 `deploy/backup/phishgenix-backup.sh`，支持参数化调用：

### 2.1 脚本参数对照表

| 简写参数 | 全称参数 | 说明  |
| --- | --- | --- |
| `-d` | `--dir PATH` | **指定备份根目录**。不指定时默认输出到 `/home/phishgenix/backup`，并在其下创建 `backup_时间戳` 子目录。 |
| `-e` | `--export-images` | **额外导出 Docker 镜像**。同时将系统运行所需的 5 个中间件镜像逐个导出并存放在备份目录下的 `images/` 子目录中。 |
| `-h` | `--help` | **显示帮助信息**。 |

### 2.2 手工备份示范

1. **执行常规数据备份（推荐日常使用，速度极快，文件极小）**：
  
  ```bash
  /home/phishgenix/deploy/backup/phishgenix-backup.sh
  ```
  
  *生成路径：`/home/phishgenix/backup/backup_20260629_140000/`*
  *该目录下包含 `phishgenix_data.tar.gz` 与 `phishgenix_db.sql.gz`。*
  
2. **执行包含镜像的完整离线备份（适合首次部署归档或重大升级前）**：
  
  ```bash
  /home/phishgenix/deploy/backup/phishgenix-backup.sh --export-images
  ```
  
  *生成的目录下会额外包含 `image_metadata.txt` 元数据以及存储有各容器镜像的 `images/` 目录。*
  

### 2.3 配置每日定时备份 (Crontab)

为避免物理机硬件故障导致备份数据随之丢失，推荐将每日备份同步上传至另一台独立机器或对象存储。以下为配置步骤：

1. 以 `root` 或 `phishgenix` 账号编辑定时任务：
  
  ```bash
  crontab -e
  ```
  
2. 添加以下内容（每天凌晨 2:00 自动执行备份，本地以子目录归档，并自动保留 15 天）：
  
  ```text
  0 2 * * * /home/phishgenix/deploy/backup/phishgenix-backup.sh > /home/phishgenix/logs/backup.log 2>&1
  ```
  
3. **异地转储建议**：
  您可以在备份脚本末尾或在定时任务中增加文件外发指令，例如利用 `rsync` 增量推送整个 backup 目录：
  
  ```text
  10 2 * * * rsync -avz --remove-source-files /home/phishgenix/backup/ backup_user@异地IP:/path/to/backup/
  ```
  

---

## 3. 手动灾后一键恢复说明 (`phishgenix-restore.sh`)

一旦主服务器发生宕机、损坏且无法短期修复，可使用恢复脚本 `deploy/backup/phishgenix-restore.sh` 快速在干净的 Linux 新服务器上恢复环境。

### 3.1 恢复脚本参数对照表

| 简写参数 | 全称参数 | 是否必填 | 说明  |
| --- | --- | --- | --- |
| `-d` | `--dir PATH` | **是** | **备份数据存放的目录路径**（如 `/tmp/restore/backup_20260629_140000`）。 |
|     | `--images-only` | 否   | **仅导入离线镜像模式**。该参数会使脚本仅读取离线镜像并载入、打上正确标签后直接成功退出，不执行后续的配置文件恢复和数据库数据导入。在全新无网环境下做初始化部署时必用。 |
| `-h` | `--help` | 否   | 显示帮助信息。 |

### 3.2 详细恢复步骤

#### 场景 A：在线恢复（新服务器能够连通互联网）

1. **准备新服务器环境**：
  将系统的原始打包发布包（与目标服务器架构匹配的 `phishgenix-release-amd64-<版本>.zip` 或 `phishgenix-release-arm64-<版本>.zip`，`<版本>` 为打包时 `VERSION` 的值）上传至新服务器的 `/home/phishgenix` 目录下，并解压。**恢复时必须使用与原系统一致的版本包** -- 版本号可从包名确认，也可在包内 `VERSION` 文件（解压后为 `/home/phishgenix/VERSION`）或原服务器上执行 `phishgenix-server -version` 核对。
2. **初始化系统**：
  执行一键配置脚本初始化用户与服务项并拉起容器：
  
  ```bash
  cd /home/phishgenix
  sudo ./install.sh
  ```
  
3. **运行恢复脚本**：
  将您的常规备份包（内含 `phishgenix_data.tar.gz` 和 `phishgenix_db.sql.gz`）解压或上传到新服务器（例如 `/tmp/restore/backup_20260629_140000`），并执行恢复：
  
  ```bash
  sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000
  ```
  
  *脚本将自动解密和清洗配置。由于系统已通过 `install.sh` 初始化且镜像已在线拉取，脚本在检测到本地已存在所需镜像后会跳过拉取动作，并自动提取本地当前容器的数据库随机密码合并入恢复的配置文件中，最终一键恢复数据库数据。*

---

#### 场景 B：完全离线恢复（新服务器处于纯内网/隔离环境，且所选备份目录下已有离线镜像目录 `images/`）

在全新内网服务器上直接执行数据恢复，会因为本地没有镜像导致 `install.sh` 初始化容器失败，而容器起不来，恢复脚本也将无法连接数据库。对此，请使用以下**两阶段离线恢复法**：

1. **第一阶段：预导入离线镜像**
  将包含 `images/` 目录的完整备份包上传至新服务器（如 `/tmp/restore/backup_20260629_140000`）。
  在新解压的 `deploy/backup/` 目录下执行只导入镜像的恢复命令：
  
  ```bash
  sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000 --images-only
  ```
  
  *脚本会：使用原生的 podman/docker 载入 `images/` 下的所有离线镜像归档，并根据 `image_metadata.txt` 元数据自动校准这些离线镜像的名称与标签，随后成功退出。*
  
2. **第二阶段：系统初始化**
  进入解压后的发布包根目录，执行初始化安装：
  
  ```bash
  cd /home/phishgenix
  sudo ./install.sh
  ```
  
  *因为第一阶段已经预导入了所有镜像，此时即使在无网环境下，`install.sh` 也能够瞬间利用本地缓存的镜像将所有的 Docker 容器服务完整拉起，并自动在本地生成新的随机高强度数据库密码。*
  
3. **第三阶段：数据完全恢复**
  最后，再次执行恢复脚本，不带 `--images-only` 参数：
  
  ```bash
  sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000
  ```
  
  *脚本会：自动将备份的数据和上传文件恢复至 `/home/phishgenix/` 目录，同时智能提取新系统在第二阶段随机生成的本地数据库密码，并自动合并入恢复出的 `.env` 配置文件与 `docker-compose.yml` 中，最终使用此密码正常连接并注入您的 MySQL 数据库，完美达成灾后离线恢复。*
  

---

## 4. 拉起 Go 后端服务与状态验证

1. 将打包好的 Linux 后端运行二进制程序 `phishgenix-server` 上传并放置到 `/home/phishgenix/server/` 目录下。
2. 启动服务并检查运行状态：
  
  ```bash
  sudo systemctl restart phishgenix
  sudo systemctl status phishgenix
  ```
  
3. **解析切换**：
  数据和后端运行正常后，管理员前往域名控制台，将原解析记录（如 A 记录）修改为新服务器的 IP 地址，即完成最终的容灾切换。

---

# 🇬🇧 English Documentation

# PhishGenix Backup & Restore Guide

This document explains how system administrators perform scheduled automated backups and disaster recovery for the core configuration, database data, and uploaded user files of PhishGenix.
The disaster recovery strategy is **scheduled directory-based cold backup + manual one-click script restore**. The default backup root directory is `/home/phishgenix/backup`.

---

## 1. Disaster Recovery Architecture and Strategy

- **Recovery Point Objective (RPO)**: less than 24 hours (depending on the scheduled backup frequency, typically once per day).
- **Recovery Time Objective (RTO)**: 15 minutes to 1 hour (mainly determined by new server initialization time and DNS propagation for the domain).
- **Backup scope and structure**:
  Each run creates a dedicated **timestamped subdirectory** under the backup root (e.g. `backup_20260629_140000/`), and the files inside always use **fixed names**, which keeps both management and restore simple:
  - `phishgenix_data.tar.gz`: the Go service configuration file `.env` plus the user upload directory `/home/phishgenix/data/uploads`.
  - `phishgenix_db.sql.gz`: a compressed MySQL dump exported dynamically using the credentials from `.env`.
  - `image_metadata.txt` (optional): metadata recording the mapping between image IDs and tags.
  - `images/` (optional directory): generated only when the export flag is enabled. It contains one `.tar` archive per middleware Docker image (5 in total). Exporting each image into its own archive neatly sidesteps the container engine bug that occurs when exporting multiple images in a single run.

---

## 2. Automated Backup (`phishgenix-backup.sh`)

The backup script lives at `deploy/backup/phishgenix-backup.sh` inside the deployment package and supports parameterized invocation:

### 2.1 Script Parameters

| Short | Long              | Description |
| ----- | ----------------- | ----------- |
| `-d`  | `--dir PATH`      | **Specify the backup root directory.** Defaults to `/home/phishgenix/backup`, where a `backup_<timestamp>` subdirectory is created. |
| `-e`  | `--export-images` | **Also export the Docker images.** Each of the 5 middleware images required at runtime is exported individually into the `images/` subdirectory of the backup directory. |
| `-h`  | `--help`          | **Show help information.** |

### 2.2 Manual Backup Examples

1. **Run a standard data backup (recommended for daily use — very fast, very small)**:

  ```bash
  /home/phishgenix/deploy/backup/phishgenix-backup.sh
  ```

  *Output path: `/home/phishgenix/backup/backup_20260629_140000/`*
  *The directory contains `phishgenix_data.tar.gz` and `phishgenix_db.sql.gz`.*

2. **Run a full offline backup including images (for first-deployment archiving or before major upgrades)**:

  ```bash
  /home/phishgenix/deploy/backup/phishgenix-backup.sh --export-images
  ```

  *The resulting directory additionally contains the `image_metadata.txt` metadata file and an `images/` directory holding the container images.*

### 2.3 Configure Daily Scheduled Backup (Crontab)

To avoid losing backups together with a failed physical machine, we recommend uploading the daily backup to a separate machine or object storage. Configuration steps:

1. Edit the crontab as `root` or `phishgenix`:

  ```bash
  crontab -e
  ```

2. Add the following line (runs the backup every day at 02:00, archives into a local subdirectory, and keeps 15 days of history):

  ```text
  0 2 * * * /home/phishgenix/deploy/backup/phishgenix-backup.sh > /home/phishgenix/logs/backup.log 2>&1
  ```

3. **Off-site copy recommendation**:
  You can append a transfer command at the end of the backup script or as another cron entry, for example using `rsync` to incrementally push the whole backup directory:

  ```text
  10 2 * * * rsync -avz --remove-source-files /home/phishgenix/backup/ backup_user@remote-ip:/path/to/backup/
  ```

---

## 3. Manual Post-Disaster One-Click Restore (`phishgenix-restore.sh`)

If the primary server goes down or is damaged beyond a short-term repair, use the restore script `deploy/backup/phishgenix-restore.sh` to rebuild the environment quickly on a clean new Linux server.

### 3.1 Restore Script Parameters

| Short | Long            | Required | Description |
| ----- | --------------- | -------- | ----------- |
| `-d`  | `--dir PATH`    | **Yes**  | **Path of the directory holding the backup data** (e.g. `/tmp/restore/backup_20260629_140000`). |
|       | `--images-only` | No       | **Image-import-only mode.** The script only loads the offline images, applies the correct tags, and exits successfully without restoring configuration files or importing database data. Required when initializing a brand-new air-gapped environment. |
| `-h`  | `--help`        | No       | Show help information. |

### 3.2 Detailed Restore Steps

#### Scenario A: Online restore (the new server has internet access)

1. **Prepare the new server environment**:
  Upload the original release package (`phishgenix-release-amd64-<version>.zip` or `phishgenix-release-arm64-<version>.zip` matching the target architecture; `<version>` is the `VERSION` value at packaging time) to `/home/phishgenix` on the new server and extract it. **The restore must use the same version package as the original system** — confirm the version from the package name, from the `VERSION` file inside the package (`/home/phishgenix/VERSION` after extraction), or by running `phishgenix-server -version` on the original server.
2. **Initialize the system**:
  Run the one-click script to create the user and service and to start the containers:

  ```bash
  cd /home/phishgenix
  sudo ./install.sh
  ```

3. **Run the restore script**:
  Extract or upload your regular backup package (containing `phishgenix_data.tar.gz` and `phishgenix_db.sql.gz`) to the new server (e.g. `/tmp/restore/backup_20260629_140000`) and run the restore:

  ```bash
  sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000
  ```

  *The script decrypts and sanitizes the configuration automatically. Because the system was already initialized by `install.sh` and the images were pulled online, it detects the required local images and skips pulling; it then extracts the random database password of the current local container and merges it into the restored configuration file, and finally restores the database data in one step.*

---

#### Scenario B: Fully offline restore (the new server is in a pure intranet / air-gapped environment and the selected backup directory already contains the offline image directory `images/`)

Running the data restore directly on a brand-new intranet server fails because there are no local images: `install.sh` cannot start the containers, and without containers the restore script cannot reach the database. Use the following **two-phase offline restore method** instead:

1. **Phase 1: Pre-import the offline images**
  Upload the complete backup package that contains the `images/` directory to the new server (e.g. `/tmp/restore/backup_20260629_140000`).
  In the freshly extracted `deploy/backup/` directory, run the image-only restore command:

  ```bash
  sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000 --images-only
  ```

  *The script loads every offline image archive under `images/` using the native podman/docker runtime, calibrates their names and tags according to the `image_metadata.txt` metadata, and then exits successfully.*

2. **Phase 2: Initialize the system**
  Enter the extracted release package root and run the initialization:

  ```bash
  cd /home/phishgenix
  sudo ./install.sh
  ```

  *Because all images were pre-imported in phase 1, `install.sh` can start every Docker container instantly from the local cache even without internet access, and it generates a new random high-strength database password locally.*

3. **Phase 3: Fully restore the data**
  Finally, run the restore script again without the `--images-only` flag:

  ```bash
  sudo /home/phishgenix/deploy/backup/phishgenix-restore.sh -d /tmp/restore/backup_20260629_140000
  ```

  *The script restores the backed-up data and uploaded files to `/home/phishgenix/`, extracts the random database password newly generated by the system in phase 2, merges it automatically into the restored `.env` file and `docker-compose.yml`, and then connects with that password to import your MySQL database — completing a fully offline post-disaster restore.*

---

## 4. Start the Go Backend Service and Verify Status

1. Upload the packaged Linux backend binary `phishgenix-server` to `/home/phishgenix/server/`.
2. Start the service and check its status:

  ```bash
  sudo systemctl restart phishgenix
  sudo systemctl status phishgenix
  ```

3. **DNS cutover**:
  Once the data and the backend are running normally, the administrator updates the original DNS record (e.g. the A record) in the domain console to point to the new server IP, which completes the final failover.
