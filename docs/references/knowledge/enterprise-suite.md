---
description: 沃莱特企业 AI 生态套件结构（Backend/KB/Cherry/AppCenter）与分期；桌面唯一 Cherry
sources:
  - src/main/features/knowledge
  - src/main/ai/tools/knowledgeLookup.ts
  - src/main/services/OpenClawService.ts
  - .cursor/rules/enterprise-edition-plan.mdc
  - .cursor/rules/upstream-sync.mdc
---

# 沃莱特企业 AI 生态套件

本文是企业整合方案的本仓权威摘要。WLT-Backend / WLT-KB / AppCenter 的**最新实现以你方上传的新代码为准**；下文职责与分期冻结，具体 URL/字段待契约校准。

## 产品结构（四核 + 共享包）

| 模块 | 仓 | 职责 |
|------|-----|------|
| 身份与组织 | WLT-Backend | OIDC IdP、用户/部门/角色、应用目录、企业模型密钥、工作台壳 |
| 企业知识 | WLT-KB | 多用户文档、向量、服务端 RAG、ACL；**对外 LLM OpenAPI**（见 [wlt-kb-openapi-for-llm.md](./wlt-kb-openapi-for-llm.md)） |
| 桌面 Studio | **Cherry Studio（本仓，唯一企业桌面）** | Agent / MCP / Skills / 本机库；内置 OpenClaw（Code CLI）；企业登录与联邦检索客户端 |
| 分发更新 | WLT-AppCenter | 企业安装包 Releases（manifest）；主分发 Cherry 企业版 |
| 共享 UI | WLT-ChatUI | npm/workspace 包，不当独立产品 |

**退役：** WLT-ClawX（与 Cherry 桌面 Agent 重复；OpenClaw 用 Cherry 内置）。归档仓并下架企业目录主入口。

## 硬约束

1. 一套身份（Backend JWT）；Cherry/KB/第三方 LLM 均带**用户或受控凭证**访问。
2. 一套企业知识权限（KB 服务端强制）；AI 权限 = 文档权限（飞书式）。
3. 企业桌面只要 Cherry；不删 Cherry 已有功能（本机库、Agent、MCP、OpenClaw 等）。
4. 企业代码集中、少散改官方核心，便于 [upstream 同步](../../../.cursor/rules/upstream-sync.mdc)。

## 知识：本机 vs 企业

详见 [permission-aware-rag.md](./permission-aware-rag.md)。

- **本机库**：留在 Cherry；个人草稿。
- **企业/共享库**：权威在 WLT-KB；Cherry 用用户 token 联邦检索。
- **本机变共享**：发布到 KB（上传 + 选可见性），不是只在本机打共享标。

## 分期

| 期 | 交付 |
|----|------|
| 0 | 本仓文档与契约占位（本文档集） |
| 1 | Cherry ↔ Backend 登录；权限感知（过渡 ACL 或直连 KB 列表） |
| 2 | 企业模型下发；AppCenter 分发 Cherry；**发布到企业** API |
| 3 | 企业检索只调 KB（KnowledgeRouter） |
| 4 | 技能市场、审批、合规；完成 ClawX 归档 |

## 与官方 Cherry upstream

见 [.cursor/rules/upstream-sync.mdc](../../../.cursor/rules/upstream-sync.mdc)：`upstream` = CherryHQ，按 release 合并。
