# Aiflowy 项目架构深度分析报告

## 文档目录

- [项目整体结构分析](./project-structure.md) - 包含目录组织架构、模块职责划分、核心调用链路
- [AI模块深度分析](./ai-module-analysis.md) - 包含模型架构、数据预处理、推理链路、接口规范

---

## 概述

本报告对 Aiflowy 项目进行了全面深度的架构分析，涵盖以下内容：

### 分析范围

| 分析维度 | 内容描述 |
|---------|---------|
| **项目结构分析** | 目录组织架构、模块划分、调用链路、数据流向 |
| **AI模块深度分析** | 模型架构设计、检索流程、推理链路、性能优化 |

### 项目技术栈

| 技术领域 | 使用的技术 |
|---------|-----------|
| 后端框架 | Spring Boot + MyBatis-Flex |
| AI框架 | agentsflex（对话、嵌入、重排） |
| 权限认证 | SaToken |
| 缓存 | JetCache + Redis |
| 向量数据库 | Redis/Milvus/ElasticSearch/阿里云/腾讯云 |
| 全文检索 | Lucene/ElasticSearch |
| 异步处理 | ThreadPoolTaskExecutor |
| 流式通信 | SSE（Server-Sent Events） |
| 前端框架 | Vue 3 + Vite + TypeScript + TailwindCSS |

---

## 快速导航

### AI模块核心组件

| 组件 | 文件路径 | 核心功能 |
|-----|---------|---------|
| [BotServiceImpl](./ai-module-analysis.md#22-bot服务层) | `aiflowy-module-ai/.../service/impl/BotServiceImpl.java` | Bot聊天核心逻辑 |
| [ModelServiceImpl](./ai-module-analysis.md#23-模型服务) | `aiflowy-module-ai/.../service/impl/ModelServiceImpl.java` | 模型配置与实例化 |
| [ChatStreamListener](./ai-module-analysis.md#24-流式监听器) | `aiflowy-module-ai/.../agentsflex/listener/ChatStreamListener.java` | 流式响应处理 |
| [DocumentCollectionServiceImpl](./ai-module-analysis.md#25-知识库检索) | `aiflowy-module-ai/.../service/impl/DocumentCollectionServiceImpl.java` | 知识库混合检索 |
| [VectorDatabase](./ai-module-analysis.md#3-向量数据库集成) | `aiflowy-module-ai/.../entity/VectorDatabase.java` | 多向量库适配 |

---

## 文档更新日志

| 版本 | 日期 | 更新内容 |
|-----|------|---------|
| v1.0 | 2026-05-07 | 初始版本，包含项目结构与AI模块深度分析 |

---

*本报告由代码分析工具自动生成*
