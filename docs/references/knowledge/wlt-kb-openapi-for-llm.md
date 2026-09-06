---
description: WLT-KB 须对外提供的 OpenAPI 能力，供员工自研 LLM/Agent 直连上传、检索与管理知识库（契约；实现在 KB 仓）
sources:
  - src/main/features/knowledge
  - src/main/ai/tools/knowledgeLookup.ts
  - docs/references/knowledge/enterprise-suite.md
  - docs/references/knowledge/permission-aware-rag.md
---

# WLT-KB：供外部 LLM / 业务系统调用的 OpenAPI 契约

**目标：** 除 Cherry / KB Web 外，其他员工开发的 LLM、Agent、业务后端必须能通过 **稳定 HTTP API** 完成知识库的创建、授权、上传、解析状态、检索、问答等操作，且 **权限与人机客户端同一套 ACL**（禁止「API 上帝通道」绕过可见性）。

> 实现归属 **WLT-KB** 仓。本文是生态契约：KB 新代码须满足；Cherry 联邦客户端与第三方 LLM 共用此面。路径/字段名在新代码落地后填入「校准表」，语义不得削弱。

## 1. 设计要求

| 要求 | 说明 |
|------|------|
| OpenAPI 3 | 提供 `/openapi.json` 或等价；建议 Scalar/Swagger UI |
| 鉴权 | **默认** `Authorization: Bearer <user_access_token>`（Backend 签发，与人登录同一 issuer） |
| 可选机器凭证 | 仅允许 **窄权限** service account（绑定服务身份 + 显式授予的库/角色）；禁止全局万能 Key 默认可读全库 |
| 权限前置 | 所有 list/get/search/chat/download 均按调用者 ACL 过滤；无权限 → `403`/`404`（不泄露存在性策略需统一文档化） |
| 多租户/组织 | 与 Backend `sub`、部门/角色 claims 一致 |
| 幂等与审计 | 写操作可追 actor；检索/问答建议审计 query 摘要 + 召回 id |
| 版本 | URL 前缀如 `/api/v1`；破坏性变更升版本 |

## 2. 能力清单（必须覆盖「知识库全部操作」）

第三方 LLM 开发至少需要下列能力（名称可为 REST 资源名，须在 OpenAPI 中完整出现）：

### 2.1 知识库（Library / Base）

- 列出当前调用者可见的知识库（分页、关键词）
- 获取单个库详情（含 settings：chunk/embedding/rerank 等若对外）
- 创建库（名称、描述、可见性 scope）
- 更新库元数据 / settings
- 删除或归档库（软删策略与人机一致）
- （推荐）库成员/ACL：授予 user/role、变更角色、移除

### 2.2 文档（Document / Item）

- 列出库内文档（状态：processing/ready/failed）
- 获取文档元数据
- **上传文档**（multipart 文件；以及 URL/纯文本若产品支持）
- 更新标题等元数据
- 删除文档
- 触发/查询重新索引
- 下载或预览原文（受 ACL）
- （推荐）文档级 ACL 收紧（比库更严）

### 2.3 检索与问答（供 LLM RAG）

- **语义/混合检索**：`query` + 可选 `kb_ids` + `top_k` → 片段、score、doc_id、标题、引用定位
- **问答/RAG chat**（可选但强烈建议）：流式 SSE/JSON；返回答案 + citations
- 检索范围不得超过调用者 ACL；`kb_ids` 只能收窄不能扩大

### 2.4 任务与状态

- 上传后异步索引：job id、进度、错误信息
- Webhook 或轮询接口（二选一，文档写清）

### 2.5 辅助

- 健康检查、版本号
- （推荐）用量配额查询

## 3. 建议的资源形状（校准前占位）

实际路径以 WLT-KB OpenAPI 为准；语义应对齐：

```http
GET    /api/v1/knowledge-bases
POST   /api/v1/knowledge-bases
GET    /api/v1/knowledge-bases/{kb_id}
PATCH  /api/v1/knowledge-bases/{kb_id}
DELETE /api/v1/knowledge-bases/{kb_id}

GET    /api/v1/knowledge-bases/{kb_id}/documents
POST   /api/v1/knowledge-bases/{kb_id}/documents          # multipart upload
GET    /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}
DELETE /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}
POST   /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}/reindex

POST   /api/v1/search                                     # body: query, kb_ids?, top_k?
POST   /api/v1/chat/completions                           # 或 /ask ；流式 + citations

GET    /api/v1/jobs/{job_id}
```

ACL 示例：

```http
GET/PUT/PATCH/DELETE /api/v1/knowledge-bases/{kb_id}/acl
```

## 4. 外部 LLM 接入示例（逻辑）

```text
1. 员工或服务用 Backend 换 token（或企业发放的 scoped service token）
2. GET knowledge-bases → 得到可操作的 kb_id 列表
3. POST documents 上传 → 轮询 job 至 ready
4. POST search(query, kb_ids) → 取 chunks
5. 自研 LLM 用 chunks 生成答案（或直接调 KB chat）
```

与 Cherry 联邦：**同一 OpenAPI**；Cherry 是其中一个官方客户端，不另开特权协议。

## 5. 安全红线

- 不得提供「跳过 ACL 的 admin 检索」给普通集成方。
- Service account 必须可审计、可吊销、可限制到库。
- 上传文件类型/大小限制与病毒扫描策略在 KB 运维文档中写明。
- CORS/网络：默认仅内网或零信任网关后暴露。

## 6. 与 Cherry 本机 API Gateway 的区别

| | Cherry 本地 ApiGateway | WLT-KB OpenAPI |
|--|------------------------|----------------|
| 进程 | 用户本机 Electron | 企业服务器 |
| 数据 | 本机库为主 | 企业多用户库 |
| 对象 | 其他员工 LLM **应优先调 KB** | 跨员工、跨应用共享知识 |

本机 Gateway 可继续服务个人自动化；**企业共享知识一律走 WLT-KB OpenAPI**。

## 7. 校准表（上传新 KB 代码后填写）

| 契约项 | 新代码位置 / 最终路径 | 负责人 |
|--------|----------------------|--------|
| OpenAPI URL | _待填_ | |
| Token 校验方式 | _待填_ | |
| 上传文档 | _待填_ | |
| 检索 | _待填_ | |
| 问答流式 | _待填_ | |
| ACL | _待填_ | |
| 错误码表 | _待填_ | |

## 8. 验收（KB 仓 + 任一外部客户端）

- 用员工 A token 上传并检索成功；员工 B 无 ACL 时 list/search 不可见。
- 仅持有 service token（限定库）不能读未授权库。
- OpenAPI 文档足以让第三方不读源码完成：建库 → 上传 → 等到 ready → search。
- Cherry 与外部 LLM 对同一 query + 同一用户可见片段集合一致（允许排序细微差，不允许权限差）。
