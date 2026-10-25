# Changelog

本文件记录 GoodJob 外贸 CRM 维护分支的重要变更，格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循
[语义化版本](https://semver.org/lang/zh-CN/)（MAJOR.MINOR.PATCH）。

## [Unreleased]

### Added

- 全部容器日志轮转：json-file 单文件 50m、保留 3 份，防止磁盘被容器日志写满。

## [0.1.0] - 2026-10-25

首次上线（fork 自 [huangguojia/GoodJob](https://github.com/huangguojia/GoodJob)）。

### Added

- 生产部署：Docker Compose 四容器（mysql / backend / communication / gateway），
  1Panel（OpenResty）反向代理 + Cloudflare HTTPS（`https://crm.ysec.cn`）。
- 本地镜像构建模式（`GOODJOB_IMAGE_MODE=local`），不依赖上游 ACR 镜像。
- 部署运维手册《GoodJobCRM部署运维手册.md》，含上线前检查、备份恢复与故障排查。
- GitHub 部署流：本地同步上游 → 审核合并 → 调试 → 推送 GitHub → 服务器拉取重建。

### Changed

- AI 助手与界面署名由 Kevin 改为 Yves（后端 `ai-agent.ts`、登录页、agent-skills、测试断言）。
- 去除 README 中作者打赏信息，补充本维护分支与生产部署说明。
- communication 环境新增 `CRM_BASE_URL=http://backend:4188`，修复 CRM 待办与站内信通道
  （`crm_adapter` 商业上线检查项 fetch failed 问题）。

### Fixed

- Docker Compose 卷定义错误（`postgres_data` 卷 name 错位导致
  `mapping key "name" already defined`）。
- 生产登录凭据初始化：`INITIAL_ADMIN` 经 configure.sh 管道未写入的问题；
  以幂等播种账号（`superadmin@goodjob.com`）为基础，新增业务管理员账号。
- scrypt 密码哈希在 bash 变量展开中被 `$` 截断导致登录失败；改用 Node 脚本
  直接更新数据库哈希修复。

### Security

- 生产强制 WhatsApp 官方通道（`WHATSAPP_OFFICIAL_ONLY=true`）。
- 会话 Cookie 强制 Secure（`SESSION_COOKIE_SECURE=true`）、`PUBLIC_SCHEME=https`。
- 生产密钥（`JWT_SECRET`、加密密钥、数据库口令等）统一存放于
  `/opt/goodjobcrm/shared/secrets/`，不经由 GitHub 流转。
