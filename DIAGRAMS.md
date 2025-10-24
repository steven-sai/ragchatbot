# 系统架构图和产品逻辑图

## 1. 系统架构图 (System Architecture Diagram)

### 1.1 整体系统架构

```mermaid
graph TB
    subgraph "前端层 Frontend"
        A[Web Browser<br/>浏览器]
        A1[index.html<br/>界面]
        A2[script.js<br/>逻辑]
        A3[style.css<br/>样式]
        A --> A1
        A --> A2
        A --> A3
    end

    subgraph "API 网关层 API Gateway"
        B[FastAPI Server<br/>app.py]
        B1[POST /api/query<br/>查询接口]
        B2[GET /api/courses<br/>课程接口]
        B --> B1
        B --> B2
    end

    subgraph "核心业务层 Core Business Layer"
        C[RAG System<br/>rag_system.py]
        C1[Session Manager<br/>会话管理]
        C2[AI Generator<br/>ai_generator.py]
        C3[Tool Manager<br/>search_tools.py]
    end

    subgraph "AI 服务层 AI Service Layer"
        D[Claude AI<br/>Anthropic API]
        D1[Claude Sonnet 4<br/>大语言模型]
        D2[Tool Calling<br/>工具调用机制]
    end

    subgraph "数据存储层 Data Storage Layer"
        E[Vector Store<br/>vector_store.py]
        E1[ChromaDB]
        E2[Course Catalog<br/>课程目录集合]
        E3[Course Content<br/>课程内容集合]
        E1 --> E2
        E1 --> E3
    end

    subgraph "文档处理层 Document Processing"
        F[Document Processor<br/>document_processor.py]
        F1[TXT Parser<br/>文本解析]
        F2[PDF Parser<br/>PDF解析]
        F3[DOCX Parser<br/>Word解析]
        F4[Text Chunker<br/>文本分块]
        F --> F1
        F --> F2
        F --> F3
        F --> F4
    end

    subgraph "嵌入服务层 Embedding Service"
        G[Sentence Transformers<br/>all-MiniLM-L6-v2]
    end

    subgraph "数据源 Data Source"
        H[Course Documents<br/>/docs 文件夹]
        H1[course1_script.txt]
        H2[course2_script.txt]
        H3[course3_script.txt]
        H4[course4_script.txt]
        H --> H1
        H --> H2
        H --> H3
        H --> H4
    end

    subgraph "配置管理 Configuration"
        I[Config<br/>config.py]
        I1[.env<br/>环境变量]
    end

    %% 数据流连接
    A2 -->|HTTP Request| B
    B --> C
    C --> C1
    C --> C2
    C --> C3
    C2 -->|API Call| D
    D --> D1
    D1 --> D2
    D2 -->|Tool Execution| C3
    C3 -->|Semantic Search| E
    E --> E1
    C2 -->|Response| B
    B -->|JSON Response| A2

    %% 文档摄取流程
    H --> F
    F -->|Parsed Content| G
    G -->|Embeddings| E

    %% 配置
    I --> C
    I1 --> I

    style A fill:#e1f5ff
    style B fill:#fff3e0
    style C fill:#f3e5f5
    style D fill:#e8f5e9
    style E fill:#fff9c4
    style F fill:#fce4ec
    style G fill:#f1f8e9
    style H fill:#e0f2f1
    style I fill:#f5f5f5
```

### 1.2 三层架构视图

```mermaid
graph LR
    subgraph "表现层 Presentation Layer"
        P1[Web UI<br/>HTML/CSS/JS]
    end

    subgraph "业务逻辑层 Business Logic Layer"
        L1[RAG Engine<br/>rag_system.py]
        L2[AI Generator<br/>ai_generator.py]
        L3[Session Manager<br/>session_manager.py]
        L4[Tool Manager<br/>search_tools.py]
        L5[Document Processor<br/>document_processor.py]
    end

    subgraph "数据访问层 Data Access Layer"
        D1[Vector Store<br/>vector_store.py]
        D2[ChromaDB<br/>向量数据库]
        D3[Sentence Transformers<br/>嵌入模型]
    end

    P1 <-->|REST API| L1
    L1 --> L2
    L1 --> L3
    L1 --> L4
    L1 --> L5
    L2 --> L4
    L4 --> D1
    L5 --> D3
    D1 --> D2
    D3 --> D2

    style P1 fill:#bbdefb
    style L1 fill:#c8e6c9
    style L2 fill:#c8e6c9
    style L3 fill:#c8e6c9
    style L4 fill:#c8e6c9
    style L5 fill:#c8e6c9
    style D1 fill:#fff9c4
    style D2 fill:#fff9c4
    style D3 fill:#fff9c4
```

---

## 2. 产品逻辑图 (Product Logic Diagram)

### 2.1 完整业务流程图

```mermaid
flowchart TD
    Start([用户访问系统]) --> LoadUI[加载 Web 界面]
    LoadUI --> LoadCourses[后台加载课程统计]
    LoadCourses --> DisplayUI[显示界面和建议问题]

    DisplayUI --> UserInput{用户输入查询}

    UserInput -->|输入问题| CreateSession[创建/获取 Session]
    CreateSession --> SendQuery[发送查询到后端]

    SendQuery --> CheckSession{检查会话历史}
    CheckSession -->|有历史| LoadHistory[加载对话上下文]
    CheckSession -->|无历史| NewSession[创建新会话]

    LoadHistory --> SendToClaude[发送到 Claude AI]
    NewSession --> SendToClaude

    SendToClaude --> ClaudeAnalyze{Claude 分析查询}

    ClaudeAnalyze -->|需要搜索| UseTool[调用搜索工具]
    ClaudeAnalyze -->|无需搜索| DirectAnswer[直接生成答案]

    UseTool --> ParseParams[解析工具参数]
    ParseParams --> CourseMatch{课程名匹配?}

    CourseMatch -->|指定课程| FilterByCourse[按课程过滤]
    CourseMatch -->|未指定| SearchAll[搜索所有课程]

    FilterByCourse --> LessonFilter{Lesson 过滤?}
    SearchAll --> LessonFilter

    LessonFilter -->|指定 Lesson| FilterByLesson[按 Lesson 过滤]
    LessonFilter -->|未指定| VectorSearch[语义向量搜索]

    FilterByLesson --> VectorSearch
    VectorSearch --> GetResults[获取相关内容]

    GetResults --> FormatResults[格式化结果<br/>+ 来源信息]
    FormatResults --> ReturnToClaude[返回给 Claude]

    ReturnToClaude --> GenerateAnswer[生成最终答案]
    DirectAnswer --> GenerateAnswer

    GenerateAnswer --> SaveHistory[保存到会话历史]
    SaveHistory --> ReturnResponse[返回答案 + 来源]

    ReturnResponse --> DisplayAnswer[前端展示答案]
    DisplayAnswer --> ShowSources[展示来源引用]

    ShowSources --> NextQuery{继续提问?}
    NextQuery -->|是| UserInput
    NextQuery -->|否| End([结束])

    style Start fill:#4caf50,color:#fff
    style End fill:#f44336,color:#fff
    style ClaudeAnalyze fill:#2196f3,color:#fff
    style UseTool fill:#ff9800,color:#fff
    style VectorSearch fill:#9c27b0,color:#fff
    style GenerateAnswer fill:#00bcd4,color:#fff
    style DisplayAnswer fill:#8bc34a,color:#fff
```

### 2.2 文档摄取流程

```mermaid
flowchart TD
    Start([系统启动]) --> CheckDocs{检查 /docs 文件夹}

    CheckDocs -->|有文件| ReadFiles[读取所有文档文件]
    CheckDocs -->|无文件| Skip([跳过加载])

    ReadFiles --> ParseLoop[遍历每个文件]
    ParseLoop --> ParseMetadata[解析文档元数据]

    ParseMetadata --> ExtractInfo[提取信息:<br/>- 课程标题<br/>- 讲师<br/>- 课程链接]
    ExtractInfo --> ParseLessons[解析 Lessons]

    ParseLessons --> ExtractLessons[提取每个 Lesson:<br/>- 编号<br/>- 标题<br/>- 链接<br/>- 内容]

    ExtractLessons --> ChunkText[文本分块处理]
    ChunkText --> CreateChunks[创建 Chunks:<br/>大小: 800 字符<br/>重叠: 100 字符]

    CreateChunks --> GenerateEmbeddings[生成嵌入向量<br/>Sentence Transformers]

    GenerateEmbeddings --> StoreCatalog[存储课程元数据<br/>→ course_catalog 集合]
    StoreCatalog --> StoreContent[存储课程内容<br/>→ course_content 集合]

    StoreContent --> CheckMore{还有文件?}
    CheckMore -->|是| ParseLoop
    CheckMore -->|否| Complete([加载完成])

    style Start fill:#4caf50,color:#fff
    style Complete fill:#2196f3,color:#fff
    style GenerateEmbeddings fill:#9c27b0,color:#fff
    style StoreCatalog fill:#ff9800,color:#fff
    style StoreContent fill:#ff9800,color:#fff
```

### 2.3 查询处理详细流程

```mermaid
sequenceDiagram
    participant U as 用户 (User)
    participant F as 前端 (Frontend)
    participant API as FastAPI
    participant RAG as RAG System
    participant SM as Session Manager
    participant AI as AI Generator
    participant Claude as Claude AI
    participant TM as Tool Manager
    participant VS as Vector Store
    participant DB as ChromaDB

    U->>F: 输入查询问题
    F->>API: POST /api/query<br/>{query, session_id}
    API->>RAG: query(query, session_id)

    RAG->>SM: get_session(session_id)
    SM-->>RAG: 返回会话历史

    RAG->>AI: generate_response(query, history)
    AI->>Claude: API Call<br/>+ 系统提示词<br/>+ 工具定义<br/>+ 对话历史

    Claude->>Claude: 分析查询意图

    alt 需要搜索
        Claude-->>AI: tool_use: course_search<br/>{course_name, lesson, top_k}
        AI->>TM: execute_tool("course_search", params)
        TM->>VS: search(course_name, lesson, top_k)

        VS->>DB: 1. 课程名匹配<br/>(course_catalog)
        DB-->>VS: 返回匹配课程

        VS->>DB: 2. 语义搜索<br/>(course_content)
        DB-->>VS: 返回相关内容块

        VS->>VS: 格式化结果 + 元数据
        VS-->>TM: 返回搜索结果 + 来源
        TM-->>AI: tool_result

        AI->>Claude: 发送工具结果
        Claude->>Claude: 基于结果生成答案
    else 无需搜索
        Claude->>Claude: 直接生成答案
    end

    Claude-->>AI: 最终答案
    AI->>SM: add_exchange(query, answer)
    SM->>SM: 保存到会话历史<br/>(最多保留2轮)

    AI-->>RAG: {answer, sources}
    RAG-->>API: {answer, sources, session_id}
    API-->>F: JSON Response
    F->>F: 渲染答案 (Markdown)
    F->>F: 显示来源引用
    F-->>U: 展示结果
```

### 2.4 工具调用机制

```mermaid
flowchart TD
    Start([Claude 收到查询]) --> Analyze[分析查询内容]

    Analyze --> Decision{需要外部信息?}

    Decision -->|是| SelectTool[选择合适工具<br/>course_search]
    Decision -->|否| DirectGen[直接生成答案]

    SelectTool --> ParseQuery[从查询中提取参数]
    ParseQuery --> BuildParams[构建工具参数:<br/>- course_name<br/>- lesson_filter<br/>- top_k]

    BuildParams --> CallTool[调用工具]
    CallTool --> ExecuteSearch[执行语义搜索]

    ExecuteSearch --> GetContext[获取相关上下文]
    GetContext --> FormatContext[格式化上下文<br/>+ 课程名<br/>+ Lesson 标题<br/>+ 链接]

    FormatContext --> ReturnToModel[返回上下文给 Claude]
    ReturnToModel --> GenerateWithContext[基于上下文生成答案]

    DirectGen --> FinalAnswer[最终答案]
    GenerateWithContext --> FinalAnswer

    FinalAnswer --> AddSources[附加来源引用]
    AddSources --> Return([返回答案])

    style Decision fill:#2196f3,color:#fff
    style SelectTool fill:#ff9800,color:#fff
    style ExecuteSearch fill:#9c27b0,color:#fff
    style GenerateWithContext fill:#00bcd4,color:#fff
    style AddSources fill:#4caf50,color:#fff
```

### 2.5 用户交互流程

```mermaid
stateDiagram-v2
    [*] --> 访问页面
    访问页面 --> 加载界面
    加载界面 --> 等待输入

    等待输入 --> 输入查询: 用户输入
    输入查询 --> 处理中: 发送请求

    处理中 --> 显示答案: 收到响应
    处理中 --> 显示错误: 请求失败

    显示答案 --> 查看来源: 点击来源
    查看来源 --> 等待输入: 返回

    显示答案 --> 输入查询: 继续提问
    显示错误 --> 等待输入: 重试

    等待输入 --> [*]: 关闭页面
```

---

## 3. 数据流图 (Data Flow Diagram)

### 3.1 查询数据流

```mermaid
graph LR
    subgraph "输入 Input"
        A[用户查询<br/>User Query]
        B[Session ID]
    end

    subgraph "处理 Processing"
        C[会话上下文<br/>Session Context]
        D[Claude AI<br/>分析]
        E[工具调用<br/>Tool Calling]
        F[向量搜索<br/>Vector Search]
    end

    subgraph "数据源 Data Source"
        G[ChromaDB<br/>course_catalog]
        H[ChromaDB<br/>course_content]
    end

    subgraph "输出 Output"
        I[AI 答案<br/>Answer]
        J[来源引用<br/>Sources]
        K[新 Session ID]
    end

    A --> D
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    F --> H
    G --> F
    H --> F
    F --> D
    D --> I
    F --> J
    C --> K

    style A fill:#e3f2fd
    style D fill:#fff3e0
    style F fill:#f3e5f5
    style G fill:#e8f5e9
    style H fill:#e8f5e9
    style I fill:#c8e6c9
    style J fill:#c8e6c9
```

### 3.2 文档到向量的数据流

```mermaid
graph TD
    A[原始文档<br/>Raw Documents] --> B[文档解析器<br/>Document Parser]
    B --> C[课程元数据<br/>Course Metadata]
    B --> D[课程内容<br/>Course Content]

    C --> E[元数据对象<br/>Course Object]
    D --> F[文本分块<br/>Text Chunker]

    F --> G[文本块<br/>Text Chunks]
    G --> H[嵌入模型<br/>Embedding Model]

    H --> I[向量<br/>Vectors]
    E --> J[ChromaDB<br/>course_catalog]
    I --> K[ChromaDB<br/>course_content]

    K --> L[语义搜索<br/>Semantic Search]
    J --> L

    style A fill:#ffebee
    style H fill:#e1f5fe
    style I fill:#f3e5f5
    style J fill:#e8f5e9
    style K fill:#e8f5e9
    style L fill:#fff9c4
```

---

## 4. 组件依赖图 (Component Dependency Diagram)

```mermaid
graph TD
    A[app.py<br/>应用入口] --> B[rag_system.py<br/>RAG 引擎]
    B --> C[ai_generator.py<br/>AI 生成器]
    B --> D[vector_store.py<br/>向量存储]
    B --> E[document_processor.py<br/>文档处理器]
    B --> F[session_manager.py<br/>会话管理]

    C --> G[search_tools.py<br/>工具管理]
    G --> D

    C --> H[Anthropic API<br/>Claude]
    D --> I[ChromaDB]
    E --> J[Sentence Transformers]

    A --> K[config.py<br/>配置]
    B --> K
    C --> K
    D --> K

    B --> L[models.py<br/>数据模型]
    D --> L
    E --> L

    style A fill:#4caf50,color:#fff
    style B fill:#2196f3,color:#fff
    style H fill:#ff9800,color:#fff
    style I fill:#9c27b0,color:#fff
    style J fill:#00bcd4,color:#fff
```

---

## 5. 部署架构图 (Deployment Architecture)

```mermaid
graph TB
    subgraph "用户端 Client Side"
        U[Web Browser<br/>浏览器]
    end

    subgraph "服务器端 Server Side"
        subgraph "Web 服务器 Web Server"
            W[Uvicorn<br/>ASGI Server<br/>:8000]
            W --> FA[FastAPI App]
        end

        subgraph "应用层 Application Layer"
            FA --> RAG[RAG System]
            RAG --> SM[Session Manager<br/>内存存储]
        end

        subgraph "AI 服务 AI Service"
            RAG --> AI[AI Generator]
            AI --> EXT[Anthropic API<br/>外部服务]
        end

        subgraph "数据存储 Data Storage"
            RAG --> VS[Vector Store]
            VS --> DB[(ChromaDB<br/>本地文件系统<br/>./chroma_db)]
        end

        subgraph "嵌入服务 Embedding Service"
            RAG --> EMB[Sentence Transformers<br/>本地模型]
        end
    end

    U <-->|HTTP/HTTPS| W

    style U fill:#e3f2fd
    style W fill:#fff3e0
    style FA fill:#f3e5f5
    style EXT fill:#ffebee
    style DB fill:#e8f5e9
    style EMB fill:#fff9c4
```

---

## 图表说明

### 系统架构图说明
1. **整体系统架构**: 展示了从前端到后端的完整技术栈和各层之间的交互关系
2. **三层架构视图**: 清晰展示表现层、业务逻辑层、数据访问层的分层设计
3. **组件依赖图**: 展示各个 Python 模块之间的依赖关系

### 产品逻辑图说明
1. **完整业务流程图**: 用户从访问到获得答案的完整流程
2. **文档摄取流程**: 系统启动时如何加载和处理课程文档
3. **查询处理详细流程**: 详细的序列图展示各组件交互时序
4. **工具调用机制**: Claude AI 如何决策和执行工具调用
5. **用户交互流程**: 状态机展示用户操作的各种状态转换

### 数据流图说明
1. **查询数据流**: 展示数据在查询过程中的流转路径
2. **文档到向量的数据流**: 展示文档如何转化为可搜索的向量

### 部署架构说明
- 展示系统在运行时的部署结构
- 包含网络层、应用层、存储层的完整视图
- 标注了端口号、存储路径等关键配置

---

## 如何查看这些图表

这些图表使用 **Mermaid** 语法编写,可以在以下平台查看:

1. **GitHub** - 直接在 GitHub 上查看此 Markdown 文件
2. **VS Code** - 安装 "Markdown Preview Mermaid Support" 插件
3. **在线编辑器** - https://mermaid.live/
4. **Notion, Obsidian** 等支持 Mermaid 的笔记工具

---

**文档版本**: 1.0
**创建日期**: 2025-10-24
**适用系统**: 课程材料 RAG 聊天机器人
