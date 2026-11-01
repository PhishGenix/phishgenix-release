# PhishGenix Releases

### AI-Powered Phishing Simulation & Security Awareness Platform

**PhishGenix 官方软件发布与部署仓库**

[官方网站](https://www.phishgenix.cn/) · [GitHub Releases](https://github.com/PhishGenix/phishgenix-release/releases) · [项目主页](https://github.com/PhishGenix)

---

# 🇨🇳 中文说明

## 项目简介

**PhishGenix** 是一款 AI 驱动的企业安全意识平台，集钓鱼邮件演练、AI 智能分析、安全数据统计和安全意识培训于一体，帮助组织识别人员安全风险，提升员工网络安全意识和威胁防范能力。

本仓库是 PhishGenix 官方软件发布仓库，提供编译后的软件部署包、版本更新信息及部署相关文档。

## 核心功能

- **钓鱼邮件演练：** 创建和管理钓鱼邮件演练活动，模拟真实攻击场景。

- **AI 智能分析：** 智能分析演练数据，识别潜在人员安全风险。

- **数据统计与报告：** 可视化展示演练结果，生成安全分析报告。

- **安全意识培训：** 提供体系化安全培训课程与学习管理。

- **多租户管理：** 支持多组织、多租户独立管理。

## 软件获取

所有正式版本均通过 GitHub Releases 发布。

[**进入下载页面**](https://github.com/PhishGenix/phishgenix-release/releases)

| 文件类型           | 说明                          |
| -------------- | --------------------------- |
| Linux AMD64    | 适用于 x86_64 架构服务器的部署包        |
| Linux ARM64    | 适用于 ARM64 架构服务器的部署包（如该版本提供） |
| SHA256SUMS.txt | 安装包完整性校验文件                  |

请根据服务器 CPU 架构选择对应的安装包。

## 部署要求

建议部署环境：

| 项目     | 要求                     |
| ------ | ---------------------- |
| 操作系统   | Linux                  |
| CPU 架构 | AMD64 / ARM64（以具体版本为准） |
| 容器环境   | Docker Engine          |
| 编排工具   | Docker Compose Plugin  |
| CPU    | 建议 4 核及以上              |
| 内存     | 建议 8 GB 及以上            |
| 网络     | 能够访问所需 Docker 镜像仓库     |

具体资源要求以实际部署规模为准。

## 快速安装

- 下载软件包

进入 [GitHub Releases](https://github.com/PhishGenix/phishgenix-release/releases)，下载对应版本的部署包。

详细步骤请参考安装指南。

## 文档中心

- [安装部署指南](docs/installation.md)

- [数据备份与恢复](docs/backup-restore.md)

## 版本更新

请关注 [CHANGELOG.md](CHANGELOG.md) 或 GitHub Releases 获取版本更新信息。

建议正式环境使用稳定版本，并在升级前做好数据库和业务数据备份。

## 问题反馈

如果在安装、部署或使用过程中遇到问题，可以通过以下渠道获取帮助：

- [GitHub Issues](https://github.com/PhishGenix/phishgenix-release/issues)

- [官方网站](https://www.phishgenix.cn/)

提交问题时，建议提供软件版本、操作系统版本、部署环境及相关错误日志。请勿上传密码、密钥或其他敏感信息。

## 免责声明

本软件用于企业内部安全意识培训、授权的钓鱼邮件安全演练及网络安全教育。

使用者应确保所有演练活动均获得相关组织授权，并遵守适用的法律法规。不得将本软件用于未经授权的攻击、欺诈或其他非法活动。

---

# 🇬🇧 English Documentation

## About PhishGenix

**PhishGenix** is an AI-powered security awareness platform that combines phishing simulation, intelligent analytics, security reporting, and structured security awareness training.

It helps organizations identify human security risks, evaluate employee awareness, and strengthen their ability to recognize and respond to cyber threats.

This repository is the official software distribution channel for PhishGenix, providing compiled deployment packages, release information, and deployment documentation.

## Key Features

- **Phishing Simulation:** Create and manage realistic phishing campaigns.

- **AI-Powered Analysis:** Analyze simulation results and identify potential human security risks.

- **Analytics & Reporting:** Visualize campaign results and generate security reports.

- **Security Awareness Training:** Deliver structured security education and training.

- **Multi-Tenant Management:** Support independent management of multiple organizations and tenants.

## Downloads

All official releases are published through GitHub Releases.

[**Download PhishGenix**](https://github.com/PhishGenix/phishgenix-release/releases)

| Package        | Description                                          |
| -------------- | ---------------------------------------------------- |
| Linux AMD64    | Deployment package for x86_64 servers                |
| Linux ARM64    | Deployment package for ARM64 servers, when available |
| SHA256SUMS.txt | Package integrity verification                       |

Please select the package that matches your server architecture.

## System Requirements

Recommended deployment environment:

| Component         | Requirement                             |
| ----------------- | --------------------------------------- |
| Operating System  | Linux                                   |
| CPU Architecture  | AMD64 / ARM64, depending on release     |
| Container Runtime | Docker Engine                           |
| Orchestration     | Docker Compose Plugin                   |
| CPU               | 4 cores or more recommended             |
| Memory            | 8 GB or more recommended                |
| Network           | Access to required container registries |

Actual resource requirements depend on deployment scale.

## Quick Installation

### 1. Download

Visit [GitHub Releases](https://github.com/PhishGenix/phishgenix-release/releases) and download the appropriate deployment package.

### 2. Extract

```
tar -zxvf PhishGenix-v1.0.0-linux-amd64.tar.gzcd phishgenix
```

### 3. Configure

Create and configure the `.env` file based on `.env.example`.

Configure database settings, service ports, and other required parameters.

### 4. Install

```
chmod +x install.sh./install.sh
```

For detailed instructions, please refer to the installation guide.

## Documentation

- [Installation Guide](docs/installation.md)

- [Backup & Restore](docs/backup-restore.md)

## Release Notes

Refer to [CHANGELOG.md](CHANGELOG.md) or [GitHub Releases](https://github.com/PhishGenix/phishgenix-release/releases) for version updates and release information.

For production environments, we recommend using stable releases and backing up databases and business data before upgrading.

## Support & Feedback

If you encounter installation, deployment, or usage issues:

- [GitHub Issues](https://github.com/PhishGenix/phishgenix-release/issues)

- [Official Website](https://www.phishgenix.cn/)

When reporting an issue, include the software version, operating system, deployment environment, and relevant error logs. Do not share passwords, credentials, or other sensitive information.

## Disclaimer

PhishGenix is intended for authorized internal security awareness training, phishing simulation exercises, and cybersecurity education.

Users are responsible for ensuring that all simulation activities are properly authorized and comply with applicable laws and regulations. The software must not be used for unauthorized attacks, fraud, or other unlawful activities.

---

**PhishGenix — Turn Security Awareness into Security Resilience.**
