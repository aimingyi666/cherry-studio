---
description: 权限感知 RAG、Cherry 与 WLT-KB 联邦检索、本机发布到企业共享库
sources:
  - src/main/features/knowledge
  - src/main/features/knowledge/query/visibility.ts
  - src/main/ai/tools/knowledgeLookup.ts
  - src/main/ai/utils/knowledgeScope.ts
  - .cursor/rules/enterprise-edition-plan.mdc
---

# 权限感知 RAG 与知识联邦

## 原则（对齐飞书）

1. **Authorization-First**：模型只能看到已过滤内容。
2. **AI 权限 = 文档权限**：list / search / read / agent 同一权威。
3. **用户身份检索**：Bearer 员工 JWT（或文档化的受控机器凭证）；禁止无身份上帝搜库。
4. **召回 ≠ 返回**：命中后仍可按 ACL 丢弃。

Cherry 本机挂点（过渡期本地 ACL 或过滤）：

- [`visibility.ts`](../../../src/main/features/knowledge/query/visibility.ts)
- [`knowledgeScope.ts`](../../../src/main/ai/utils/knowledgeScope.ts)
- [`knowledgeLookup.ts`](../../../src/main/ai/tools/knowledgeLookup.ts)（`kb_search` / `kb_read`）

企业终态权威在 **WLT-KB**（见 [wlt-kb-openapi-for-llm.md](./wlt-kb-openapi-for-llm.md)）。

## Cherry 如何获取共享库并检索回答

```text
登录 Backend → JWT
  → GET KB「当前用户可读企业库」→ 填「企业」列表
  → 用户/Agent 勾选企业库进 knowledgeScope
  → 提问：KnowledgeRouter
        本机 id → KnowledgeService
        企业 id → WltKbClient（Bearer 用户 JWT）→ KB search/chat
  → 片段映射为 citation → 模型生成答案
```

KB 不可达时企业库明确失败，**不得**静默用空本机库冒充有知识。

## 本机文档变为共享

**发布到 WLT-KB**（选目标库 + 可见性）→ KB 入库建索引 → 本地标记已发布与 enterprise id。  
他人只通过 KB ACL 看见；企业问答只查 KB。  
推荐本地保留草稿，再发布/同步；不要只做本机 `public` 标记。

## 可见性语义（与 KB 对齐，字段以新代码为准）

过渡期可镜像：`public` / `department` / `private` + 文档级收紧。  
终态以 WLT-KB `accessible_*` 为准，Cherry 不维护第二套企业向量中台。

## 相关文档

- 套件总览：[enterprise-suite.md](./enterprise-suite.md)
- KB 对外 LLM API：[wlt-kb-openapi-for-llm.md](./wlt-kb-openapi-for-llm.md)
- 本机知识服务：[knowledge-service.md](./knowledge-service.md)
