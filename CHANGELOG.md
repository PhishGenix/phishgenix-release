# 更新日志

本文件记录面向使用者的版本变更，按版本倒序排列。

> 内部实现细节、缺陷根因分析与文档维护记录不在本文件维护。

---

## [v1.9.2] - 2026-11-03

### 新增

- **演练环境自检（超管侧）**：超管仪表盘新增「系统配置检查」9 项（授权与租户 5 项、系统配置 4 项），每项可「去配置」一键跳转对应设置；租户管理员同步补上该入口，操作员不显示无权访问的入口。
- **内置免费授权提示**：超管首次登录弹出可用范围清单，可勾选「下次不再提醒」；顶栏新增内置授权提示，切换到正式授权后自动消失；服务计划到期提醒同步改版。
- **官网**：支持中英文切换，与控制台共用语言设置；页脚新增「联系我们」，「私有化部署」补软件包下载与安装指南；首页文案更新为「AI 驱动的钓鱼邮件演练与安全意识培训平台」。

### 变更

- **内置免费授权收紧外链与自定义内容**：未激活或过期实例只能使用追踪链接（模板按钮与文本链接不可指向任意地址），自定义 HTML、脚本、二维码、外链及新增附加域名一并关闭；存量附加域名仍可使用或移除，只是不可新增。公共模板上限由 50 套上调至 100 套。
- **顶栏状态标签统一**：「系统后台」「租户」「发信队列」外观统一，emoji 图标替换为矢量图标。

### 修复

- 追踪外网根地址与跨域来源不再带默认值兜底，避免演练链接与打开像素失效（未填写时仅打印告警、不阻断）。
- 收紧授权状态信息：客户名称与授权签发/到期时间现仅登录后可见，匿名访问不再返回。
- 修复换域名后微课堂图库与模板缩略图坏图，域名变更后无需修复存量数据。
- 免费试用申请页移动端工具条改为贴顶显示。

### 升级注意

- **请核对「演练追踪外网根地址」**：不再有默认值兜底，库中旧值不会自动清理；未配置时演练邮件链接与打开像素失效，且仅在后端日志提示。
- **内置授权收紧不追溯存量**：正式授权到期回落内置授权后，含自定义外链或 HTML 的模板需清理后才能继续编辑。

---

## [v1.9.1] - 2026-11-01

### 变更

- **用户管理改为左租户栏 + 右用户列表**：右侧只显示所选租户的用户，点「全部租户」回到全局视角；租户栏支持按名称、域名或租户 ID 过滤，输入后回车直接选中。
- 新建用户默认落在左侧选中的租户，停在「全部租户」时留空需显式选择，避免租户建错归属后只能删号重建。
- **品牌名统一跟随系统名称**：官网页面的品牌名、版权、标题、description 与分享卡片标题均读取「系统设置 → 系统名称」，私有化部署可替换为客户自己的名称。
- 控制台浏览器标签标题与登录页大标题未自定义时按「系统名称 + 演练控制台」生成，版权按系统名称与当前年份拼接，不再写死产品名。
- 官网版式微调：锚点跳转后区块留白收紧、首屏移除副标题标签、各区块标语改为正式表述、页脚移除版本号。

### 修复

- 官网刷新不再闪出默认品牌名：先隐藏品牌名，取到系统设置后再显示，并缓存到本地供后续刷新直接套用。
- 修复 `install.sh` 数据库预检误判，避免「库已存在」被判为不存在、空服务器被判为已有其他库而终止安装。
- 目标库不存在、本机却残留旧数据目录时，安装脚本会明确提示「这更像旧数据未迁移，继续将建出空库」。

---

## [v1.9.0] - 2026-10-31

### 新增

- **营销官网（可选启用）**：纯静态产品官网，可与产品同机部署，改文案无需重新构建；`install.sh --with-site` 发布、`--no-site` 收回、不带参数保持现状。官网文件始终随产品提供，客户私有化实例默认不启用，`/` 仍进控制台。
- **演练任务定时启动**：可预约启动时间（不早于下一分钟整）并支持取消，到点由后台自动启动，失败会发通知；列表新增「已预约」状态。
- **开箱即用的内容库**：首次初始化随包导入 85 套诱饵邮件模板与 85 个配套落地页，以及 22 门微课堂课程、107 道题与配图，装好即可直接搭演练。
- **版本可见**：`phishgenix-server -version` 与 `phishgenix-cli -version` 不读数据库、不读配置文件，配置异常时同样可用；启动日志首行带版本号。
- `phishgenix-cli tenant update` 新增域名、配额、演练审批、可信四组参数。

### 变更

- **产品更名为 PhishGenix**：界面文案、邮件与报告、帮助文档、程序与服务名、默认安装路径、数据库名等统一改名。
- 系统设置改为子标签页（安全设置、系统授权各自拆分），各块独立保存。
- 入口收敛：免费试用入口只保留在营销官网，官网内「登录」统一表述为「控制台」。
- 用户管理操作列固定为三项，「更多」承载低频操作；历史与已停用服务计划收敛为「仅可删除」。

### 修复

- 修复全新机器上安装失败或输出突然截断：MySQL 就绪改为按容器生命周期判定并计时等待，迁移失败最多重试 3 次。
- 修复升级后「部署成功却登录不了」：启动后端前核对目标库，缺失且无其他业务库时自动建库，情况不明时终止安装并打印处理指引。
- 修复未配置「系统访问根地址」时仍能保存演练，导致员工提交后拿不到学习链接。
- 修复已发布官网的实例重跑安装脚本会把官网关掉。
- 修复作业状态落库锁冲突导致演练被无故暂停，以及自动重试链路从未真正生效的问题。
- 修复回收站删除租户后域名仍被占用。
- 修复计次套餐被错误拦截（演练人次上限与授权单场人数上限混用）。

### 升级注意

- **产品更名属破坏性变更**：旧部署标识、旧学习链接、旧 DNS 核验记录与旧备份档名均不兼容，已有环境建议按全新部署处理。
- **首页地址变化**：产品首页从 `/` 变为 `/dashboard`；未启用官网时 `/` 自动跳转，老书签照旧可用。
- **官网默认不启用**：已启用官网的实例升级时不要顺手加 `--no-site`，否则会被收回。
- **内置模板与课程只导入一次**：后续版本更新内置内容对已安装实例无效，管理员主动删除的内容也不会恢复。
- **自注册默认开放**：新装实例默认开启自注册（`ALLOW_SELF_REGISTRATION=true`），已存在的配置不会被覆盖。

---

## [v1.8.0] - 2026-10-21

### 新增

- **内置免费授权**：不导入授权证书也能完成初始化并直接使用，按内置额度运行（1 个租户、5 个用户、1 个发信网关、单次演练 50 人、累计 500 人、同时 1 场演练）。内置授权仅开放自定义模板，AI、模板导入与克隆、备份恢复、报表导出、界面定制、多语言、微课堂、自注册均关闭。
- 正式授权到期后不再直接停服，而是回落内置额度继续运行，系统设置页顶部会明确提示当前处于内置免费授权。
- **微课堂、自注册纳入授权范围**，未授权时相关入口与功能一并不可用；新建或修改演练时学习环节只剩「自定义外链」可选，存量任务保留原有配置。
- **AI 能力纳入授权门控**，未开通时「统计分析 → AI 用量」标签置灰并给出原因。

### 变更

- 授权判定统一为「证书已激活且开关为真」，未激活、已过期、时钟回拨一律按未授权处理。
- 系统设置 → 授权页增加微课堂、自注册两项状态；未提供证书时来源显示为「内置免费授权」。
- 仓库不再内置 demo 授权文件，本地开发环境同样按内置授权运行。

### 修复

- 富文本渲染补做 HTML 清洗（法律文档、微课程编辑与浏览、门户课程大厅），修复从外部导入的课程内容可注入脚本的问题；白名单不放行 `iframe` / `script` / `style`，`data:` 内联资源同样拦截。
- 登录页的免费试用入口不再延迟出现。

### 升级注意

- 正在使用微课堂或自助注册的环境，需重新签发含对应开关的证书，旧证书缺这两项会按未授权处理；自助注册还需部署层 `ALLOW_SELF_REGISTRATION` 同时为真。
- 未导入正式证书时按内置额度运行，已超出额度的数据不会被删除，但相关新增操作会被拒绝。
- 富文本中若曾用 `iframe` 嵌入或 `data:` 内联图片，升级后不再渲染，需改为外链图片。

---

## [v1.7.0] - 2026-10-18

### 新增

- **中招—学习—测验闭环**：学习链接带签名令牌，免登录直接作答课后测，提交即出成绩与解析，只认首次提交，重复作答不刷分。
- 演练报告新增四张学习指标卡（已学习、未学习、学习参与率、课后测全对人数）与中招人员学习明细，支持服务端分页与一键导出 Excel。
- **演练学习环节可配置**：不配置学习、自动推荐课程、手动选课、自定义外部链接四种形态；手动选课只列「已发布且已配置课后测」的课程。
- **演练环境自检**：一键并发检查服务计划次数、租户可信状态、发信网关、目标与分组，逐项给出就绪状态与下一步动作，只读可重复执行。
- 目标人导入模板补齐手机号、职位，支持「导入到指定公司」；批量修改归属时公司必选、部门可选。
- 系统设置新增「全局设置」，集中配置系统访问根地址，用于学习链接、通知深链、自注册激活链接、模板缩略图等长期在线地址。

### 变更

- 用户管理列表的「手机号」列改为「邮箱」，搜索新增「邮箱」选项。
- 课后小测题数上限由 5 道放宽到 10 道，随机抽取默认由 3 道改为 5 道。

### 修复

- 发信网关的「发信账号邮箱」不再豁免系统自带发信服务，与第三方网关同一套域名校验。
- 安装脚本数据库迁移改为单文件幂等执行，修复密码提取时把行内注释当成密码导致迁移静默失败的问题。
- 导入 CSV 时，部门路径首段与公司同名不再重复建出同名层级。
- 动态令牌绑定时，验证器显示的账号优先取登录邮箱，不再显示随机用户名。
- 发信网关测试信与自注册激活信改为 HTML + 纯文本双版本，修复中文主题在部分邮件服务器上乱码。

### 升级注意

- 此前依赖「微课堂根地址留空即回落追踪域名」的环境，升级后必须在全局设置中配置该地址，否则学习链接等长期地址拼不出来（无回落是有意设计）。

---

## [v1.6.0] - 2026-10-14

### 新增

- **免费试用自注册**：登录页新增「免费试用」入口，填写企业与联系人信息、通过图形验证码后提交，收到确认邮件并设置密码即可开通；是否需要人工审核由超级管理员配置。
- 采用「先验证、后建号」：提交阶段只登记申请单，域名与账号归属在打开确认邮件的那一刻判定，避免域名被空壳申请占住。
- 超级管理员新增「自注册管理」：申请单列表与状态计数、详情、放行、拒绝、重发确认信。
- 系统设置新增「自注册」标签页（开通模式、频控阈值、域名黑名单）与「技术支持设置」。
- 注册与激活全程防枚举、防滥用，对外不透露某个域名或邮箱是否已注册。
- **发信账号限定**：「发信账号邮箱」必须是完整邮箱，域名须落在租户自有域名或当前服务计划允许的目标域内。

### 变更

- 新建租户与自注册开通的最大操作员数统一默认为 1。
- 「内网地址限制」由常驻说明改为悬浮提示，长文案提示统一限宽 360px。

### 修复

- 修复登录页协议文字在残留浅色主题偏好时几乎不可读。
- 修复租户改级为试用后残留原服务计划、导致试用限制形同虚设。
- 修复「创建即试用」的租户没有带出服务计划、连一个目标人都无法创建。
- 修复 SMTP 配置页必填星号被挤到单独一行。

---

## [v1.5.0] - 2026-10-13

### 新增

- **登录邮箱绑定**：绑定后可用「邮箱 + 密码」登录，也支持继续用用户名登录；解绑改为验证码两段式确认，验证码 60 秒冷却、每日 10 次、错 5 次作废。
- 管理员可在用户管理页直接为用户改绑或清空登录邮箱。
- 新增「试用」服务级别（最低档），租户管理支持逐个调整服务级别与配额。
- **租户可信与域名核验**：未可信租户启动演练会被闸口拦截，修改租户域名自动作废已有核验。
- 新增轻量追踪角色 `-role=tracking`，追踪接收服务可与管理面分开部署。
- 登录前须勾选同意《隐私政策》与《产品服务协议》，协议改版后自动要求重新确认并记录确认版本与时间。
- 系统设置新增「演练草稿任务上限」（默认 50，配置 0 为不限）。

### 变更

- 通知邮件信封改为朴素排版，与发信网关测试信一致。
- 演练结束时间统一按当天 23:59:59 收口。
- 定时任务空转收敛，空闲期数据库访问大幅下降，活动期行为不变。

### 修复

- 修复服务端日志被写入系统日志文件、导致其持续膨胀。

---

## [v1.4.0] - 2026-10-11

### 新增

- **消息通知 · 邮件通道**：系统设置新增「通知设置」，可配置服务器、端口、发件人邮箱、密码、发信名称、邮件语言与发信速率；端口 25 / 465 / 587 自动匹配加密方式。
- 「测试」按钮固定自发自收，失败时直接回显失败环节与邮件服务器原始应答。
- 覆盖 8 类通知的中英双语主题与正文；通知邮件与演练发信完全隔离，不混入演练统计，也不被退信回调污染。
- 投递可靠性：仅对可恢复失败重试最多 2 次，重投前重新校验收件人是否仍可接收，通道级令牌桶限流默认 30 封/分钟。
- 消息中心支持勾选删除，只影响自己的那条消息。
- 用户新增「通知邮箱」字段，个人设置中的邮件通道开关可用。

### 变更

- 消息中心从「系统设置」迁到「个人设置」；操作员侧边栏收起「系统设置」入口。
- 内网访问开关收敛为一个「允许访问内网」总开关（默认关闭），并补上租户发信网关的内网地址校验；系统自带发信服务不受该开关约束。

### 修复

- 通知邮件的主题、正文与发信人展示名统一取「系统名称」，改名后不再出现两套名字。
- 修复消息列表总数重复扣减、导致翻到最后一页是空的。
- 全仓清理散落的旧产品名，本地存储键前缀随之改名，升级后异地登录提醒可能重新提示一次。

### 升级注意

- 旧的内网开关一律不再读取，升级后内网访问回到关闭状态；升级前开过任意一个的，需超级管理员在界面上重新开启一次。

---

## [v1.3.0] - 2026-10-10

### 新增

- **消息通知模块**：站内消息中心上线，右上角铃铛与未读徽标，WebSocket 实时推送、轮询兜底；覆盖演练审批、演练状态、服务计划到期三类通知。
- 通知范围开关化，分平台级与用户级两层，语义为「无记录即开启」，升级后不会把既有用户静默关掉。
- **统计分析模块**：AI 用量（多维度聚合 + 调用明细分页）与参演目标分析（重复中招分布、风险等级分布、综合安全评分）。
- 演练活动支持超级管理员按租户筛选，状态统计卡与列表同步收敛。
- 课程列表新增「学习次数」列；右上角新增角色标识；落地页移动端自适应。

### 变更

- 任务状态统一为七档口径（草稿 / 投递中 / 投递完成 / 待正式投递 / 已暂停 / 已结束 / 已终止），列表与统计卡同源。
- 下线遗留的租户级到期时间字段，租户可用期统一由套餐服务期承载。

### 修复

- 修复「投递中」统计漏行，以及报告「进行中」横幅未展示结束时间与剩余时长。
- 修复分组成员抽屉的请求风暴：5000 人分组打开一次从约 50 个请求降至 1 次。
- 修复消息详情抽屉在深色模式下不可读、通知深链无反应、批量已读在空参数下会全部标为已读。
- 升级不会覆盖现场修改过的配置与证书（仅提供 `*.example` 模板）；`install.sh` 自动为存量 Nginx 配置补齐 WebSocket 段。

---

## [v1.2.0] - 2026-10-07

### 新增

- **微课堂课后小测与题库**：支持手动指定与按标签随机两种组卷方式，发布前可预览抽题结果并诊断题量是否够抽。
- 题库支持单选题、多选题、判断题，中英双语题干与解析、难度标记、启停用与作答预览；可关联课程作为专属题。
- 支持从课程正文 AI 出题，设定题型配比与目标考点后生成候选题目，确认后逐条入库。
- **AI 模型调度**：一个 API 基地址下可挂载多个候选模型，按顺序依次尝试，排序语义为数字越大越优先。
- 回收站新增题库与模板中心入口；管理端「系统用户」与「用户管理」合并为一档。

### 修复

- 安全：封堵课程库备份包导入的任意文件写，并为回收站「彻底删除」补充状态守卫。
- 修复 AI 出题每次只返回单个标签。
- 微课堂图库改为「加载更多」，修复宽屏右侧留白；门户页头搜索框、面包屑与页脚在窄屏下的适配。

---

## [v1.1.0] - 2026-10-06

### 新增

- **微课堂模块上线**：超管管理端 + 免登录课程门户，前台路径统一为 `/training`；支持正文图片上传与图库、AI 生成课程、分类与标签（含英文名）。
- 操作员演练任务审批机制，演练列表审批状态独立成列。
- 演练支持 AI 推荐微课程，落地页学习链接可跟随推荐；微课堂根地址独立配置，与演练追踪根地址解耦。
- 支持租户级自配 AI 模型；AI 运行日志落盘并带能力标签，便于按业务类型排查。
- 目标人同步新增 Coremail APIWS V3 协议支持。
- 新增内置课程批量生成工具 `phishgenix-batch-courses`，随产品提供。

### 修复

- 修复微课堂切到空栏目时整页抖动、标签删除后遗留孤儿关联、图库中文文件名显示异常等界面问题。
- 修复登录页默认标题在英文环境下仍显示中文。

---

## [v1.0.1] - 2026-09-05

### 新增

- 报告编辑器支持所见即所得分页（编辑态即打印态），配套大纲跨章拖拽、撤销/重做、版本历史与并发冲突检测。
- AI 生成演练报告改为结构化返回，生成入口收敛至编辑器工具栏。
- 邮件队列实时监控：顶栏队列状态指示与监控抽屉详情。
- SMTP 网关支持按网关独立发送限速，全局默认速率移入系统设置的演练配置。
- 落地页提交响应支持「仅提示提交成功」模式，提交后展示内容全部可自定义。
- 演练仪表盘统计卡拆分为八项互斥口径，区分任务生命周期与投递进度。

### 修复

- 修复服务重启后孤儿作业导致演练投递卡死，并加固投递队列防积压。
- 修复队列为空时监控仍显示红色「严重积压」。
- 修复重复提交落地页表单会覆盖首次提交数据（现保留首次并记录处理状态）。
- 修复报告导出 PDF 的自动分页与文本框宽度自适应问题。
- 其他：操作员可见范围纳入授权动态分组成员，编辑器与深色模式若干样式修复。

---

## [v1.0.0] - 2026-08-18

首个正式发布版本，主要能力：

- 演练任务：邮件演练全流程（任务创建、SMTP 网关发送、进度控制、中断恢复）
- 行为追踪：邮件打开、链接点击、落地页提交、二维码扫描与退信回调
- 邮件模板与落地页：可视化编辑器、提交响应自定义、批量模板生成工具
- 收件人管理：部门/分组维护、批量导入
- 演练分析报告与导出
- 多租户隔离、分级角色授权与操作审计日志
- RSA 签名 License 授权机制
- 帮助中心（中/英文）
- 一键部署（install.sh + Docker Compose + Nginx）与后台运维命令行工具

---

# 🇬🇧 English Documentation

# Changelog

This file records user-facing changes, newest first.

> Internal implementation details, root-cause analyses, and documentation maintenance notes are not tracked here.

---

## [v1.9.2] - 2026-11-03

### Added

- **Campaign environment self-check (super admin)**: the super admin dashboard gained a System Configuration Check with nine items (five for license and tenant, four for system configuration), each with a "Go to settings" link; tenant administrators get the same entry, and operators no longer see entries they cannot access.
- **Built-in license notice**: super admins see the available scope on first login with a "Don't remind me again" option; a notice beside "Admin Console" marks the built-in license and disappears after a commercial license is activated, and the service plan expiry reminder was redesigned.
- **Marketing site**: Chinese and English switching sharing the console's language setting; a "Contact us" footer using the support email, software download and installation guide on the "Private deployment" card, and home page copy updated to "AI-powered phishing simulation and security awareness training platform".

### Changed

- **Built-in license restricts external links and custom content**: inactive or expired instances can use tracking links only (template buttons and text links cannot point to arbitrary addresses); custom HTML, scripts, QR codes, external links, and new additional domains are disabled, while existing additional domains can still be used or removed but not added. The public template limit rose from 50 to 100.
- **Top-bar status labels unified**: "Admin Console", "Tenants", and "Mail Queue" share one look, with emoji icons replaced by vector icons.

### Fixed

- The tracking root URL and CORS origins no longer fall back to a default, preventing campaign links and open pixels from breaking (a warning is logged without blocking when empty).
- License status now returns the customer name and license issue and expiry times only to signed-in sessions; anonymous requests no longer receive them.
- Fixed broken micro-course images and template thumbnails after a domain change, with no data repair needed.
- The free-trial application page's mobile toolbar now sits at the top.

### Upgrade Notes

- **Review the tracking root URL**: it no longer has a default fallback, and old persisted values are not cleaned up; without it, campaign email links and open pixels fail and only a backend log records it.
- **The built-in license restrictions are not retroactive**: after a commercial license expires and the system falls back, templates with custom external links or HTML must be cleaned up before they can be edited again.

---

## [v1.9.1] - 2026-11-01

### Changed

- **User management now uses a tenant sidebar on the left and a user list on the right**: the right pane shows only the users of the selected tenant, and "All tenants" returns to the global view. The sidebar filters by name, domain, or tenant ID and selects a tenant when you press Enter.
- New users default to the tenant selected in the sidebar; with "All tenants" active the field stays empty and must be chosen explicitly, so accounts are no longer created under the wrong tenant — a mistake that previously required deleting and recreating the account.
- **Branding now follows the system name**: the marketing site's brand name, copyright line, page title, description, and share-card title all read System Settings → System Name, so on-premises deployments can display the customer's own name.
- The console browser tab title and login page heading are generated from "<system name> + Simulation Console" when not customized, and the copyright line is composed from the system name and the current year instead of a hardcoded product name.
- Marketing site layout refinements: tighter top spacing after anchor jumps, subtitle removed from the hero, section copy rewritten in a more formal tone, and the version number removed from the footer.

### Fixed

- The marketing site no longer flashes the default brand name on reload: the name stays hidden until system settings load, and is cached locally so returning visitors see it immediately.
- Fixed `install.sh` misjudging the database pre-check, which could report an existing database as missing, or abort on an empty server as if another database were present.
- When the target database is missing but the host still holds an old data directory, the installer now warns that this looks like unmigrated old data rather than a fresh deployment, and that continuing would create an empty database.

---

## [v1.9.0] - 2026-10-31

### Added

- **Marketing website (optional)**: a static product site that can be deployed on the same machine as the product, with copy changes requiring no rebuild. `install.sh --with-site` publishes it, `--no-site` retracts it, and no flag keeps the current state. The site files always ship with the product; customer on-premises instances keep it disabled and `/` still opens the console.
- **Scheduled campaign start**: campaigns can be booked to start at a future time (not earlier than the next full minute) and the booking can be cancelled. The scheduler starts them automatically, and a failure sends a notification. The list gained a "Scheduled" state.
- **Ready-to-use content library**: the first initialization imports 85 lure email templates with 85 matching landing pages, plus 22 micro-courses, 107 questions, and their images, so a campaign can be built immediately after installation.
- **Version visibility**: `phishgenix-server -version` and `phishgenix-cli -version` neither read the database nor the configuration file, so they work even when the configuration is broken. The first line of the startup log carries the version.
- `phishgenix-cli tenant update` gained domain, quota, campaign approval, and trusted parameters.

### Changed

- **Product renamed to PhishGenix**: UI copy, emails and reports, help documentation, program and service names, default install path, and database name all use the new name.
- System settings were reorganized into sub-tabs (security settings and system license are each split), and each block saves independently.
- Entry points consolidated: the free-trial entry now lives only on the marketing site, and "Login" is uniformly labelled "Console" there.
- The user management action column is fixed to three items with the rest under "More"; historical and disabled service plans are restricted to deletion only.

### Fixed

- Fixed installation failures or abruptly truncated output on brand-new machines: MySQL readiness is now judged by container lifecycle with timed waits, and migrations retry up to 3 times.
- Fixed "deployment reported success but login fails" after upgrades: the target database is verified before the backend starts, created automatically when absent and no other business database exists, and installation aborts with clear guidance when the situation is ambiguous.
- Fixed campaigns saving successfully without a configured system access root URL, which left employees without a learning link after submission.
- Fixed re-running the installer on an instance that already publishes the marketing site taking the site offline.
- Fixed campaigns being paused for no reason by job status write conflicts, and an automatic retry path that never actually took effect.
- Fixed domains remaining occupied after deleting a tenant from the recycle bin.
- Fixed metered plans being blocked incorrectly by comparing plan-level recipient totals with the license per-campaign limit.

### Upgrade Notes

- **The rename is a breaking change**: old deployment identifiers, old learning links, old DNS verification records, and old backup file names are incompatible. Existing environments should be treated as fresh deployments.
- **Home page moved**: the product home moved from `/` to `/dashboard`. Without the marketing site, `/` redirects automatically and existing bookmarks keep working.
- **The marketing site is disabled by default**: do not add `--no-site` when upgrading an instance that already publishes it, or the site will be taken offline.
- **Bundled templates and courses are imported once**: later releases do not update them on installed instances, and content an administrator deleted is not restored.
- **Self-registration is open by default**: new instances enable it (`ALLOW_SELF_REGISTRATION=true`), and existing configuration is never overwritten.

---

## [v1.8.0] - 2026-10-21

### Added

- **Built-in free license**: initialization succeeds and the product is usable without importing a license certificate, running on built-in quotas (1 tenant, 5 users, 1 SMTP gateway, 50 recipients per campaign, 500 recipients in total, 1 concurrent campaign). Only custom templates are enabled under this license; AI, template import and cloning, backup and restore, report export, UI customization, multi-language, micro-courses, and self-registration stay disabled.
- An expired commercial license no longer stops the service; the system falls back to the built-in quotas and clearly states this at the top of the system settings page.
- **Micro-courses and self-registration are now license-gated**: without the entitlement, their entry points and features become unavailable, and new or edited campaigns offer only "custom external link" as a learning step while existing campaigns keep their configuration.
- **AI capabilities are now license-gated**: without the entitlement, the analytics "AI Usage" tab is disabled with an explanation.

### Changed

- License checks now require both an activated certificate and a true flag; inactive, expired, and clock-rollback states are consistently treated as unlicensed.
- The license page shows micro-course and self-registration status, and reports "built-in free license" as the source when no certificate is present.
- The repository no longer bundles a demo license file, and local development also runs on the built-in license.

### Fixed

- Added HTML sanitization to rich-text rendering (legal documents, micro-course editing and viewing, and the course portal), fixing script injection from externally imported course content. The allowlist blocks `iframe`, `script`, and `style`, and rejects `data:` inline resources.
- The free-trial entry on the login page no longer appears with a delay.

### Upgrade Notes

- Environments using micro-courses or self-registration must reissue a certificate containing the corresponding flags; a certificate without them counts as unlicensed. Self-registration additionally requires `ALLOW_SELF_REGISTRATION` to be true at the deployment level.
- Without a commercial certificate the system runs on built-in quotas. Existing data beyond those quotas is not deleted, but related create operations are rejected.
- Rich text that embedded content with `iframe` or inlined images with `data:` no longer renders after the upgrade; switch to external image links.

---

## [v1.7.0] - 2026-10-18

### Added

- **Fell-for-it → learn → quiz loop**: the learning link handed out after a simulated form submission carries a signed token, so the post-course quiz opens without an account or login. Scores and explanations appear immediately, and only the first submission counts, so retries cannot inflate results.
- Campaign reports gained four learning metrics (learned, not learned, participation rate, quiz all-correct count) and per-person learning details, with server-side paging and one-click Excel export.
- **Configurable learning step**: no learning, automatically recommended courses, manually selected courses, or a custom external link. Manual selection lists only published courses that have a quiz configured.
- **Campaign environment self-check**: one click concurrently checks plan quota, tenant trusted status, SMTP gateway, targets, and groups, reporting each item as ready or not with the next action. It is read-only and can be re-run safely.
- The target import template gained phone and job title columns, and "import into a specified company" assigns the whole batch at once. Bulk ownership changes require a company and treat department as optional.
- System settings gained a "Global Settings" page for the system access root URL, used for learning links, notification deep links, self-registration activation links, and template thumbnails.

### Changed

- The "Phone" column in user management became "Email", with a matching search option.
- The quiz question limit rose from 5 to 10, random selection defaults to 5, and manual selection allows up to 10.

### Fixed

- The SMTP gateway sender email no longer bypasses domain validation for the built-in sending service and now uses the same rules as third-party gateways.
- Installer database migrations now run idempotently from a single file, fixing a failure where inline comments were extracted as part of the password and migrations silently failed.
- Importing a CSV no longer creates a duplicated department level when the first segment matches the company name.
- When binding an authenticator app, the account shown prefers the login email over a random `trial_` username.
- Gateway test mails and self-registration activation mails are now HTML plus plain text, fixing garbled Chinese subjects on some mail servers.

### Upgrade Notes

- Environments that relied on the micro-course root URL falling back to the tracking domain must configure the address in Global Settings after upgrading, otherwise learning links and other long-lived addresses cannot be built. The absence of a fallback is intentional.

---

## [v1.6.0] - 2026-10-14

### Added

- **Free-trial self-registration**: the login page gained a free-trial entry. Visitors submit company and contact details plus a CAPTCHA, then set a password from the confirmation email to activate. Whether manual review is required is configurable by the super admin.
- "Verify first, create later": submission only records an application. Domain and account ownership are decided when the confirmation link is opened, so a domain cannot be squatted by an abandoned application.
- Super admins gained self-registration management: application list with status counts, details, approval, rejection, and resending the confirmation email.
- System settings gained a self-registration tab (open mode, rate limits, domain blocklist) and a technical support setting for the contact email shown after registration.
- Registration and activation resist enumeration and abuse, never revealing whether a domain or email is already registered.
- **Sender account restriction**: the gateway sender email must be a full address whose domain belongs to the tenant or to the domains allowed by the current plan.

### Changed

- New tenants and self-registration activations default to a maximum of 1 operator.
- The intranet address restriction moved from a permanent note to a tooltip beside the SMTP host field, with long tooltips capped at 360 px.

### Fixed

- Fixed agreement text on the login page being nearly unreadable when a light theme preference lingered.
- Fixed a tenant downgraded to trial keeping its previous plan, which made trial limits ineffective.
- Fixed "create as trial" tenants not receiving a service plan, which left them unable to create even a single target.
- Fixed the required-field asterisk on the SMTP settings page wrapping onto its own line.

---

## [v1.5.0] - 2026-10-13

### Added

- **Login email binding**: after binding, accounts can sign in with email plus password while still supporting usernames. Unbinding requires a two-step code confirmation, and codes are limited to a 60-second cooldown, 10 per day, invalidated after 5 wrong attempts, and single-use.
- Administrators can rebind or clear a user's login email directly from user management.
- A "Trial" service level (the lowest tier) was added, and tenant management supports adjusting service level and quota per tenant.
- **Tenant trust and domain verification**: campaigns of untrusted tenants are blocked at the trust gate when starting, and changing a tenant domain automatically voids existing verification.
- A lightweight tracking role `-role=tracking` allows the tracking receiver to be deployed separately from the management plane.
- Users must accept the privacy policy and service agreement before signing in. When an agreement is revised, re-acceptance is required and the version and time each user accepted are recorded.
- System settings gained a draft campaign limit (default 50, 0 means unlimited).

### Changed

- Notification emails use a plain layout matching the gateway test mail.
- Campaign end times now close at 23:59:59 on the same day.
- Idle scheduled tasks converge, substantially reducing database access during idle periods while active behavior stays unchanged.

### Fixed

- Fixed server logs being written into the system log file and growing it continuously.

---

## [v1.4.0] - 2026-10-11

### Added

- **Notification email channel**: system settings gained a notification panel for server, port, sender address, password, display name, language, and send rate. Ports 25 / 465 / 587 automatically select the corresponding encryption.
- The test button always sends to the configured sender and reports the failing stage plus the mail server's raw response on failure.
- Bilingual subjects and bodies for 8 notification types. Notification mail is fully isolated from campaign sending: it never enters campaign statistics and is never polluted by bounce callbacks.
- Delivery reliability: recoverable failures retry up to 2 times, recipients are re-validated before resending, and a channel-wide token bucket limits sending to 30 mails per minute by default.
- The message center supports deleting selected messages, affecting only your own copy.
- Users gained a notification email field, and the per-user email channel toggle became functional.

### Changed

- The message center moved from system settings to personal settings, and the system settings entry is hidden for operators.
- Four separate intranet switches were consolidated into one "Allow intranet access" switch (off by default), and intranet validation was added for tenant SMTP gateways. Built-in sending services are not governed by this switch.

### Fixed

- Notification subjects, bodies, and sender display names now consistently use the system name.
- Fixed the message list total being decremented twice, which left the last page empty.
- Removed remaining occurrences of the old product name across UI copy, comments, and local storage keys; a one-time cross-login prompt may reappear after upgrading.

### Upgrade Notes

- The old intranet switches are no longer read, so intranet access returns to off after upgrading. Super admins must re-enable it once from the UI if it was previously on.

---

## [v1.3.0] - 2026-10-10

### Added

- **Notification module**: an in-app message center with a bell and unread badge in the top bar, real-time over WebSocket with polling fallback, covering campaign approval, campaign status, and service plan expiry.
- Notification scopes are switch-controlled at both platform and user level with an "enabled when no record exists" semantic, so existing users are not silently muted by the upgrade.
- **Analytics module**: AI usage (multi-dimensional aggregation plus call details) and target analysis (repeat victims, risk levels, overall security score).
- Super admins can filter campaigns by tenant, with stat cards and the list following the selection.
- The course list gained a "learn count" column, the top bar shows the current role, and landing pages are mobile-responsive.

### Changed

- Campaign status uses a single seven-state vocabulary shared by the list and the stat cards (draft, sending, sent, awaiting release, paused, finished, terminated).
- The legacy tenant-level expiry field was retired; tenant availability is governed solely by the service plan period.

### Fixed

- Fixed "sending" statistics dropping rows, and the report banner not showing the end time and remaining duration.
- Fixed a request storm when opening the member drawer: a 5,000-member group went from about 50 requests to 1.
- Fixed unreadable message details in dark mode, non-responsive notification deep links, and a bulk mark-as-read action that could mark everything read with empty parameters.
- Upgrades no longer overwrite locally edited configuration and certificates (only `*.example` templates are provided), and `install.sh` adds the WebSocket block to existing Nginx configuration automatically.

---

## [v1.2.0] - 2026-10-07

### Added

- **Micro-course quizzes and question bank**: quizzes can be assembled manually or by random tag-based selection, with a preview and a diagnosis of whether enough questions are available.
- The question bank supports single choice, multiple choice, and true/false, bilingual prompts and explanations, difficulty tags, enable/disable, and answer previews. Questions can be assigned to a course as exclusive questions.
- AI question generation from course content: set the question-type mix and target points, review the candidates, and import them in one step.
- **AI model scheduling**: one API base URL can hold multiple candidate models that are tried in order, with larger sort values taking priority.
- The recycle bin gained question bank and template center entries, and "System Users" merged into "User Management".

### Fixed

- Security: blocked arbitrary file writes when importing course backup packages and added a state guard to permanent deletion.
- Fixed AI question generation returning only a single tag.
- The media library now uses "load more" and the wide-screen gap is fixed; the portal header search box, breadcrumbs, and footer adapt to narrow screens.

---

## [v1.1.0] - 2026-10-06

### Added

- **Micro-courses module**: an admin console plus a login-free course portal under `/training`, with inline image uploads and a media library, AI course generation, and categories and tags including English names.
- An operator campaign approval mechanism, with approval status as its own column in the campaign list.
- AI-recommended micro-courses for campaigns, with learning links optionally following the recommendation. The micro-course root URL is configurable independently of the tracking root URL.
- Tenant-level AI model configuration, with AI runtime logs written to disk and tagged by capability.
- Target synchronization gained Coremail APIWS V3 support.
- A bundled course generation tool, `phishgenix-batch-courses`, ships with the product.

### Fixed

- Fixed the micro-course page shaking when switching to an empty category, orphaned tag relations after deletion, and garbled Chinese file names in the media library.
- Fixed the login page default title still showing Chinese in English environments.

---

## [v1.0.1] - 2026-09-05

### Added

- The report editor supports WYSIWYG pagination (the editing view is the print view), with outline drag-and-drop across chapters, undo/redo, version history, and concurrent-edit conflict detection.
- AI report generation returns structured content and is launched from the editor toolbar.
- Real-time mail queue monitoring with a top-bar indicator and a monitoring drawer.
- Per-gateway send rate limits for SMTP gateways, with the global default moved into campaign configuration.
- Landing page submission responses support a "success message only" mode, with all post-submission content customizable.
- Dashboard stat cards were split into eight mutually exclusive metrics separating campaign lifecycle from delivery progress.

### Fixed

- Fixed orphaned jobs after a service restart stalling campaign delivery, and hardened the queue against backlogs.
- Fixed the monitor showing a red "severe backlog" state while the queue was empty.
- Fixed duplicate landing page submissions overwriting the first submission; the first is now kept and the handling state recorded.
- Fixed automatic pagination and text box width in PDF report export.
- Also: operator visibility now includes licensed dynamic group members, plus assorted editor and dark-mode styling fixes.

---

## [v1.0.0] - 2026-08-18

First general availability release. Highlights:

- Campaigns: the full email simulation workflow (creation, SMTP gateway delivery, progress control, interruption recovery)
- Behavior tracking: opens, link clicks, landing page submissions, QR scans, and bounce callbacks
- Email templates and landing pages: visual editors, customizable submission responses, batch template generation
- Recipient management: departments and groups, bulk import
- Campaign reports and export
- Multi-tenant isolation, role-based permissions, and audit logs
- RSA-signed license mechanism
- Help center (Chinese and English)
- One-click deployment (install.sh + Docker Compose + Nginx) and a backend operations CLI
