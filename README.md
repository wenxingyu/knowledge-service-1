# OpenAN Knowledge Service

OpenAN 知识服务提供知识库和目录管理、文档解析及文本切片。

> **当前边界**：HTTP 接口覆盖知识库/目录/文档/算子的管理与查询。**文档上传、解析
> 重试与检索（`POST /retrieval`）的后端代码已实现，但生产入口默认未挂载**（见
> [服务边界](#服务边界)）——完整摄取/检索链路需经 OpenAN 上游平台，或自行启用相应路由。

## 快速开始

需要 Docker 和 Docker Compose。

```sh
# 1. 准备配置：复制示例并填写三个必填凭据（见「配置」）
cp deploy/.env.example deploy/.env

# 2. 启动 postgresql + opensearch + backend（--wait 等全部健康）
docker compose --env-file deploy/.env -f deploy/docker-compose.yml up -d --build --wait

# 3. 健康检查
curl http://localhost:8091/api/knowledge-service/v1/health   # -> ok
```

### 获取 admin 初始密码

`RKB_ADMIN_PASSWORD` 留空时，启动期自动生成随机密码写入容器内
`/data/rkb/initial-admin-password`（权限 0600）：

```sh
docker compose --env-file deploy/.env -f deploy/docker-compose.yml \
  exec backend cat /data/rkb/initial-admin-password
```

⚠️ 该文件**不在数据卷上**，容器重建即消失——首次登录后请立即改密
（`POST /auth/password`，改密成功后此文件自动删除）。

### 登录与调用

默认 `AUTH_MODE=local`：除健康检查与 `/auth/status|login|logout` 外，
所有接口都要求会话 Cookie（`openan_session`，HttpOnly）。

```sh
# 登录（登录本身不要求 Origin）
curl -c cookies.txt -X POST http://localhost:8091/api/knowledge-service/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<初始密码>"}'

# 建库（props 必传；embeddingModel 必须与部署的 EMBEDDING_MODEL 一致，否则 0004）
curl -b cookies.txt -X POST \
  http://localhost:8091/api/knowledge-service/v1/knowledge-bases \
  -H 'Content-Type: application/json' -H 'Origin: http://localhost:8091' \
  -d '{"name":"demo","description":"示例知识库","props":{"embeddingModel":"bge-base-zh-v1.5"}}'

# 查询知识库列表
curl -b cookies.txt http://localhost:8091/api/knowledge-service/v1/knowledge-bases
```

规则速记：`POST/PUT/PATCH/DELETE` 必须带 `Origin`（或 `Referer`）头且其
host 与请求 `Host` 一致（CSRF 防护，`curl` 不带会得到 `403 csrf_rejected`）；
未登录访问业务接口返回 `401`；登录连续失败会触发 `429` 防爆破。

## HTTP 接口

统一前缀 `/api/knowledge-service/v1`。生产环境禁用了 Swagger（`/docs`、
`/openapi.json`），完整清单如下：

| 域 | 方法与路径 | 说明 |
|---|---|---|
| 健康 | GET `/health` | 存活探针，无需登录 |
| 认证 | GET `/auth/status` · POST `/auth/login` · POST `/auth/logout` · GET `/auth/me` · POST `/auth/password` | 本地账号会话 |
| 系统 | GET `/system/info` · GET `/system/file-size-limits` | 服务信息 / 上传大小限制 |
| 知识库 | GET·POST `/knowledge-bases` · GET `/knowledge-bases/stats` · GET·PATCH·DELETE `/knowledge-bases/{id}` · GET `/knowledge-bases/{id}/graph` | 建库（`?type=graph` 建图谱库）/ 改 / 删 / 查；同名空库自动复用 |
| 目录 | GET·POST `/knowledge-bases/{id}/directories` · GET·PATCH·DELETE `.../directories/{did}` | 目录树管理（删目录保留文档，移入 Default） |
| 文档 | GET·PATCH `/knowledge-bases/{id}/documents` · GET·DELETE `.../documents/{did}` · GET `.../documents/{did}/content` · GET `.../documents/{did}/chunks` | 分页检索 / 批量移动 / 删除 / 下载原文件 / 单文档切片 |
| 切片 | GET `/knowledge-bases/{id}/chunks` | 关键词分页查询 |
| 算子 | GET·POST `/operators` · GET `/operators/types` · GET `/operators/template` · GET `/operators/available` · GET `/operators/supported` · GET·DELETE `/operators/{id}` · POST·DELETE `/operators/{id}/publish` · PUT `/operators/{id}/visibility` · GET `/operators/{id}/export` | 算子包管理、发布与导出 |

以下接口后端代码已实现、生产**未挂载**：`POST .../documents`（上传）、
`POST .../documents/{id}/retry`（重试解析）、`POST .../chunks`（手工切片）、
`POST /retrieval`（检索）。

## 服务边界

- 本仓库是从 OpenAN 平台拆出的独立组件：embedding / rerank / 生成等模型服务
  **不在**本仓库的 compose 部署范围内，`EMBEDDING_MODEL` / `EMBEDDING_DIMENSION`
  等变量供上游接入与建库校验使用。
- 摄取（上传 → 解析 → 切片）与检索路由在代码中已具备：`backend/src/knowledge_base/api.py`
  的 `enable_ingestion` / `enable_manual_chunks` 参数与 `backend/src/retrieval/api.py`。
  生产入口 `backend/src/app.py` 显式以 `enable_ingestion=False, enable_manual_chunks=False`
  注册，因为完整链路依赖上游模型服务。自行集成时改传 `True` 并挂载 retrieval 路由；
  文档解析与切片也可不经 HTTP、直接调用 `backend/src/parsing` / `backend/src/ingestion` 模块。

## 配置

复制 `deploy/.env.example` 为 `deploy/.env`；所有配置经环境变量注入：

| 变量 | 必填 | 说明 |
|---|---|---|
| `POSTGRES_PASSWORD` | ✅ | 数据库超级用户口令（建议 32 位随机值） |
| `RKB_DB_PASSWORD` | ✅ | 应用角色 `rag_app` 口令（非 superuser，空卷初始化时自动创建） |
| `RKB_SECRET_KEY` | ✅ | compose 强制必填（32+ 字节随机值）；当前后端代码未消费——会话令牌为高熵随机串、库内仅存 SHA-256 摘要，不依赖该密钥。保留以对齐上游平台配置 |
| `RKB_ADMIN_PASSWORD` | – | 留空则启动期随机生成到 `/data/rkb/initial-admin-password` |
| `AUTH_MODE` | – | `local`（默认）/ `none` / `gateway`（网关令牌模式，须由前置网关剥离客户端伪造的身份头） |
| `KNOWLEDGE_BASE_PORT` | – | 对外端口，默认 8091 |
| `RKB_AUTH_SESSION_TTL_SECONDS` | – | 会话有效期，默认 12h，钳制 [60s, 30d] |
| `RKB_AUTH_LOGIN_MAX_FAILURES` / `RKB_AUTH_LOGIN_IP_MAX_FAILURES` / `RKB_AUTH_LOGIN_WINDOW_SECONDS` | – | 登录防爆破阈值（按账号+IP 与 IP 双维度滑动窗口） |
| `RKB_AUTH_COOKIE_SECURE` | – | `auto`（默认，按 `X-Forwarded-Proto`）/ `true` / `false` |
| `URL_CONTEXT_PATH` | – | 反向代理子路径（root_path），如经 nginx 挂在子路径下时设置 |
| `RKB_STORAGE_BACKEND` | – | 文件存储后端，`local`（默认，卷 `/data/kb`） |
| `EMBEDDING_MODEL` / `EMBEDDING_DIMENSION` | – | 向量模型名与维度（默认 `bge-base-zh-v1.5` / 1024）；建库时 `props.embeddingModel` 必须与其一致 |
| `RKB_SLOW_REQUEST_LOG_MS` | – | 慢请求告警阈值，默认 2000ms |

## 故障排查

| 现象 | 处理 |
|---|---|
| OpenSearch 起不来 | 内核 `vm.max_map_count` 需 ≥ 262144：`sudo sysctl -w vm.max_map_count=262144` |
| 端口被占 | 改 `.env` 的 `KNOWLEDGE_BASE_PORT` |
| 登录成功但业务接口一直 401 | 首登处于「必须改密」状态，只放行 `/auth/me`、`/auth/password`，先改密 |
| POST 返回 `403 csrf_rejected` | 补 `Origin` 头，host 必须与请求 `Host` 一致 |
| 登录返回 429 | 触发防爆破，按响应 `Retry-After` 等待，或调整阈值变量 |
| 看日志 | `docker compose --env-file deploy/.env -f deploy/docker-compose.yml logs -f backend` |

数据卷：`pgdata`（关系库）/ `osdata`（全文索引）/ `kbdata`（文件，容器内
`/data/kb`）。`down` 保留卷；`down -v` 连数据一起清除。

## 后端测试

Linux + Python 3.11，在 `backend/` 目录：

```sh
python -m pip install -r requirements.txt -r requirements-dev.txt
python -m compileall -q src tests
python -m pytest tests -q                  # 单元与 HTTP 契约
python -m pytest src/ingestion/tests -q    # 文档解析与算子沙箱
ruff check src tests
```

## 许可

Apache-2.0。版权与归属声明见 [NOTICE](NOTICE)。
