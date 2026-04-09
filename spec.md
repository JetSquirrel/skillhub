# SkillHub 技术规格说明书

## 项目概述

**SkillHub** 是一个企业级开源智能体技能注册中心，为组织提供私有化部署的技能包管理平台。它支持技能的发布、发现、版本管理和治理，专为本地部署和防火墙后运行而设计。

### 核心定位

- **单实例共享注册中心**：不是多租户平台，而是单一共享实例
- **命名空间隔离**：以命名空间而非租户作为隔离边界
- **全局 + 团队模式**：`@global` 平台公共空间 + `@team-*` 协作空间
- **公开可访问**：公共技能（visibility=PUBLIC）支持匿名浏览和下载

### 产品蓝本

- **继承 ClawHub** 的产品模型：技能注册中心的整体边界、版本管理、治理机制
- **借鉴 OpenSkills** 的格式约定：SKILL.md 格式、目录结构
- **兼容层支持**：提供 ClawHub CLI 协议兼容层，使现有 ClawHub CLI 无需修改即可使用

---

## 技术架构

### 技术栈

#### 后端技术基线

- **语言**：Java 21
- **框架**：Spring Boot 3.2.3
- **安全**：Spring Security + OAuth2 Client
- **数据库**：PostgreSQL 16.x + Flyway 迁移
- **缓存/会话**：Redis 7.x（Session 存储 + 分布式锁 + 幂等去重）
- **对象存储**：LocalFile（开发环境）+ S3 兼容接口（生产环境）
- **搜索**：PostgreSQL 全文搜索（一期）
- **API 文档**：Springdoc OpenAPI

#### 前端技术栈

- **语言**：TypeScript
- **框架**：React 19
- **构建工具**：Vite
- **路由**：TanStack Router
- **数据获取**：TanStack Query
- **样式**：Tailwind CSS + Radix UI（shadcn/ui）
- **API 客户端**：OpenAPI TypeScript（类型安全）
- **国际化**：i18next

#### 基础设施

- **容器化**：Docker & Docker Compose
- **监控**：Prometheus + Grafana
- **部署**：Kubernetes 清单（基础版）
- **CI/CD**：GitHub Actions
- **镜像发布**：GHCR（多架构支持：linux/amd64 + linux/arm64）

### 架构设计

#### 整体架构

采用**单体优先、模块化单体**设计。业务域清晰，一期规模不需要微服务拆分。

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│   Web UI    │     │  CLI Tools  │     │  REST API    │
│  (React 19) │     │             │     │              │
└──────┬──────┘     └──────┬──────┘     └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Nginx     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Spring Boot │  Auth · RBAC · Core Services
                    │   (Java 21) │  OAuth2 · API Tokens · Audit
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼───┐  ┌─────▼────┐  ┌────▼────┐
       │PostgreSQL│  │  Redis   │  │ Storage │
       │    16    │  │    7     │  │ S3/MinIO│
       └──────────┘  └──────────┘  └─────────┘
```

#### 后端模块结构

```
server/
├── skillhub-app                 # 启动、配置装配、Controller 聚合
├── skillhub-domain              # 领域模型 + 领域服务 + 应用服务
├── skillhub-auth                # OAuth2 认证 + RBAC + 授权判定
├── skillhub-search              # 搜索 SPI + PostgreSQL 全文实现
├── skillhub-storage             # 对象存储抽象 + LocalFile/S3 双实现
└── skillhub-infra               # JPA、通用工具、配置基础
```

#### 模块依赖原则

遵循**依赖倒置原则**，禁止领域层依赖基础设施：

```
app → domain, auth, search, storage, infra
infra → domain          # infra 实现 domain 定义的 Repository 接口
auth → domain           # auth 引用 UserAccount 等领域实体
search → domain         # search 引用 SkillSearchDocument 等领域模型
storage → (独立抽象)     # 纯 SPI，不依赖 domain
```

**核心原则**：
- `domain` 是最内层，不依赖任何其他模块，只定义接口和实体
- `infra` 实现 `domain` 中定义的 Repository 接口（Spring Data JPA）
- `app` 负责装配所有模块，通过 Spring 依赖注入将 `infra` 实现注入 `domain` 接口

#### 前端工程结构

```
web/
├── src/
│   ├── app/              # 路由、全局 Provider、布局
│   ├── pages/            # 页面入口
│   ├── features/         # 搜索、上传、版本管理、审核等业务功能
│   ├── entities/         # skill、user、namespace 等领域展示逻辑
│   ├── shared/           # 通用组件、hooks、工具
│   └── api/              # openapi-typescript 生成的类型 + openapi-fetch 客户端
├── package.json
└── vite.config.ts
```

---

## 核心领域模型

### 用户标识约束

**全链路统一使用 `string` 作为用户主键**，覆盖所有用户关联字段：
- `user_id`、`owner_id`、`created_by`、`updated_by`、`published_by`
- `reviewed_by`、`actor_user_id` 及所有等价语义字段

**原因**：兼容外部 SSO / OAuth / OIDC / SCIM 身份源，外部 UID 通常是稳定字符串。

### 技能坐标体系

#### 内部坐标（Namespace 模型）

```
@{namespace_slug}/{skill_slug}
```

- **全局空间**：`@global/my-skill`
- **团队空间**：`@team-name/my-skill`

#### 兼容层坐标（ClawHub CLI 兼容）

| SkillHub 坐标 | 兼容层 Canonical Slug | 说明 |
|---|---|---|
| `@global/my-skill` | `my-skill` | 全局空间省略前缀 |
| `@team-name/my-skill` | `team-name--my-skill` | 团队空间使用 `--` 分隔 |

**约束规则**：
- 分隔符为双连字符 `--`
- slug 格式：`[a-z0-9]([a-z0-9-]*[a-z0-9])?`，长度 2-64
- 禁止包含连续两个以上的连字符 `--`
- 全局空间的 skill slug 禁止包含 `--` 以避免歧义

**保留词列表**（用户不可使用）：
`admin`, `api`, `dashboard`, `search`, `auth`, `me`, `global`, `system`, `static`, `assets`, `health`

### 核心实体

#### Namespace（命名空间）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| slug | varchar(64) | URL 友好标识（唯一） |
| display_name | varchar(128) | 展示名 |
| type | enum | `GLOBAL` / `TEAM` |
| description | text | 描述 |
| avatar_url | varchar(512) | 头像 |
| status | enum | `ACTIVE` / `FROZEN` / `ARCHIVED` |
| created_by | varchar(128) | 创建人 |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

**状态语义**：
- `ACTIVE`：正常使用
- `FROZEN`：冻结，只读不可发布新版本
- `ARCHIVED`：归档，对外不可见

#### Namespace Member（命名空间成员）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| namespace_id | bigint | 命名空间 ID |
| user_id | varchar(128) | 用户 ID |
| role | enum | `OWNER` / `ADMIN` / `MEMBER` |

**角色说明**：
- `OWNER`：命名空间创建者，可转让
- `ADMIN`：可审核该空间内的技能发布、管理成员
- `MEMBER`：可在该空间内发布技能（提交审核）

#### Skill（技能）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| namespace_id | bigint | 所属命名空间 |
| slug | varchar(128) | URL 友好标识（命名空间内唯一） |
| display_name | varchar(256) | 展示名 |
| summary | varchar(512) | 摘要 |
| owner_id | varchar(128) | 主要维护人（可转让） |
| source_skill_id | bigint | 派生来源（团队技能提升到全局时记录原 skill ID） |
| visibility | enum | `PUBLIC` / `NAMESPACE_ONLY` / `PRIVATE` |
| status | enum | `ACTIVE` / `ARCHIVED` |
| latest_version_id | bigint | 最新已发布版本指针 |
| download_count | bigint | 下载次数 |
| star_count | int | 收藏数 |
| rating_avg | decimal(3,2) | 平均评分 |
| rating_count | int | 评分人数 |
| hidden | boolean | 是否隐藏（治理覆盖层） |
| hidden_at | datetime | 隐藏时间 |
| hidden_by | varchar(128) | 隐藏操作人 |

**可见性规则**：
- `hidden=true`：仅 skill owner 或该 namespace 的 `ADMIN` / `OWNER` 可读
- `latest_version_id is null`：仅 skill owner 可读，不对外公开
- `PUBLIC`：任意人可读 skill 容器与已发布版本
- `NAMESPACE_ONLY`：该 namespace 任意成员可读
- `PRIVATE`：仅 skill owner 或该 namespace 的 `ADMIN` / `OWNER` 可读

#### Skill Version（技能版本）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| skill_id | bigint | 技能 ID |
| version | varchar(32) | 语义化版本号（唯一） |
| version_sort | bigint | 排序用数值 |
| changelog | text | 变更日志 |
| manifest_json | json | 文件清单 |
| parsed_metadata_json | json | SKILL.md frontmatter 解析结果 |
| status | enum | `DRAFT` / `PENDING_REVIEW` / `PUBLISHED` / `REJECTED` / `YANKED` |
| reject_reason | varchar(512) | 拒绝原因 |
| published_by | varchar(128) | 发布人 |
| published_at | datetime | 发布时间 |

**状态迁移约束**：
- 普通用户首次上传：`→ PENDING_REVIEW`
- `SUPER_ADMIN` 直发：`→ PUBLISHED`
- 审核通过：`PENDING_REVIEW → PUBLISHED`
- 审核拒绝：`PENDING_REVIEW → REJECTED`
- 撤回审核：`PENDING_REVIEW → DRAFT`
- 已发布撤回：`PUBLISHED → YANKED`

**版本号不可变性**：
- `DRAFT` / `REJECTED`：可删除版本记录，重新使用同版本号
- `PUBLISHED` / `YANKED`：版本号永久占用，不可复用

#### Review Task（审核任务）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| skill_version_id | bigint | 关联的版本 |
| namespace_id | bigint | 所属空间（决定谁能审核） |
| status | enum | `PENDING` / `APPROVED` / `REJECTED` |
| version | int | 乐观锁版本号 |
| submitted_by | varchar(128) | 提交人 |
| reviewed_by | varchar(128) | 审核人 |
| review_comment | text | 审核意见 |
| submitted_at | datetime | 提交时间 |
| reviewed_at | datetime | 审核时间 |

**并发约束**：同一 `skill_version_id` 在 `status=PENDING` 时只能存在一条记录。

#### Promotion Request（提升申请）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| source_skill_id | bigint | 来源团队 skill |
| source_version_id | bigint | 申请提升的版本 |
| target_namespace_id | bigint | 目标全局 namespace |
| target_skill_id | bigint | 审批通过后生成的全局 skill ID |
| status | enum | `PENDING` / `APPROVED` / `REJECTED` |
| version | int | 乐观锁版本号 |
| submitted_by | varchar(128) | 提交人 |
| reviewed_by | varchar(128) | 审核人 |
| review_comment | text | 审核意见 |

**说明**：团队技能提升到全局空间的唯一事实来源。

---

## 认证与授权

### 认证方式

#### 1. OAuth2 标准登录

- **一期支持**：GitHub OAuth
- **架构扩展性**：支持多 Provider（预留）
- **实现**：Spring Security OAuth2 Client

#### 2. CLI 认证（OAuth Device Flow）

- **流程**：CLI 请求设备码 → 用户在 Web 授权 → CLI 轮询获取凭证
- **凭证签发**：授权完成后签发 CLI 可用的 API Token

#### 3. API Token

- **用途**：自动化、兼容层和后续扩展
- **管理**：用户自行生成、命名、吊销
- **存储**：前缀 + 哈希存储，保证安全性
- **作用域**：支持 scope_json 字段（预留）

### RBAC 权限体系

#### 平台角色（系统内置）

| 角色 Code | 说明 | 典型权限 |
|---|---|---|
| `SUPER_ADMIN` | 平台超管，拥有所有权限 | 全部 |
| `SKILL_ADMIN` | 技能治理：全局空间审核、提升审核、隐藏/恢复、撤回已发布版本 | `review:approve`, `skill:manage`, `promotion:approve` |
| `USER_ADMIN` | 用户治理：准入审批、封禁/解封、角色分配（不可分配 SUPER_ADMIN） | `user:manage`, `user:approve` |
| `AUDITOR` | 审计只读：查看审计日志 | `audit:read` |

#### 命名空间角色

- `OWNER`：命名空间创建者，可转让
- `ADMIN`：可审核该空间内的技能发布、管理成员
- `MEMBER`：可在该空间内发布技能（提交审核）

**说明**：命名空间权限不走 RBAC 表，由 `namespace_member.role` 决定。

### 审核流程

#### 分级审核机制

- **团队空间**：由团队管理员（`ADMIN` / `OWNER`）审核
- **全局空间**：由平台管理员（`SKILL_ADMIN` / `SUPER_ADMIN`）审核
- **提升申请**：团队技能提升到全局需平台管理员二次审核

#### 当前审核模型

- **普通用户发布**：提交后进入 `PENDING_REVIEW`，审核通过后上线
- **`SUPER_ADMIN` 发布**：可直达 `PUBLISHED`
- **预留扩展点**：`PrePublishValidator`（当前为 NoOp，预留自动审核能力）

---

## 核心功能

### 技能发布

#### 发布流程

1. **上传技能包**：包含 `SKILL.md` 及相关文件
2. **元数据抽取**：解析 `SKILL.md` frontmatter
3. **文件校验**：扩展名白名单、文件大小限制
4. **提交审核**：进入 `PENDING_REVIEW` 状态
5. **审核通过**：状态变更为 `PUBLISHED`，更新 `latest_version_id`

#### SKILL.md 格式

- **主入口文件**：固定为 `SKILL.md`
- **格式**：frontmatter（YAML） + markdown body
- **必填字段**：
  - `name`：技能 slug
  - `description`：技能描述
  - `version`：语义化版本号

#### 文件存储

- **对象存储路径**：
  - 正式路径：`skills/{skillId}/{versionId}/{filePath}`
  - 打包路径：`packages/{skillId}/{versionId}/bundle.zip`
- **存储实现**：
  - 开发环境：LocalFile
  - 生产环境：S3 / MinIO

### 版本管理

#### 语义化版本（Semver）

- 格式：`MAJOR.MINOR.PATCH`（如 `1.2.3`）
- 排序：通过 `version_sort` 字段数值化排序

#### 标签管理

- **`latest` 标签**（系统保留）：
  - 只读，自动跟随 `skill.latest_version_id`
  - 严格等价于"最新已发布版本"
  - 不允许 API 手动移动
- **自定义标签**（如 `beta`、`stable`）：
  - 允许人工创建和移动
  - 唯一约束：`(skill_id, tag_name)`
  - 必须指向 `status = PUBLISHED` 的版本

#### 回滚机制

- **设计决策**：通过自定义标签实现稳定通道管理
- **示例**：创建 `stable` 标签，手动指向稳定版本

### 搜索与发现

#### PostgreSQL 全文搜索（一期）

- **索引表**：`skill_search_document`
- **索引字段**：`search_vector` (tsvector)
- **索引内容**：
  - `displayName`、`slug`、`summary`
  - frontmatter 中除 `name` / `description` / `version` 外的字段展开结果
- **索引维护**：通过触发器或 `GENERATED ALWAYS AS` 自动维护

#### 搜索过滤器

- 按命名空间筛选
- 按下载量筛选
- 按评分筛选
- 按更新时间筛选

#### 可见性控制

- 搜索结果自动过滤 `hidden=true` 和 `latest_version_id is null` 的技能
- 根据用户权限动态过滤 `NAMESPACE_ONLY` 和 `PRIVATE` 技能

### 社交功能

#### 收藏（Star）

- 唯一约束：`(skill_id, user_id)`
- 冗余字段：`skill.star_count`

#### 评分（Rating）

- 分值范围：1-5 分
- 唯一约束：`(skill_id, user_id)`，每人每技能一条，可修改
- 冗余字段：`skill.rating_avg`、`skill.rating_count`

### 审计日志

#### 记录范围

- 发布、审核、下载、删除等关键操作
- 命名空间成员变更
- 角色权限变更

#### 审计表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| actor_user_id | varchar(128) | 操作人 |
| action | varchar(64) | 操作类型 |
| target_type | varchar(64) | 目标类型 |
| target_id | bigint | 目标 ID |
| request_id | varchar(64) | 请求 ID |
| client_ip | varchar(64) | 客户端 IP |
| user_agent | varchar(512) | User-Agent |
| detail_json | json | 详细信息 |
| created_at | datetime | 创建时间 |

---

## 兼容层支持

### ClawHub CLI 兼容层

#### 协议兼容接口

一期聚焦以下核心接口：
- `search`：搜索技能
- `resolve`：解析技能坐标到版本
- `download`：下载技能包
- `publish`：发布技能
- `whoami`：获取当前用户信息

#### Well-known 发现

- 提供 `/.well-known/clawhub.json`
- 返回 `{ "apiBase": "/api/v1" }`
- ClawHub CLI 通过此机制自动发现兼容层 API 基地址

#### 坐标映射

- **输入**：ClawHub CLI 使用 canonical slug（如 `my-skill` 或 `team-name--my-skill`）
- **内部转换**：转换为 namespace 坐标（如 `@global/my-skill` 或 `@team-name/my-skill`）
- **输出**：兼容层返回 canonical slug

---

## 部署架构

### 部署模型

#### 开发路径

```bash
make dev-all
```

- 前后端在宿主机运行
- `docker-compose.yml` 仅负责 PostgreSQL、Redis、MinIO

#### 交付路径

- GitHub Actions 构建并发布 `server` / `web` 镜像
- 用户通过 `compose.release.yml` 一键拉起全栈环境
- 发布镜像为多架构 manifest（`linux/amd64` + `linux/arm64`）

### 单机运行时

#### 访问入口

- `http://localhost/` → Web 容器（Nginx）
- `http://localhost/api/*` → Nginx 反向代理到 Spring Boot
- `http://localhost:8080/actuator/health` → 后端健康检查

#### 默认 Profile

- **`docker` profile**：容器运行时使用
  - 负责初始化首个管理员账户
  - 数据库、Redis、对象存储、站点公网地址通过环境变量注入
- **`local` profile**：开发环境使用
  - 启用 mock 登录旁路
  - 生产环境不启用此 profile

#### 默认管理员账户

- 用户名：`admin`
- 密码：`ChangeMe!2026`
- **生产环境务必修改密码**（`validate-release-config.sh` 会拒绝默认值）

### Kubernetes 部署

#### 基础清单

位于 `deploy/k8s/`：
- `configmap.yaml`
- `secret.yaml.example`
- `backend-deployment.yaml`
- `frontend-deployment.yaml`
- `services.yaml`
- `ingress.yaml`

#### 部署命令

```bash
kubectl apply -f deploy/k8s/configmap.yaml
kubectl apply -f deploy/k8s/secret.yaml
kubectl apply -f deploy/k8s/backend-deployment.yaml
kubectl apply -f deploy/k8s/frontend-deployment.yaml
kubectl apply -f deploy/k8s/services.yaml
kubectl apply -f deploy/k8s/ingress.yaml
```

### 分布式环境要求

| 组件 | 要求 | 职责 |
|------|------|------|
| PostgreSQL 16.x | 主从 | 主存储 |
| Redis 7.x | Sentinel 或 Cluster | Session 存储 + 分布式锁 + 幂等去重 |
| 对象存储 | LocalFile / MinIO / 云厂商 S3 | 技能包文件 + 预打包 zip |
| Ingress | Nginx Ingress Controller | 路由分发 + TLS 终止 |

---

## 数据库设计

### 核心约束

- **用户标识主键**：全链路使用 `varchar(128)`
- **唯一约束**：
  - `namespace.slug`
  - `(namespace_id, slug)` on `skill`
  - `(skill_id, version)` on `skill_version`
  - `(skill_id, tag_name)` on `skill_tag`
  - `(skill_id, user_id)` on `skill_star` / `skill_rating`
  - `(namespace_id, user_id)` on `namespace_member`
  - `(provider_code, subject)` on `identity_binding`

### 关键索引

| 表 | 索引 | 用途 |
|------|------|------|
| `namespace` | `(slug)` UNIQUE | 唯一约束 |
| `skill` | `(namespace_id, status)` | 命名空间内技能列表 |
| `skill` | `(namespace_id, slug)` UNIQUE | 唯一约束 |
| `skill_version` | `(skill_id, status)` | 版本列表 |
| `skill_version` | `(skill_id, version)` UNIQUE | 唯一约束 |
| `skill_tag` | `(skill_id, tag_name)` UNIQUE | 标签唯一约束 |
| `review_task` | `(namespace_id, status)` | 审核列表 |
| `review_task` | `(submitted_by, status)` | 我的提交 |
| `promotion_request` | `(source_skill_id)` | 按来源 skill 查询 |
| `promotion_request` | `(status)` | 待审核列表 |
| `skill_star` | `(user_id)` | 我的收藏 |
| `skill_star` | `(skill_id)` | 技能收藏数 |
| `audit_log` | `(created_at)` | 审计查询 |
| `audit_log` | `(actor_user_id, created_at)` | 用户操作历史 |

### 幂等设计

#### 幂等记录表（idempotency_record）

| 字段 | 类型 | 说明 |
|------|------|------|
| request_id | varchar(64) | 主键，客户端传入的 UUID v4 |
| resource_type | varchar(64) | 如 `skill_version`, `api_token` |
| resource_id | bigint | 业务操作产生的资源 ID |
| status | enum | `PROCESSING` / `COMPLETED` / `FAILED` |
| response_status_code | int | 原始响应状态码 |
| created_at | datetime | 创建时间 |
| expires_at | datetime | 过期时间（默认 24h） |

#### 幂等流程

1. 收到请求 → 插入 record（`PROCESSING`）
2. 业务处理
3. 更新为 `COMPLETED` + `resource_id`
4. 重复请求时查 record 返回已有结果

#### 实现层次

- **Redis**：快速去重缓存（SETNX）
- **PostgreSQL**：持久化兜底
- **定时任务**：清理过期记录

---

## 开发与运维

### 本地开发

#### 快速启动

```bash
# 启动完整本地栈（后端 + 前端 + 依赖）
make dev-all

# 或分别启动
make dev-backend    # 仅后端
make dev-web        # 仅前端
```

#### 访问地址

- Web UI: `http://localhost:3000`
- Backend API: `http://localhost:8080`

#### 默认账户（`local` profile）

- `local-user`：普通用户，用于发布和命名空间操作
- `local-admin`：超级管理员（`SUPER_ADMIN`），用于审核和管理流程
- `admin` / `ChangeMe!2026`：bootstrap 管理员（可账号密码登录）

#### 常用命令

```bash
make help                    # 显示所有可用命令
make test                    # 运行后端测试
make test-backend-app        # 运行 skillhub-app 及其依赖模块测试
make build-backend-app       # 构建 skillhub-app 及其依赖模块
make typecheck-web           # TypeScript 类型检查
make build-web               # 构建前端
make generate-api            # 重新生成 OpenAPI 类型
```

#### 注意事项

- 不要在 `server/` 下直接执行 `./mvnw -pl skillhub-app clean test`
- `skillhub-app` 依赖同仓库的 sibling modules，需要使用 `-am` 或 `make test-backend-app`

### API 契约同步

#### 生成 OpenAPI 类型

```bash
make generate-api
```

#### 严格检查

```bash
./scripts/check-openapi-generated.sh
```

- 启动本地依赖
- 启动后端
- 重新生成前端 schema
- 检查是否有未提交的变更

### 监控

#### Prometheus + Grafana

位于 `monitoring/`，抓取后端 Actuator Prometheus 端点。

```bash
cd monitoring
docker compose -f docker-compose.monitoring.yml up -d
```

#### 访问地址

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3001`（默认账号：`admin` / `admin`）

### 冒烟测试

```bash
./scripts/smoke-test.sh http://localhost:8080
```

---

## 集成与生态

### 智能体平台集成

#### OpenClaw

[OpenClaw](https://github.com/openclaw/openclaw) 是开源的智能体技能 CLI 工具。

```bash
# 配置注册中心地址
export CLAWHUB_REGISTRY=https://skillhub.your-company.com

# 认证（如需）
clawhub login --token YOUR_API_TOKEN

# 搜索和安装技能
npx clawhub search email
npx clawhub install my-skill
npx clawhub install my-namespace--my-skill

# 发布技能
npx clawhub publish ./my-skill
```

**提示**：上述命令不仅适用于 OpenClaw，通过指定安装目录（`--dir`），也可适用于其他的 CLI Coding Agent 或 Agent 助手。例如：
```bash
npx clawhub --dir ~/.claude/skills install my-skill
```

#### AstronClaw

[AstronClaw](https://agent.xfyun.cn/astron-claw) 是基于 OpenClaw 核心能力打造的云端 AI 助手，提供 24/7 在线服务。支持技能市场一键安装、仓库搜索、对话自动安装，以及管理和分发组织内部的自定义私有技能。

#### Loomy

[Loomy](https://loomy.xunfei.cn/) 是聚焦真实办公场景的桌面端 AI 工作搭子。深入打通本地文件和系统工具，为个人及小团队构建高效的自动化工作流。

#### astron-agent

[astron-agent](https://github.com/iflytek/astron-agent) 是科大讯飞星火智能体框架。存储在 SkillHub 中的技能可以被 astron-agent 引用和加载，实现从开发到生产的受治理、版本化的技能生命周期。

---

## 产品路线图

### 已完成（Phase 1-4）

- [x] 核心技能注册功能
- [x] 命名空间和团队管理
- [x] 审核和治理工作流
- [x] 全文搜索和筛选
- [x] 社交功能（收藏、评分、下载）
- [x] API 令牌管理
- [x] 账户合并
- [x] 国际化支持

### 规划中（Phase 5）

- [ ] 评论功能（含举报机制）
- [ ] 自动安全扫描（接入 `PrePublishValidator` 扩展点）
- [ ] 举报/标记机制（配合评论和治理闭环）
- [ ] Webhook/事件通知
- [ ] Helm Chart 部署

### 未来规划

- [ ] 高级搜索过滤器
- [ ] 技能依赖管理
- [ ] 审计日志导出
- [ ] LDAP/SAML 集成
- [ ] 向量搜索（仅搜索增强，不引入推荐系统）

### 明确不做

- 在线编辑器（暂不规划）
- 技能兼容性声明（预留 `parsed_metadata_json` 字段）

---

## 安全与合规

### 文件上传安全

#### 扩展名白名单

默认白名单定义在 [`SkillPackagePolicy.java`](./server/skillhub-domain/src/main/java/com/iflytek/skillhub/domain/skill/validation/SkillPackagePolicy.java)。

#### 运行时覆盖

通过环境变量整体替换默认白名单：

```bash
SKILLHUB_PUBLISH_ALLOWED_FILE_EXTENSIONS=.md,.json,.xsd,.xsl,.dtd,.docx,.xlsx,.pptx
```

### 访问控制

#### 匿名访问

- 公共技能（`visibility=PUBLIC`）匿名可浏览和下载
- 无需登录即可访问搜索、详情、下载等接口

#### 认证访问

- OAuth2 登录后获得完整权限
- CLI 通过 Device Flow 或 API Token 认证

#### 权限层级

1. **平台角色**（RBAC）：`SUPER_ADMIN` / `SKILL_ADMIN` / `USER_ADMIN` / `AUDITOR`
2. **命名空间角色**：`OWNER` / `ADMIN` / `MEMBER`
3. **资源级授权**：基于 skill owner、namespace membership、visibility 规则

### 审计日志

- 记录所有关键操作（发布、审核、下载、删除等）
- 包含操作人、时间、IP、User-Agent、详细信息
- 仅 `AUDITOR` / `SUPER_ADMIN` 可查看

---

## 扩展点与未来演进

### 搜索扩展

- **当前**：PostgreSQL 全文搜索
- **未来**：Elasticsearch / OpenSearch / 向量检索
- **SPI 接口**：`SearchIndexService`, `SearchQueryService`, `SearchRebuildService`

### 存储扩展

- **当前**：LocalFile（开发）+ S3（生产）
- **未来**：阿里云 OSS、腾讯云 COS 等
- **SPI 接口**：`ObjectStorageService`

### 认证扩展

- **当前**：GitHub OAuth
- **未来**：GitLab、Google、企业 SSO（LDAP/SAML）
- **架构**：Spring Security OAuth2 Client 多 Provider 配置

### 审核扩展

- **当前**：人工审核
- **未来**：自动安全扫描、规则引擎
- **扩展点**：`PrePublishValidator`（当前为 NoOp）

### 通知扩展

- **当前**：无
- **未来**：Webhook、邮件、企业 IM
- **预留**：事件基础设施（Spring Events）

---

## 贡献指南

### 开发流程

详见 [docs/dev-workflow.md](docs/dev-workflow.md)。

### 规范

- **贡献指南**：[CONTRIBUTING.md](./CONTRIBUTING.md)
- **行为准则**：[CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md)

### 支持渠道

- 💬 **社区讨论**：[GitHub Discussions](https://github.com/iflytek/skillhub/discussions)
- 🐛 **Bug 报告**：[Issues](https://github.com/iflytek/skillhub/issues)
- 👥 **企业微信群**：见 README

---

## 许可证

Apache License 2.0

---

## 附录

### 术语表

| 术语 | 说明 |
|------|------|
| Skill | 智能体技能，包含 SKILL.md 及相关文件的技能包 |
| Namespace | 命名空间，隔离和组织技能的边界 |
| Slug | URL 友好标识，如 `my-skill` |
| Canonical Slug | 兼容层坐标，如 `team-name--my-skill` |
| Semver | 语义化版本，如 `1.2.3` |
| Device Flow | OAuth2 设备授权流程，用于 CLI 认证 |
| RBAC | 基于角色的访问控制（Role-Based Access Control） |
| Frontmatter | YAML 格式的元数据头，位于 Markdown 文件顶部 |
| SPI | 服务提供者接口（Service Provider Interface） |

### 参考文档

- 📖 **[用户指南](https://iflytek.github.io/skillhub/)** — 技能发布、搜索、CLI 使用等用户操作指南
- 🛠️ **[开发者文档](https://zread.ai/iflytek/skillhub)** — 架构设计、API 参考、本地开发、部署运维等技术文档
- 📄 **系统架构设计**：[docs/01-system-architecture.md](docs/01-system-architecture.md)
- 📄 **领域模型与数据模型**：[docs/02-domain-model.md](docs/02-domain-model.md)
- 📄 **产品定位与 MVP 范围**：[docs/00-product-direction.md](docs/00-product-direction.md)

---

**版本**：v1.0
**最后更新**：2026-04-09
