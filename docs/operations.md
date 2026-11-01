# PhishGenix 运维指南

本指南主要面向系统管理员和运维人员，用于记录 PhishGenix 在 Linux 生产环境下的日常维护命令、系统服务管理、容器管理、数据库访问以及各模块日志文件的分布。

---

## 一、 系统服务管理 (Systemd)

在生产环境中，PhishGenix 后端主程序作为一个 Systemd 系统服务运行。

### 1. 服务状态与控制
* **查看服务状态**：
  ```bash
  sudo systemctl status phishgenix
  ```
* **启动服务**：
  ```bash
  sudo systemctl start phishgenix
  ```
* **停止服务**：
  ```bash
  sudo systemctl stop phishgenix
  ```
* **重启服务**（修改配置或更新可执行文件后使用）：
  ```bash
  sudo systemctl restart phishgenix
  ```
* **设置开机自启**：
  ```bash
  sudo systemctl enable phishgenix
  ```
* **取消开机自启**：
  ```bash
  sudo systemctl disable phishgenix
  ```

### 2. 查看服务控制台日志 (Stdout / Stderr)
后端服务的标准输出被 Systemd 日志守护进程拦截，可通过 `journalctl` 查看：
* **实时追踪滚动日志**（最常用）：
  ```bash
  sudo journalctl -u phishgenix -f
  ```
* **查看最近的 100 行日志**：
  ```bash
  sudo journalctl -u phishgenix -n 100
  ```
* **查看指定时间段的日志**：
  ```bash
  sudo journalctl -u phishgenix --since "2026-06-30 08:00:00" --until "2026-06-30 12:00:00"
  ```

### 3. 服务重启后演练变为「已暂停」的处理（预期行为）
主进程重启时，正在投递中的演练任务会被启动自愈流程自动置为 `paused` 并清理其 Redis 队列，同时把中断瞬间处于发送中的邮件作业重置回待投递状态。这是预期设计：**避免服务器无人值守重启后突然恢复对外发信**。
处理方式：在演练详情页对该任务执行一次「继续投递」即可从断点完整补发剩余邮件，无需暂停等其他操作。若启动日志中出现 `Self-healing: ... reset N interrupted processing mail jobs to pending` 字样，即代表上述孤儿作业已被正确回收。

### 4. 确认当前运行版本
不需要登录管理后台，服务器上即可确认：

| 查看方式 | 命令 | 说明 |
| --- | --- | --- |
| 版本标记文件 | `cat /home/phishgenix/VERSION` | 最直接；`install.sh` 每次安装/升级都会按二进制自报版本重写 |
| 二进制自报版本 | `/home/phishgenix/server/phishgenix-server -version` | **权威来源**，输出形如 `phishgenix-server v1.8.0 (linux/amd64)`。不连数据库、不读 `.env`，配置异常时同样可用 |
| systemd 单元描述 | `systemctl status phishgenix` | 形如 `Description=PhishGenix Server Service v1.8.0` |
| 启动日志首行 | `journalctl -u phishgenix -n 20 --no-pager` | 形如 `Starting server (role=all, version=v1.8.0)...` |

> `phishgenix-cli -version` 同样可用，便于在只保留运维工具的环境里核对版本。

---

## 二、 模块级滚动日志文件

除了系统控制台日志外，PhishGenix 后端内部还将不同业务模块的日志输出到了本地文件。

### 1. 日志存放目录
* 绝对路径：`/home/phishgenix/server/logs/` (即运行目录下的 `logs/` 文件夹)

### 2. 日志命名规则
所有模块日志按天滚动生成，文件命名格式为：
`[module].log.[YYYY-MM-DD]`

### 3. 日志级别控制
模块日志文件的详细程度由环境变量 `LOG_LEVEL` 控制（默认 `info`）：
* `info`（默认）：仅记录常规操作与汇总日志。
* `debug`：额外输出逐条明细日志（如目标人同步每条的新增/更新/删除/移动记录），排查问题时可临时开启。

修改 `/home/phishgenix/server/.env` 中的 `LOG_LEVEL` 后，需重启服务生效：
```bash
sudo systemctl restart phishgenix.service
```

### 4. 各模块日志定义与查看
| 模块名称 | 日志文件名示例 | 记录内容说明 | 快捷查看命令 |
| :--- | :--- | :--- | :--- |
| **邮件投递** | `logs/smtp.log.2026-06-30` | 记录发信成功/失败、网络状态、发件路由等。 | `tail -f /home/phishgenix/server/logs/smtp.log.$(date +%F)` |
| **追踪上报** | `logs/tracking.log.2026-06-30` | 记录用户的打开、点击、数据提交、退信及IP/UA。 | `tail -f /home/phishgenix/server/logs/tracking.log.$(date +%F)` |
| **演练任务** | `logs/campaigns.log.2026-06-30` | 记录演练任务的启动、暂停、终止和调度事件。 | `tail -f /home/phishgenix/server/logs/campaigns.log.$(date +%F)` |
| **发信网关** | `logs/gateways.log.2026-06-30` | 记录发信服务器（SMTP Gateway）的状态和通信。 | `tail -f /home/phishgenix/server/logs/gateways.log.$(date +%F)` |
| **收件人** | `logs/recipients.log.2026-06-30` | 记录收件人导入、脱敏校验、群组修改等记录。 | `tail -f /home/phishgenix/server/logs/recipients.log.$(date +%F)` |
| **模版管理** | `logs/templates.log.2026-06-30` | 记录发信模板及落地页模板的增删改。 | `tail -f /home/phishgenix/server/logs/templates.log.$(date +%F)` |
| **用户与权限**| `logs/users.log.2026-06-30` | 记录管理员登录、密码重置、API 密钥调用等审计。 | `tail -f /home/phishgenix/server/logs/users.log.$(date +%F)` |
| **系统设置** | `logs/settings.log.2026-06-30` | 记录系统全局变量配置、License 续期等修改。 | `tail -f /home/phishgenix/server/logs/settings.log.$(date +%F)` |
| **AI 模型调用** | `logs/ai.log.2026-06-30` | 记录大模型调用的运行过程：选用的模型配置、失败重试与降级、生成结果、结果被丢弃、套餐次数记账异常等。每行都带能力标签 `[AI:full_report]` / `[AI:section_report]` / `[AI:mail_template]` / `[AI:page_template]` / `[AI:micro_course]` / `[AI:course_recommend]`（与用量表 `call_type` 同名），可用 `grep '\[AI:mail_template\]'` 只看某一类。AI 功能异常时优先看这里。 | `tail -f /home/phishgenix/server/logs/ai.log.$(date +%F)` |

---

## 三、 Docker 容器组件管理 (MySQL, Redis, Nginx 等)

系统的辅助中间件运行于 Docker 容器中。

### 1. 容器目录与管理
* 编排目录：`/home/phishgenix/deploy`
* **查看运行中的容器列表**：
  ```bash
  docker ps
  ```

### 2. 容器生命周期控制
*(以下命令均须在 `/home/phishgenix/deploy` 目录下运行)*
* **启动所有辅助容器**：
  ```bash
  docker compose up -d
  ```
* **重启所有辅助容器**：
  ```bash
  docker compose restart
  ```
* **重启特定组件**（例如 Nginx）：
  ```bash
  docker compose restart nginx
  ```
* **停止并销毁容器组件**：
  ```bash
  docker compose down
  ```

### 3. 查看容器组件日志
* **查看 Nginx 反向代理日志**（排查 502/504 等网络问题时使用）：
  ```bash
  docker logs -f phish_nginx
  ```
* **查看 Redis 运行状态**（排查发信及追踪队列堆积时使用）：
  ```bash
  docker logs -f phish_redis
  ```
* **查看 MySQL 数据库日志**：
  ```bash
  docker logs -f phish_mysql
  ```

---

## 四、 访问数据库（MySQL）

业务数据存放在 `phish_mysql` 容器内的 MySQL 8.0 中，库名固定为 `phishgenix`。日常排障以**只读查询**为主：确认数据是否落库、核对演练与追踪记录、检查配置项是否生效。

### 1. 连接信息

| 项目 | 取值 |
| :--- | :--- |
| 容器名 | `phish_mysql` |
| 镜像 / 版本 | `mysql:8.0`（以 `mysql_native_password` 认证） |
| 数据库名 | `phishgenix` |
| 用户名 | `root` |
| 密码 | 以 `/home/phishgenix/deploy/docker-compose.yml` 中的 `MYSQL_ROOT_PASSWORD` 为准 |
| 监听端口 | 宿主机 `3306`（生产模板绑定 `0.0.0.0`，为兼容 Podman 环回路由放开） |
| 数据目录 | 宿主机 `/home/phishgenix/data/mysql`（容器内 `/var/lib/mysql`） |

查看当前密码：

```bash
# 权威值：容器实际初始化时使用的密码
grep -E '^[[:space:]]*MYSQL_ROOT_PASSWORD' /home/phishgenix/deploy/docker-compose.yml

# 应用侧连接密码：首次安装时与上面同源，后续若单独改过则可能不一致
grep -E '^MYSQL_PASSWORD=' /home/phishgenix/server/.env
```

> `install.sh` 只在**首次生成** `docker-compose.yml` 时把随机密码写入 `MYSQL_ROOT_PASSWORD`，文件已存在则跳过替换。因此两处不一致时，以 `docker-compose.yml`（容器真正用于初始化的值）为准。

### 2. 方式一：进入交互式命令行（推荐）

```bash
docker exec -it phish_mysql mysql -uroot -p phishgenix
```

输入上一步查到的密码即可。出现 `Using a password on the command line interface can be insecure` 之类的警告属正常现象，不影响使用。

也可以在编排目录下通过 compose 调用（不依赖容器名，Podman 或改过容器名的环境更稳）：

```bash
cd /home/phishgenix/deploy
docker compose exec mysql mysql -uroot -p phishgenix
```

退出使用 `exit` 或 `Ctrl-D`。

### 3. 方式二：在宿主机直接执行单条 SQL

脚本化排障、需要把结果贴进工单时更顺手：

```bash
PASS=$(sed -n 's/^MYSQL_PASSWORD=//p' /home/phishgenix/server/.env)

docker exec -i phish_mysql mysql -uroot -p"$PASS" phishgenix -e "SHOW TABLES;"
```

要点：
* `docker exec` 这里加 `-i`、**不要加 `-t`**，否则非交互脚本里会因无法分配 TTY 而失败。
* `-p"$PASS"` 传参会让密码短暂出现在 `ps` 输出里；更稳妥的写法是设置 `MYSQL_PWD="$PASS"` 后省略 `-p`。
* 执行 SQL 文件：`docker exec -i phish_mysql mysql -uroot -p"$PASS" phishgenix < /tmp/query.sql`

### 4. 方式三：图形客户端（Navicat / DBeaver / DataGrip）

**不要把 3306 直接暴露到公网。** 生产模板为了让 Podman 环境能连上，把端口绑定成了 `0.0.0.0:3306`；若云主机安全组也放行了 3306，等于把 `root` 账号连同密码交给全网扫描器。正确姿势是先建 SSH 隧道，把远端 3306 映射到本机回环口：

```bash
# 在本地机器执行（本机端口可自选）
ssh -N -L 13306:127.0.0.1:3306 ops@<服务器地址>
```

随后在客户端中连接 `127.0.0.1:13306`，库名 `phishgenix`。用完关闭隧道即可。

若运维策略允许收紧，建议把 `docker-compose.yml` 里的 mysql 端口改为 `127.0.0.1:3306:3306` 后执行 `docker compose up -d` 生效 —— 只有宿主机（以及 SSH 隧道）能连，比依赖安全组更可靠。

### 5. 常用只读查询

```sql
-- 各表体量（读元数据，不扫全表）
SELECT table_name, table_rows, ROUND(data_length / 1024 / 1024, 1) AS data_mb
FROM information_schema.tables
WHERE table_schema = 'phishgenix'
ORDER BY data_length DESC
LIMIT 20;

-- 当前连接与正在执行的语句（排查连接池打满、慢查询）
SHOW PROCESSLIST;

-- 系统设置（产品名、发信速率、跟踪根地址等都在这里）
SELECT setting_key, setting_value FROM system_settings ORDER BY setting_key;

-- 租户与账号
SELECT id, tenant_name, service_level, status FROM tenants ORDER BY id;
SELECT id, tenant_id, username, role, status, login_email FROM users
WHERE deleted_at IS NULL ORDER BY id;

-- 演练任务
SELECT id, tenant_id, task_name, status, created_at FROM campaign_tasks
WHERE deleted_at IS NULL ORDER BY id DESC LIMIT 20;

-- 某次演练的邮件作业分布（pending / processing / sent / failed）
SELECT status, COUNT(*) AS cnt FROM mail_jobs
WHERE task_id = <演练任务ID> GROUP BY status;

-- 最近审计记录
SELECT id, tenant_id, username, module, action, result, action_time
FROM audit_logs ORDER BY id DESC LIMIT 50;
```

> 表名由 GORM 模型名转蛇形复数得到（`CampaignTask` → `campaign_tasks`、`MailJob` → `mail_jobs`、`TrackingRecord` → `tracking_records`）。`users` / `tenants` / `campaign_tasks` / `mail_jobs` 等表带软删除字段，查询时记得加 `deleted_at IS NULL`，否则会把已删除数据算进去。

### 6. 建议：为日常排障单独建只读账号

平时用 `root` 查库风险偏高（一个漏写 `WHERE` 的 `UPDATE` 就足以致命）。建议建一个只读账号，供日常排查与图形客户端使用：

```sql
CREATE USER 'phish_ro'@'%' IDENTIFIED WITH mysql_native_password BY '<强密码>';
GRANT SELECT ON phishgenix.* TO 'phish_ro'@'%';
FLUSH PRIVILEGES;
```

* 指定 `mysql_native_password` 是为了兼容部分老客户端；连新版客户端时可省略 `WITH mysql_native_password`。
* `'%'` 表示任意来源网段，按需收窄为具体网段更安全。只读账号连不上时，先检查宿主机防火墙与云安全组，再确认 `mysql.user` 表中的 `host` 是否匹配客户端来源。
* 该账号只能 `SELECT`，无法执行 `UPDATE` / `DELETE` / DDL，可以放心给排障人员。

### 7. 直接改库的注意事项（重要）

界面上能完成的增删改**一律走界面或 `phishgenix-cli`**，直接写库只用于救急，并且必须清楚以下后果：

* **绕过审计**：界面操作会落 `audit_logs`，SQL 直改不会。事后合规举证时这段改动将查无实据。
* **配置改动不立即生效**：`system_settings`（含系统名称、`smtp_global_send_rate`、追踪外网根地址等）与租户信息带 30 秒进程内缓存，改完最多等 30 秒；`notification_smtp_config`（通知邮件发信配置）是**不带 TTL 的进程内缓存**，只在界面保存时失效 —— 直接改库后必须 `sudo systemctl restart phishgenix` 才会生效。
* **内存态不会自动对齐**：Redis 队列、启动时预加载的活跃追踪 ID、License 状态都在服务启动阶段装载。改库后重启服务是最省事的收敛手段。
* **改动前先备份**：数据库导出与恢复流程见《[备份恢复操作指南](backup-restore.md)》，涉及 `users`、`tenants`、`campaign_tasks` 的批量改动或结构变更尤其如此。
* **不要手改表结构**：Schema 由服务启动时的 GORM `AutoMigrate` 统一管理，手工 `ALTER TABLE` 加字段会被下一轮模型定义覆盖或产生不一致；需要原生 SQL 变更时，应在 `server/migrations/` 下新增带日期的 `.sql` 文件并随发行包交付。

---

## 五、 常见参数调优与故障排查

### 1. 调整高并发演练参数（如 20000 人演练前夕）
需要调整 `/home/phishgenix/server/.env` 配置文件：
* **增大数据库连接池上限**：
  ```ini
  DB_MAX_OPEN_CONNS=500
  DB_MAX_IDLE_CONNS=50
  ```
* **增大邮件投递 worker 并发**（默认 `10`，所有演练共享，钳位 1~100）：
  ```ini
  WORKER_CONCURRENCY=20
  ```
* **压测期间关闭常规 SQL 输出**：
  ```ini
  DB_LOG_LEVEL=error
  ```
* **应用配置**：
  ```bash
  sudo systemctl restart phishgenix
  ```
* **调整发信限速（免重启）**：全局投递速率由数据库配置 `smtp_global_send_rate`（封/分钟，默认 `600` 即 10 封/秒）控制，推荐在超级管理员「系统设置 → 演练配置 → 发信网关策略」中直接调整，也可通过后台运维 CLI 或更新 `system_settings` 表修改，约 30 秒热生效。调大前请确认发信网关的速率与连接配额，避免触发服务商限流造成失败重试。
* **按网关差异化限速**：不同发信服务商的配额不同，可在「发信网关配置」中为每个网关单独设置「发送速率限制」（封/分钟，0=跟随全局默认，1~60000）。例如 A 中继允许 300 封/分钟、B 仅允许 60 封/分钟时分别配置；多个演练共用同一网关时共享该网关配额。

### 2. 验证 Redis 异步队列积压情况
进入 Redis 容器查看未落库 of 追踪事件数量，确认消费者协程消费进度：
```bash
docker exec -it phish_redis redis-cli LLEN phish:queue:tracking_events
```
*(若该数值在演练后短时间内出现数值，说明正在“削峰”，且数值在平滑递减即代表落库正常)*

邮件投递队列相关键位说明：

| Redis 键 | 类型 | 含义 |
| :--- | :--- | :--- |
| `phish:queue:active_campaigns` | Set | 当前有专属投递队列的演练任务 ID 集合 |
| `phish:queue:mail_jobs:{campaignID}` | List | 各演练任务的待投递邮件作业队列 |
| `phish:queue:delayed_jobs` | ZSet | 失败退避中的作业（score = 计划重试的 Unix 时间戳） |

常用检查命令：
```bash
# 查看某演练任务的待投递数量（将 {id} 替换为演练任务 ID）
docker exec -it phish_redis redis-cli LLEN phish:queue:mail_jobs:12

# 查看延迟重试队列中的作业总数
docker exec -it phish_redis redis-cli ZCARD phish:queue:delayed_jobs

# 查看最近一条退避重试的计划时间（最早到期的 5 条）
docker exec -it phish_redis redis-cli ZRANGE phish:queue:delayed_jobs 0 4 WITHSCORES
```

> 说明：单封邮件投递失败后会按指数退避自动重试最多 3 次（30 秒 / 1 分钟 / 2 分钟起，逐次翻倍），期间该作业暂存于 `phish:queue:delayed_jobs`，由系统每 5 秒扫描到期后自动回队，无需人工干预。3 次均失败的作业标记为 failed，可在界面上对演练使用"重新入队"操作。

### 3. 邮件投递队列监控
* **顶栏状态指示（推荐入口）**：超级管理员登录后，顶部导航栏「🛠️ 系统后台」标签右侧会常驻一个队列状态指示条，每 60 秒自动刷新。待投递与延迟重试总数为 0 时恒为绿色常态；总数 ≥ 1000 或最老待投递超过 5 分钟时变为橙色积压告警；≥ 20000 或最老待投递超过 30 分钟时变为红色严重告警（图标脉冲）。作业年龄仅统计进行中演练的待投递作业，已暂停任务的滞留作业属于合法等待恢复，不会触发告警。点击任意时刻可打开右侧抽屉查看完整详情（活跃演练数、Redis 待投递、延迟重试、追踪事件待落库、DB 作业统计、Top-20 积压明细），抽屉内默认开启 15 秒自动刷新。
* **服务日志**：worker 每 5 分钟输出一行聚合快照，可用于趋势观察与告警接入：
  ```
  [QUEUE_OVERVIEW] active_campaigns=3 queued_total=15230 delayed_retries=42
  ```
* **管理端 API**：超级管理员可调用 `GET /api/admin/queue/stats` 获取实时深度数据（活跃任务数、Top-20 队列深度含演练名称与租户、延迟重试数、追踪事件待落库数、DB pending/processing 总数、最老 pending 作业的产生时间），用于判断是否出现投递积压或接入外部监控。

---

## 六、 后台运维命令行工具

租户管理、用户管理（重置密码 / 强制解绑双因素 / 账号状态与角色 / 强制下线）等后台运维操作，
请使用 `phishgenix-cli` 命令行工具。

---

# 🇬🇧 English Documentation

# PhishGenix Operations Guide

This guide is aimed at system administrators and operations staff. It documents the day-to-day maintenance commands, system service management, container management, database access, and per-module log file layout of PhishGenix in a Linux production environment.

---

## 1. System Service Management (systemd)

In production the PhishGenix backend main program runs as a systemd system service.

### 1. Service status and control
* **Check service status**:
  ```bash
  sudo systemctl status phishgenix
  ```
* **Start the service**:
  ```bash
  sudo systemctl start phishgenix
  ```
* **Stop the service**:
  ```bash
  sudo systemctl stop phishgenix
  ```
* **Restart the service** (after changing configuration or replacing the binary):
  ```bash
  sudo systemctl restart phishgenix
  ```
* **Enable at boot**:
  ```bash
  sudo systemctl enable phishgenix
  ```
* **Disable at boot**:
  ```bash
  sudo systemctl disable phishgenix
  ```

### 2. Viewing service console logs (stdout / stderr)
The standard output of the backend is captured by the systemd journal and can be read with `journalctl`:
* **Follow the log in real time** (most common):
  ```bash
  sudo journalctl -u phishgenix -f
  ```
* **Show the last 100 lines**:
  ```bash
  sudo journalctl -u phishgenix -n 100
  ```
* **Show a specific time range**:
  ```bash
  sudo journalctl -u phishgenix --since "2026-06-30 08:00:00" --until "2026-06-30 12:00:00"
  ```

### 3. Campaigns become "Paused" after a service restart (expected behavior)
When the main process restarts, campaigns that are still delivering are set to `paused` by the startup self-healing routine, their Redis queues are cleaned up, and mail jobs that were mid-send at the moment of interruption are reset to pending. This is by design: **it prevents a server that restarted unattended from suddenly resuming outbound sending**.
Remedy: open the campaign detail page and run "Resume delivery" once. The remaining mails are sent from the breakpoint and no other action is needed. If the startup log contains `Self-healing: ... reset N interrupted processing mail jobs to pending`, the orphaned jobs were reclaimed correctly.

### 4. Confirming the running version
No admin console login is required — the version can be confirmed on the server itself:

| How to check | Command | Notes |
| --- | --- | --- |
| Version marker file | `cat /home/phishgenix/VERSION` | Most direct; `install.sh` rewrites it on every install/upgrade using the version the binary reports |
| Version reported by the binary | `/home/phishgenix/server/phishgenix-server -version` | **Authoritative source**; prints e.g. `phishgenix-server v1.8.0 (linux/amd64)`. It neither connects to the database nor reads `.env`, so it still works when the configuration is broken |
| systemd unit description | `systemctl status phishgenix` | Prints e.g. `Description=PhishGenix Server Service v1.8.0` |
| First line of the startup log | `journalctl -u phishgenix -n 20 --no-pager` | Prints e.g. `Starting server (role=all, version=v1.8.0)...` |

> `phishgenix-cli -version` works as well, which is handy in environments where only the operations tooling is retained.

---

## 2. Module-Level Rolling Log Files

Besides the system console log, the PhishGenix backend also writes the logs of individual business modules to local files.

### 1. Log directory
* Absolute path: `/home/phishgenix/server/logs/` (the `logs/` folder under the runtime directory)

### 2. Log naming convention
All module logs roll over daily and are named:
`[module].log.[YYYY-MM-DD]`

### 3. Log level control
The verbosity of the module log files is controlled by the `LOG_LEVEL` environment variable (default `info`):
* `info` (default): only routine operations and summary logs are written.
* `debug`: additionally writes per-item detail logs (e.g. each add/update/delete/move record while syncing recipients). Enable it temporarily when troubleshooting.

After editing `LOG_LEVEL` in `/home/phishgenix/server/.env`, restart the service for it to take effect:
```bash
sudo systemctl restart phishgenix.service
```

### 4. Module log definitions and viewing
| Module | Example log file | Contents | Quick view command |
| :--- | :--- | :--- | :--- |
| **Mail delivery** | `logs/smtp.log.2026-06-30` | Send successes/failures, network state, sender routing, etc. | `tail -f /home/phishgenix/server/logs/smtp.log.$(date +%F)` |
| **Tracking events** | `logs/tracking.log.2026-06-30` | User opens, clicks, data submissions, bounces, plus IP/UA. | `tail -f /home/phishgenix/server/logs/tracking.log.$(date +%F)` |
| **Campaign tasks** | `logs/campaigns.log.2026-06-30` | Campaign start, pause, termination, and scheduling events. | `tail -f /home/phishgenix/server/logs/campaigns.log.$(date +%F)` |
| **SMTP gateways** | `logs/gateways.log.2026-06-30` | Status and communication of the sending servers (SMTP Gateway). | `tail -f /home/phishgenix/server/logs/gateways.log.$(date +%F)` |
| **Recipients** | `logs/recipients.log.2026-06-30` | Recipient imports, desensitization checks, group changes, etc. | `tail -f /home/phishgenix/server/logs/recipients.log.$(date +%F)` |
| **Templates** | `logs/templates.log.2026-06-30` | Create/update/delete of mail templates and landing page templates. | `tail -f /home/phishgenix/server/logs/templates.log.$(date +%F)` |
| **Users & permissions** | `logs/users.log.2026-06-30` | Audit trail of admin logins, password resets, API key usage, etc. | `tail -f /home/phishgenix/server/logs/users.log.$(date +%F)` |
| **System settings** | `logs/settings.log.2026-06-30` | Global system variable changes, license renewals, etc. | `tail -f /home/phishgenix/server/logs/settings.log.$(date +%F)` |
| **AI model calls** | `logs/ai.log.2026-06-30` | Runtime of LLM calls: model configuration selected, retries and fallbacks on failure, generated results, discarded results, quota accounting anomalies, etc. Every line carries a capability tag `[AI:full_report]` / `[AI:section_report]` / `[AI:mail_template]` / `[AI:page_template]` / `[AI:micro_course]` / `[AI:course_recommend]` (identical to `call_type` in the usage table), so `grep '\[AI:mail_template\]'` narrows to one category. Check this first when an AI feature misbehaves. | `tail -f /home/phishgenix/server/logs/ai.log.$(date +%F)` |

---

## 3. Docker Container Management (MySQL, Redis, Nginx, etc.)

The auxiliary middleware of the system runs in Docker containers.

### 1. Container directory and management
* Compose directory: `/home/phishgenix/deploy`
* **List running containers**:
  ```bash
  docker ps
  ```

### 2. Container lifecycle control
*(all commands below must be run in `/home/phishgenix/deploy`)*
* **Start all auxiliary containers**:
  ```bash
  docker compose up -d
  ```
* **Restart all auxiliary containers**:
  ```bash
  docker compose restart
  ```
* **Restart a specific component** (e.g. Nginx):
  ```bash
  docker compose restart nginx
  ```
* **Stop and remove the container components**:
  ```bash
  docker compose down
  ```

### 3. Viewing container logs
* **Nginx reverse proxy logs** (for diagnosing 502/504 network issues):
  ```bash
  docker logs -f phish_nginx
  ```
* **Redis runtime state** (for diagnosing send and tracking queue backlogs):
  ```bash
  docker logs -f phish_redis
  ```
* **MySQL database logs**:
  ```bash
  docker logs -f phish_mysql
  ```

---

## 4. Accessing the Database (MySQL)

Business data lives in MySQL 8.0 inside the `phish_mysql` container and the database name is always `phishgenix`. Day-to-day troubleshooting is mostly **read-only**: confirming that data was persisted, checking campaign and tracking records, and verifying that configuration changes took effect.

### 1. Connection information

| Item | Value |
| :--- | :--- |
| Container name | `phish_mysql` |
| Image / version | `mysql:8.0` (using `mysql_native_password` authentication) |
| Database name | `phishgenix` |
| User name | `root` |
| Password | Per `MYSQL_ROOT_PASSWORD` in `/home/phishgenix/deploy/docker-compose.yml` |
| Listening port | Host `3306` (the production template binds `0.0.0.0` for Podman loopback routing compatibility) |
| Data directory | Host `/home/phishgenix/data/mysql` (container `/var/lib/mysql`) |

To read the current password:

```bash
# Authoritative value: the password the container actually initialized with
grep -E '^[[:space:]]*MYSQL_ROOT_PASSWORD' /home/phishgenix/deploy/docker-compose.yml

# Application-side password: same source at first install, may differ if changed separately later
grep -E '^MYSQL_PASSWORD=' /home/phishgenix/server/.env
```

> `install.sh` writes the random password into `MYSQL_ROOT_PASSWORD` only the **first time** it generates `docker-compose.yml`; if the file already exists the replacement is skipped. So when the two values differ, trust `docker-compose.yml` — the value the container actually initialized with.

### 2. Option 1: interactive shell (recommended)

```bash
docker exec -it phish_mysql mysql -uroot -p phishgenix
```

Enter the password found in the previous step. A warning such as `Using a password on the command line interface can be insecure` is normal and does not affect usage.

You can also go through compose from the deployment directory (independent of the container name; more robust with Podman or when containers were renamed):

```bash
cd /home/phishgenix/deploy
docker compose exec mysql mysql -uroot -p phishgenix
```

Exit with `exit` or `Ctrl-D`.

### 3. Option 2: run a single SQL statement from the host

Handier for scripted troubleshooting or for pasting results into a ticket:

```bash
PASS=$(sed -n 's/^MYSQL_PASSWORD=//p' /home/phishgenix/server/.env)

docker exec -i phish_mysql mysql -uroot -p"$PASS" phishgenix -e "SHOW TABLES;"
```

Key points:
* Use `-i` with `docker exec` and **do not add `-t`**, otherwise it fails in non-interactive scripts where no TTY can be allocated.
* Passing `-p"$PASS"` briefly exposes the password in `ps` output; the safer form is to set `MYSQL_PWD="$PASS"` and omit `-p`.
* Executing an SQL file: `docker exec -i phish_mysql mysql -uroot -p"$PASS" phishgenix < /tmp/query.sql`

### 4. Option 3: GUI clients (Navicat / DBeaver / DataGrip)

**Never expose 3306 to the public internet.** The production template binds `0.0.0.0:3306` so Podman environments can connect; if the cloud security group also allows 3306, the `root` account and its password are handed to every scanner on the internet. The right approach is to create an SSH tunnel first and map the remote 3306 to a local loopback port:

```bash
# Run on your local machine (pick any free local port)
ssh -N -L 13306:127.0.0.1:3306 ops@<server-address>
```

Then connect the client to `127.0.0.1:13306` with database `phishgenix`. Close the tunnel when done.

If your operations policy allows tightening, change the mysql port in `docker-compose.yml` to `127.0.0.1:3306:3306` and apply it with `docker compose up -d` — only the host (and SSH tunnels) can connect, which is more reliable than relying on a security group.

### 5. Common read-only queries

```sql
-- Table sizes (reads metadata, does not scan the tables)
SELECT table_name, table_rows, ROUND(data_length / 1024 / 1024, 1) AS data_mb
FROM information_schema.tables
WHERE table_schema = 'phishgenix'
ORDER BY data_length DESC
LIMIT 20;

-- Current connections and running statements (for connection pool exhaustion and slow queries)
SHOW PROCESSLIST;

-- System settings (product name, send rate, tracking root URL, etc.)
SELECT setting_key, setting_value FROM system_settings ORDER BY setting_key;

-- Tenants and accounts
SELECT id, tenant_name, service_level, status FROM tenants ORDER BY id;
SELECT id, tenant_id, username, role, status, login_email FROM users
WHERE deleted_at IS NULL ORDER BY id;

-- Campaigns
SELECT id, tenant_id, task_name, status, created_at FROM campaign_tasks
WHERE deleted_at IS NULL ORDER BY id DESC LIMIT 20;

-- Mail job distribution of one campaign (pending / processing / sent / failed)
SELECT status, COUNT(*) AS cnt FROM mail_jobs
WHERE task_id = <campaign-task-ID> GROUP BY status;

-- Recent audit records
SELECT id, tenant_id, username, module, action, result, action_time
FROM audit_logs ORDER BY id DESC LIMIT 50;
```

> Table names are the snake_case plural form of the GORM model names (`CampaignTask` → `campaign_tasks`, `MailJob` → `mail_jobs`, `TrackingRecord` → `tracking_records`). Tables such as `users` / `tenants` / `campaign_tasks` / `mail_jobs` carry soft-delete columns, so remember to add `deleted_at IS NULL` — otherwise deleted rows are counted as well.

### 6. Recommendation: create a dedicated read-only account for daily troubleshooting

Running queries as `root` carries unnecessary risk (a single `UPDATE` with a missing `WHERE` is enough to be fatal). Create a read-only account for daily troubleshooting and GUI clients:

```sql
CREATE USER 'phish_ro'@'%' IDENTIFIED WITH mysql_native_password BY '<strong-password>';
GRANT SELECT ON phishgenix.* TO 'phish_ro'@'%';
FLUSH PRIVILEGES;
```

* `mysql_native_password` is specified for compatibility with some older clients; you may omit `WITH mysql_native_password` for recent clients.
* `'%'` means any source network; narrowing it to specific subnets is safer. If the read-only account cannot connect, check the host firewall and cloud security group first, then confirm that the `host` value in the `mysql.user` table matches the client's source.
* The account can only `SELECT` and cannot run `UPDATE` / `DELETE` / DDL, so it can safely be given to troubleshooting staff.

### 7. Cautions about writing to the database directly (important)

Any insert, update, or delete that the UI supports **must go through the UI or `phishgenix-cli`**. Direct database writes are for emergencies only, and you must understand the following consequences:

* **Audit bypass**: UI actions are recorded in `audit_logs`, direct SQL is not. When you later need to prove compliance, that change has no evidence.
* **Configuration changes do not take effect immediately**: `system_settings` (system name, `smtp_global_send_rate`, external tracking root URL, etc.) and tenant information are cached in-process for 30 seconds, so wait at most 30 seconds after changing them. `notification_smtp_config` (notification mail SMTP settings) is **cached in-process without a TTL** and is only invalidated when saved from the UI — after a direct database change you must run `sudo systemctl restart phishgenix`.
* **In-memory state does not realign automatically**: Redis queues, the active tracking IDs preloaded at startup, and license state are all loaded during service startup. Restarting the service after a database change is the simplest way to converge.
* **Back up before changing anything**: for the database dump and restore procedure see the [Backup & Restore Guide](backup-restore.md). This matters especially for bulk changes or schema changes involving `users`, `tenants`, and `campaign_tasks`.
* **Do not modify table structures by hand**: the schema is managed by GORM `AutoMigrate` at service startup; a manually added `ALTER TABLE` column is overwritten by the next model definition or causes inconsistency. When native SQL changes are required, add a dated `.sql` file under `server/migrations/` and ship it with the release package.

---

## 5. Common Parameter Tuning and Troubleshooting

### 1. Tuning parameters for high-concurrency campaigns (e.g. the eve of a 20,000-recipient campaign)
Edit the `/home/phishgenix/server/.env` configuration file:
* **Raise the database connection pool limits**:
  ```ini
  DB_MAX_OPEN_CONNS=500
  DB_MAX_IDLE_CONNS=50
  ```
* **Raise mail delivery worker concurrency** (default `10`, shared by all campaigns, clamped to 1–100):
  ```ini
  WORKER_CONCURRENCY=20
  ```
* **Silence routine SQL output during load tests**:
  ```ini
  DB_LOG_LEVEL=error
  ```
* **Apply the configuration**:
  ```bash
  sudo systemctl restart phishgenix
  ```
* **Adjust the send rate limit (no restart needed)**: the global delivery rate is controlled by the `smtp_global_send_rate` database setting (mails per minute, default `600` = 10 mails per second). We recommend adjusting it in Super Admin → System Settings → Campaign Configuration → SMTP Gateway Policy; it can also be changed through the backend operations CLI or by updating the `system_settings` table, taking effect in about 30 seconds. Before raising it, confirm the gateway's rate and connection quota so that provider throttling does not cause failed retries.
* **Per-gateway rate limits**: providers have different quotas, so each gateway can be given its own "send rate limit" in SMTP Gateway Configuration (mails per minute, 0 = follow the global default, 1–60000). For example, configure 300 mails/minute for relay A and 60 mails/minute for relay B; campaigns sharing a gateway share that gateway's quota.

### 2. Checking the Redis async queue backlog
Enter the Redis container to see how many tracking events have not been persisted yet and confirm consumer progress:
```bash
docker exec -it phish_redis redis-cli LLEN phish:queue:tracking_events
```
*(If the number appears shortly after a campaign, it is being smoothed out; a steadily decreasing value means persistence is working normally.)*

Mail delivery queue keys:

| Redis key | Type | Meaning |
| :--- | :--- | :--- |
| `phish:queue:active_campaigns` | Set | IDs of campaigns that currently have a dedicated delivery queue |
| `phish:queue:mail_jobs:{campaignID}` | List | Pending mail job queue of each campaign |
| `phish:queue:delayed_jobs` | ZSet | Jobs in failure backoff (score = Unix timestamp of the planned retry) |

Common check commands:
```bash
# Number of pending jobs for one campaign (replace {id} with the campaign task ID)
docker exec -it phish_redis redis-cli LLEN phish:queue:mail_jobs:12

# Total number of jobs in the delayed retry queue
docker exec -it phish_redis redis-cli ZCARD phish:queue:delayed_jobs

# Planned retry time of the earliest 5 backoff jobs
docker exec -it phish_redis redis-cli ZRANGE phish:queue:delayed_jobs 0 4 WITHSCORES
```

> A failed delivery is retried automatically with exponential backoff up to 3 times (starting at 30 seconds / 1 minute / 2 minutes, doubling each time). During that period the job stays in `phish:queue:delayed_jobs`, and the system rescans every 5 seconds and requeues it once due — no manual intervention is needed. A job that fails all 3 attempts is marked failed, and the campaign can then be re-queued from the UI using "Requeue".

### 3. Mail delivery queue monitoring
* **Top bar status indicator (recommended entry point)**: after signing in as super admin, a queue status strip sits permanently to the right of the "🛠️ System Backend" tab in the top navigation and refreshes every 60 seconds. It stays green when the sum of pending and delayed retries is 0; it turns orange (backlog warning) when the total reaches 1000 or the oldest pending job is older than 5 minutes; it turns red (severe warning, pulsing icon) at 20000 or older than 30 minutes. Job age only counts pending jobs of running campaigns — jobs stranded in paused campaigns are legitimately waiting to resume and never trigger alerts. Clicking the indicator at any time opens a right-hand drawer with full details (active campaign count, Redis pending, delayed retries, tracking events not yet persisted, DB job statistics, top-20 backlog details); the drawer auto-refreshes every 15 seconds.
* **Service log**: the worker prints an aggregate snapshot every 5 minutes, which is useful for trend watching and alerting integrations:
  ```
  [QUEUE_OVERVIEW] active_campaigns=3 queued_total=15230 delayed_retries=42
  ```
* **Admin API**: super admins can call `GET /api/admin/queue/stats` for real-time depth data (active campaign count, top-20 queue depths including campaign names and tenants, delayed retry count, tracking events not yet persisted, DB pending/processing totals, creation time of the oldest pending job), to judge whether delivery is backing up or to feed external monitoring.

---

## 6. Backend Operations CLI

For backend operations such as tenant management and user management (password reset / force-unbind two-factor authentication / account status and role / force sign-out), use the `phishgenix-cli` command-line tool.
