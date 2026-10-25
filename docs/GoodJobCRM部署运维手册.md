# GoodJob 外贸 CRM 部署运维手册

> 日期：2026-10-25
> 适用版本：v0.1.0（维护分支，fork 自 huangguojia/GoodJob）
> 生产地址：https://crm.ysec.cn
>
> 本文档面向本系统维护者，不含任何生产密钥明文；所有敏感值均存放于服务器
> `/opt/goodjobcrm/shared/secrets/`，变更与查询一律在服务器上进行。

---

## 1. 架构总览

```
用户浏览器
   │  https://crm.ysec.cn
   ▼
Cloudflare（橙云，Full strict，Origin CA 证书）
   ▼
1Panel OpenResty（反向代理，监听 443）
   ▼
gateway 容器（127.0.0.1:4188 → 8080，统一入口）
   ├── backend 容器（4188，CRM 业务 API）
   └── communication 容器（3100，WhatsApp 插件 / 商业上线检查）
              └── mysql 容器（3306，MySQL 8.4）
```

- 容器编排：`deploy/docker-compose/compose.yml`（Compose 项目名 `goodjobcrm`）
- 全部容器运行于内网桥 `goodjobcrm_private`，仅 gateway 通过
  `127.0.0.1:4188` 暴露到宿主机，公网只经 OpenResty 访问
- 数据卷：`goodjobcrm_mysql_data`（数据库）、`goodjobcrm_communication_media`
  （WhatsApp 媒体）、`goodjobcrm_uploads_data`（后端上传）

## 2. 服务器目录与凭据

| 路径 | 用途 |
|---|---|
| `/opt/goodjobcrm-source` | 源码（git 仓库，与 GitHub 同步） |
| `/opt/goodjobcrm/shared/deploy.env` | Compose 环境变量（`PUBLIC_ORIGIN`、`PUBLIC_SCHEME`、`GOODJOB_IMAGE_MODE=local` 等） |
| `/opt/goodjobcrm/shared/secrets/` | 各类密钥与口令（初始管理员密码、`JWT_SECRET`、数据库口令、`crm_runtime_env` 等） |

> 凭据管理纪律：
> - 生产密钥只存在于服务器 secrets 目录，**禁止**写入 GitHub、README 或本手册；
> - 登录口令等个人凭据的明文保存位置由维护者自行管理（如 1Panel 或本地密码本），
>   服务器侧只存服务运行所需的最小口令集。

## 3. 发布流程（GitHub 部署流）

所有生产变更一律走以下链路，**禁止本地直连服务器传输**：

```
本地开发机                 GitHub 仓库              服务器
──────────                ──────────              ──────
拉取上游同步  ──merge──>  审核合并
审核 diff 合并
本地调试验证
上传 GitHub ─────────────> push origin master
                          pull ───────────────> 拉取远程源码
                                                docker compose up -d --build 重建
```

### 3.1 本地侧（开发机）

```powershell
# 1. 同步上游并合并（审核 diff）
git fetch upstream
git diff master upstream/master          # 审核差异
git merge upstream/master                # 合并（有冲突时逐个解决）

# 2. 本地调试验证
npm install
npm run build
npm run test

# 3. 推送 GitHub（本机经代理 127.0.0.1:7890）
git push origin master
```

### 3.2 服务器侧（拉取与重建）

```bash
cd /opt/goodjobcrm-source
sudo git pull origin master
# 若上游源码覆盖了部署必需定制（compose.yml 卷挂载等），需重新恢复：
#   backend 卷需含 backend_private_data:/app/backend/.data，否则后端数据目录丢失
sudo GOODJOB_RELEASE_ID=$(date +manual-%Y%m%d%H%M%S) docker compose \
  --project-name goodjobcrm --env-file /opt/goodjobcrm/shared/deploy.env \
  -f deploy/docker-compose/compose.yml up -d --build
```

- 每次升级会以当前时间生成新的 `GOODJOB_RELEASE_ID`（镜像 tag），旧镜像保留
  可回滚：`docker compose up -d --force-recreate` 并指回旧 tag。
- 升级前建议先 `docker compose config --quiet` 校验 YAML。

## 4. 上线前检查（商业上线检查）

进入 CRM「WhatsApp / 商业上线」页，或调用插件接口核对 10 项：

```bash
# 服务器上执行（token 通过 /api/auth/login 获取）
curl -s http://127.0.0.1:3100/api/v1/commercial-readiness \
  -H "Authorization: Bearer <JWT>"
```

检查项与当前状态（2026-10-25）：

| 检查项 | 状态 | 说明 |
|---|---|---|
| crm_auth | pass | 会话身份验证 |
| crm_adapter | pass | CRM 待办与站内信通道（依赖 `CRM_BASE_URL`） |
| official_strategy | pass | 新账号默认 Meta 官方通道 |
| meta_app | blocking | 需配置 Meta App / App Secret / Verify Token |
| meta_account | blocking | 需绑定 WABA / Phone Number ID / Access Token |
| meta_connection | blocking | 需完成 Graph API 验证的官方账号 |
| public_https | pass | 公网 HTTPS 回调可达 |
| webhook | blocking | 需收到真实 Meta Webhook 测试消息 |
| ai_intelligence | warning | 当前规则分析，商用前建议配置 AI Provider |
| automation | pass | 周期分析与每日通知启用 |
| production_database | pass | 连接生产 MySQL |
| retention | pass | 附件发送后立即清理 |

## 5. 配置项清单

### 5.1 deploy.env（服务器）

| 变量 | 当前值 | 说明 |
|---|---|---|
| `GOODJOB_IMAGE_MODE` | `local` | 从源码构建本地镜像，不使用上游 ACR |
| `PUBLIC_SCHEME` | `https` | 公网协议 |
| `PUBLIC_ORIGIN` | `https://crm.ysec.cn` | 站点源（CORS / 回调基址） |
| `SESSION_COOKIE_SECURE` | `true` | 会话 Cookie 仅 HTTPS |
| `DOMAIN` | `crm.ysec.cn` | 邮件 Message-ID 域名 |
| `TRACKING_PROVIDER` | `17track` | 物流跟踪服务商 |

### 5.2 关键环境变量（compose.yml / secrets）

- `CRM_BASE_URL=http://backend:4188`：communication 访问 CRM 的容器内地址，
  缺失会导致 `crm_adapter` 检查 fetch failed。
- `WHATSAPP_OFFICIAL_ONLY=true`：生产强制 Meta 官方通道，禁用非官方协议。
- `JWT_SECRET`、`PROVIDER_CREDENTIAL_KEY`、`TRADE_OBSERVATION_CURSOR_SECRET`、
  `MARKET_OPPORTUNITY_CURSOR_SECRET`、`ORGANIZATION_IDENTITY_MASTER_SECRET`、
  `PROSPECT_SOURCE_RAW_ENVELOPE_SECRET`：各密钥之间互不复用，由部署脚本生成，
  升级部署沿用旧值（避免历史密文失效）。

## 6. 账号管理

| 账号 | 邮箱 | 角色 | 说明 |
|---|---|---|---|
| u_superadmin | superadmin@goodjob.com | 内置超级管理员 | 系统幂等播种账号，勿改邮箱 |
| u_yves | zytzenger@163.com | 管理员 | 业务管理员 |

- 内置账号由后端 `ensureBuiltInSuperAdmin` 按邮箱幂等播种；**不要修改
  u_superadmin 的邮箱**，否则播种逻辑找不到原账号会插入冲突导致后端启动失败。
- 新增/重置密码：使用 Node 脚本直接写 MySQL（scrypt 哈希），注意 `$` 符会
  被 bash 变量展开吞掉，脚本文件方式运行，改后必须立即自验登录。

## 7. 备份与恢复

```bash
# 数据库备份（容器内执行，输出到宿主机）
sudo docker exec goodjobcrm-mysql-1 sh -c \
  'MYSQL_PWD=$(cat /run/secrets/mysql_root_password) mysqldump --databases goodjob_crm_prod' \
  > /opt/goodjobcrm/backups/crm-$(date +%Y%m%d-%H%M).sql

# 数据卷（上传文件、媒体）备份
sudo tar -czf /opt/goodjobcrm/backups/volumes-$(date +%Y%m%d).tgz \
  -C /var/lib/docker/volumes goodjobcrm_uploads_data goodjobcrm_communication_media
```

- 备份保留策略由维护者按 1Panel 计划任务落地（每日）。
- 恢复：先停 backend/communication，再 `mysql < 备份.sql`，解包卷后重建容器。

## 8. 常见故障排查

| 症状 | 原因与处理 |
|---|---|
| 登录提示账号密码错误 | scrypt 哈希含 `$` 被 bash 展开破坏；用 Node 脚本（非内联 echo）重写哈希并自验 |
| `crm_adapter` 检查 fetch failed | communication 缺 `CRM_BASE_URL`；compose.yml 已含 `http://backend:4188`，重建该容器 |
| CORS origin 拒绝 | `PUBLIC_ORIGIN` 与访问协议不一致；确保 `PUBLIC_SCHEME=https` + `SESSION_COOKIE_SECURE=true` |
| compose 报 `mapping key "name" already defined` | 卷定义语法错位；按 mysql_data → postgres_data → backend_private_data → communication_media → uploads_data 顺序修正 |
| 后端数据目录丢失 | backend 卷缺少 `backend_private_data:/app/backend/.data` 挂载；升级 pull 后需恢复该定制 |
| `npm ci` 在 Linux 构建失败 | 本机 package-lock 缺 Linux 平台依赖；Dockerfile 已用 `npm install` 规避 |

## 9. 文档与版本维护

- 每次变更：更新 `CHANGELOG.md`（Keep a Changelog + SemVer），并在本次部署说明中记录。
- `README.md` 保持与当前版本事实一致；部署相关细节一律以本手册为准。
- 本手册为活文档，随部署演进更新，文首日期同步刷新；文件名不带版本号。
