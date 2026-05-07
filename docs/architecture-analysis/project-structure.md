# 项目整体结构分析

## 1. 目录组织架构

```
aiflowy/
├── aiflowy-api/                    # API层（对外暴露接口）
│   ├── aiflowy-api-admin/          # 管理后台API
│   ├── aiflowy-api-mcp/            # MCP（Model Context Protocol）接口
│   ├── aiflowy-api-public/         # 公共API（第三方接入）
│   └── aiflowy-api-usercenter/     # 用户中心API
├── aiflowy-commons/                # 公共模块（可复用组件）
│   ├── aiflowy-common-ai/          # AI相关工具类
│   ├── aiflowy-common-all/         # 全量公共组件
│   ├── aiflowy-common-audio/       # 音频处理
│   ├── aiflowy-common-base/        # 基础工具
│   ├── aiflowy-common-cache/        # 缓存抽象
│   ├── aiflowy-common-captcha/      # 验证码
│   ├── aiflowy-common-satoken/      # 权限认证工具
│   └── aiflowy-common-web/         # Web通用组件
├── aiflowy-modules/                # 业务模块（核心业务逻辑）
│   ├── aiflowy-module-ai/           # AI核心模块（BOT、模型、知识库）
│   ├── aiflowy-module-auth/         # 认证模块
│   ├── aiflowy-module-system/       # 系统管理
│   ├── aiflowy-module-datacenter/   # 数据中心
│   ├── aiflowy-module-job/          # 定时任务
│   └── aiflowy-module-log/           # 日志模块
├── aiflowy-starter/                # 启动器（Spring Boot自动配置）
│   ├── aiflowy-starter-admin/       # 管理后台启动器
│   ├── aiflowy-starter-all/          # 全量启动器
│   ├── aiflowy-starter-codegen/      # 代码生成器
│   └── aiflowy-starter-public/      # 公共启动器
└── aiflowy-ui-admin/               # 前端管理后台（Vue 3）
    └── aiflowy-ui-usercenter/      # 前端用户中心
```

## 2. 模块职责划分

| 模块层级 | 模块名称 | 核心职责 | 关键类 |
|---------|---------|---------|--------|
| **API层** | `aiflowy-api-public` | 第三方API接入、Apikey认证 | `PublicBotController`, `PublicWorkflowController` |
| **API层** | `aiflowy-api-admin` | 管理后台API接口 | - |
| **API层** | `aiflowy-api-usercenter` | 用户中心API | - |
| **公共层** | `aiflowy-common-ai` | AI工具类、拦截器、文档处理 | `ChatSseEmitter`, `ToolLoggingInterceptor` |
| **公共层** | `aiflowy-common-cache` | 缓存配置与抽象 | - |
| **公共层** | `aiflowy-common-satoken` | SaToken权限认证封装 | `SaTokenUtil` |
| **业务层** | `aiflowy-module-ai` | AI核心业务逻辑 | `BotServiceImpl`, `ModelServiceImpl`, `DocumentCollectionServiceImpl` |
| **业务层** | `aiflowy-module-auth` | 用户认证与权限 | - |
| **业务层** | `aiflowy-module-system` | 系统配置管理 | - |
| **启动层** | `aiflowy-starter-all` | 应用启动入口 | `application.yml` |

## 3. 核心调用链路

### 3.1 整体请求处理流程

```mermaid
sequenceDiagram
    participant Client as 客户端/第三方应用
    participant Gateway as 网关/负载均衡
    participant API as PublicBotController
    participant Interceptor as PublicApiInterceptor
    participant Service as BotServiceImpl
    participant ModelSvc as ModelServiceImpl
    participant LLM as ChatModel(agentsflex)
    participant VectorDB as DocumentStore
    participant Searcher as DocumentSearcher
    participant SSE as SseEmitter

    Client->>Gateway: HTTP POST /public-api/bot/chat
    Gateway->>API: 路由到对应Controller
    API->>Interceptor: 前置拦截器校验

    Interceptor->>Interceptor: 校验Apikey权限
    Interceptor->>API: 权限校验通过

    API->>Service: checkChatBeforeStart(botId, prompt, conversationId)

    Service->>Service: 获取Bot配置信息
    Service->>ModelSvc: getModelInstance(modelId)
    ModelSvc-->>Service: 返回Model实体（含Provider信息）

    Service->>Service: 校验匿名访问权限
    Service->>Service: 校验模型是否配置
    Service-->>API: ChatCheckResult（Bot、ChatModel、配置）

    API->>Service: startPublicChat(...)
    Service->>Service: 构建MemoryPrompt
    Service->>Service: 加载工具列表

    par 并行工具准备
        Service->>Service: 加载知识库工具
        Service->>Service: 加载工作流工具
        Service->>Service: 加载插件工具
        Service->>Service: 加载MCP工具
    end

    Service->>LLM: chatStream(memoryPrompt, listener, options)
    LLM->>LLM: 流式推理处理

    loop 工具调用循环（最多20次）
        alt 触发工具调用
            LLM-->>Service: 检测到工具调用请求
            Service->>VectorDB: 向量检索
            Service->>Searcher: 全文检索
            Service->>LLM: 返回工具执行结果
        else 生成最终回复
            LLM-->>SSE: 流式输出delta
        end
    end

    SSE-->>Client: Server-Sent Events流式响应
```

### 3.2 请求验证流程

```mermaid
flowchart TD
    A[接收HTTP请求] --> B{Apikey存在?}
    B -->|否| C[返回错误: Apikey不能为空]
    B -->|是| D[校验Apikey权限]
    D --> E{权限校验通过?}
    E -->|否| F[返回错误: 权限不足]
    E -->|是| G{BotId存在?}
    G -->|否| H[返回错误: 聊天助手不存在]
    G -->|是| I{已配置模型?}
    I -->|否| J[返回错误: 请配置大模型]
    I -->|是| K{匿名访问允许?}
    K -->|否| L{用户已登录?}
    K -->|是| M[继续处理]
    L -->|否| N[返回错误: 不支持匿名访问]
    L -->|是| M[继续处理]
```

### 3.3 数据流转过程

```mermaid
flowchart LR
    subgraph 输入数据
        A[用户Prompt] --> B[Messages历史]
        B --> C[Bot配置]
        C --> D[Model配置]
        D --> E[Provider配置]
    end

    subgraph 处理层
        F[参数校验] --> G[构建ChatOptions]
        G --> H[构建MemoryPrompt]
        H --> I[加载Tools]
        I --> J[异步推理]
    end

    subgraph 工具层
        J -->|知识库| K[Document检索]
        J -->|工作流| L[Workflow执行]
        J -->|插件| M[Plugin调用]
        J -->|MCP| N[MCP调用]
    end

    subgraph 输出层
        O[流式Delta] --> P[SSE推送]
        P --> Q[客户端渲染]
    end

    style A fill:#f9f,stroke:#333
    style Q fill:#9f9,stroke:#333
```

## 4. 关键模块依赖关系

```mermaid
graph TB
    subgraph 前端层
        UI_Admin[UI-Admin Vue3]
        UI_User[UI-UserCenter Vue3]
    end

    subgraph API层
        PublicAPI[PublicBotController]
        AdminAPI[AdminController]
    end

    subgraph 业务层
        BotSvc[BotServiceImpl]
        ModelSvc[ModelServiceImpl]
        DocSvc[DocumentCollectionServiceImpl]
        WorkflowSvc[WorkflowServiceImpl]
        PluginSvc[PluginServiceImpl]
        McpSvc[McpServiceImpl]
    end

    subgraph 公共层
        Cache[Cache抽象]
        Auth[SaToken认证]
        File[文件存储]
    end

    subgraph 存储层
        MySQL[(MySQL)]
        Redis[(Redis)]
        VectorDB[(向量数据库)]
        SearchDB[(搜索引擎)]
    end

    UI_Admin --> AdminAPI
    UI_User --> PublicAPI
    PublicAPI --> BotSvc
    AdminAPI --> BotSvc

    BotSvc --> ModelSvc
    BotSvc --> DocSvc
    BotSvc --> WorkflowSvc
    BotSvc --> PluginSvc
    BotSvc --> McpSvc

    DocSvc --> ModelSvc
    DocSvc --> VectorDB
    DocSvc --> SearchDB

    ModelSvc --> Cache
    BotSvc --> Auth
    BotSvc --> File

    ModelSvc --> MySQL
    BotSvc --> MySQL
    DocSvc --> MySQL
```

## 5. 核心业务逻辑分支

### 5.1 聊天请求处理分支

```mermaid
flowchart TD
    A[startPublicChat] --> B{是否为首次对话?}
    B -->|是| C[创建新Memory]
    B -->|否| D[加载历史Memory]

    C --> E[PublicBotMessageMemory]
    D --> F[DefaultBotMessageMemory]

    E --> G[设置SystemPrompt]
    F --> G

    G --> H[添加用户消息]
    H --> I[添加工具列表]

    I --> J[异步执行chatStream]

    J --> K{收到AI响应}
    K -->|文本响应| L[发送MESSAGE事件]
    K -->|思考内容| M[发送THINKING事件]
    K -->|工具调用| N[执行工具]

    L --> O{还有更多响应?}
    M --> O
    N --> J

    O -->|是| K
    O -->|否| P[发送DONE事件]

    style L fill:#9f9
    style M fill:#ff9
    style N fill:#f90
    style P fill:#09f
```

### 5.2 知识库检索分支

```mermaid
flowchart TD
    A[search知识库] --> B[获取配置]
    B --> C[初始化VectorStore]
    B --> D[获取EmbeddingModel]

    C --> E{向量库类型}
    E -->|Redis| F[RedisVectorStore]
    E -->|Milvus| G[MilvusVectorStore]
    E -->|ES| H[ElasticSearchVectorStore]
    E -->|阿里云| I[AliyunVectorStore]

    D --> J[执行embed查询]

    J --> K[并行召回]
    K --> L[向量检索]
    K --> M[全文检索]

    L --> N[合并结果]
    M --> N

    N --> O{配置了Rerank?}
    O -->|是| P[执行重排]
    O -->|否| Q[直接返回]

    P --> Q

    Q --> R[过滤低分结果]
    R --> S[返回TopK结果]

    style L fill:#9f9
    style M fill:#9f9
    style S fill:#09f
```

## 6. 响应生成与输出

### 6.1 SSE响应格式

```json
// 文本消息事件
{
  "domain": "LLM",
  "type": "MESSAGE",
  "payload": {
    "conversation_id": "uuid-string",
    "role": "assistant",
    "delta": "这是AI的回复内容..."
  }
}

// 思考内容事件
{
  "domain": "LLM",
  "type": "THINKING",
  "payload": {
    "conversation_id": "uuid-string",
    "role": "assistant",
    "delta": "正在思考..."
  }
}

// 错误事件
{
  "domain": "SYSTEM",
  "type": "ERROR",
  "payload": {
    "code": "SYSTEM_ERROR",
    "message": "错误描述",
    "retryable": false
  }
}

// 结束事件
{
  "domain": "SYSTEM",
  "type": "DONE",
  "payload": {}
}
```

### 6.2 响应处理流程

```mermaid
flowchart LR
    subgraph ChatStreamListener
        A[onMessage] --> B{是否为最终响应?}
        B -->|是且有工具调用| C[禁止停止]
        B -->|普通文本| D[发送MESSAGE事件]
        B -->|思考内容| E[发送THINKING事件]

        C --> F[执行工具调用]
        F --> G[将结果加入Memory]
        G --> H[继续chatStream]

        D --> I{是否已进工具调用?}
        E --> I
        I -->|是| J[允许停止]

        H --> A
    end

    subgraph SSEEmitter
        K[SseEmitter] --> L[send事件]
        L --> M[complete完成]
    end

    J --> K

    style K fill:#f9f
    style M fill:#9f9
```

## 7. 文件类型分类标准

| 文件类型 | 存放目录 | 说明 |
|---------|---------|------|
| Controller | `aiflowy-api-*/controller/` | 处理外部请求 |
| Service接口 | `aiflowy-module-*/service/` | 业务接口定义 |
| Service实现 | `aiflowy-module-*/service/impl/` | 业务逻辑实现 |
| Entity实体 | `aiflowy-module-*/entity/` | 数据模型 |
| Mapper | `aiflowy-module-*/mapper/` | 数据库映射 |
| 配置类 | `aiflowy-module-*/config/` | Spring配置 |
| 监听器 | `*-ai/agentsflex/listener/` | 事件监听处理 |
| 工具类 | `aiflowy-common-*/` | 公共工具 |

---

*返回 [文档首页](./README.md)*
