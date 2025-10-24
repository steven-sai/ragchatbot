# 课程材料 RAG 聊天机器人 - 系统架构文档

## 项目概述

这是一个基于 **RAG (Retrieval-Augmented Generation，检索增强生成)** 技术的智能课程问答系统。用户可以通过自然语言查询课程内容，系统会通过语义搜索找到相关内容，并利用 Claude AI 生成准确、有上下文的回答。

---

## 技术栈

### 后端技术
- **FastAPI** (v0.116.1) - 高性能 Web 框架
- **Uvicorn** (v0.35.0) - ASGI 服务器
- **Anthropic Claude** (v0.58.2) - AI 模型（Claude Sonnet 4）
- **ChromaDB** (v1.0.15) - 向量数据库，用于语义搜索
- **Sentence Transformers** (v5.0.0) - 文本嵌入模型（all-MiniLM-L6-v2）
- **Python-dotenv** (v1.1.1) - 环境变量管理
- **Python 3.13+** - 运行时环境

### 前端技术
- **HTML5 + CSS3** - 界面结构和样式
- **Vanilla JavaScript** - 客户端逻辑
- **Marked.js** - Markdown 渲染

---

## 核心组件架构

### 1. **app.py - 应用入口**
FastAPI 应用的主入口，负责：
- 提供 REST API 端点
- 托管静态前端文件
- 启动时加载课程文档
- 配置 CORS 和中间件

**主要端点：**
- `POST /api/query` - 处理用户查询
- `GET /api/courses` - 获取课程统计信息
- `GET /` - 提供前端界面

### 2. **rag_system.py - RAG 核心引擎**
整个 RAG 系统的协调器，整合所有子系统：
- **功能：**
  - 添加单个课程文档
  - 批量加载课程文件夹
  - 处理用户查询（结合工具调用）
  - 提供课程分析统计

- **关键方法：**
  - `add_course_document()` - 单文档处理
  - `add_course_folder()` - 批量文档加载
  - `query()` - 查询处理（工具调用）
  - `get_course_analytics()` - 统计分析

### 3. **vector_store.py - 向量存储**
基于 ChromaDB 的向量数据库管理：
- **两个集合：**
  - `course_catalog` - 课程元数据（课程名、讲师、链接等）
  - `course_content` - 课程实际内容（按 Lesson 分块）

- **核心功能：**
  - 语义搜索（支持过滤）
  - 课程名称模糊匹配
  - Lesson 级别过滤
  - 元数据管理

### 4. **ai_generator.py - AI 生成器**
Claude API 集成层：
- **功能：**
  - 调用 Claude Sonnet 4 模型
  - 实现工具调用（Tool Calling）机制
  - 管理对话历史
  - 编排工具执行流程

- **配置：**
  - Temperature: 0（确定性输出）
  - Max Tokens: 800
  - 系统提示词定制

### 5. **document_processor.py - 文档处理器**
解析和处理课程文档：
- **支持格式：** TXT, PDF, DOCX
- **解析内容：**
  - 课程标题、讲师、链接
  - 课程的各个 Lessons（编号、标题、链接）
  - 课程内容文本

- **文本分块：**
  - Chunk 大小：800 字符
  - 重叠区域：100 字符
  - 保留 Lesson 边界
  - 上下文丰富化

### 6. **search_tools.py - 工具管理系统**
工具定义和执行框架：
- **抽象基类：** `Tool` - 可扩展的工具接口
- **具体工具：**
  - `CourseSearchTool` - 课程语义搜索
    - 支持课程名称模糊匹配
    - Lesson 过滤
    - 结果格式化

- **ToolManager** - 工具注册表和执行器
  - 工具注册
  - 工具调用分发
  - 来源追踪

### 7. **session_manager.py - 会话管理**
对话上下文管理：
- 基于 Session ID 的会话隔离
- 维护对话历史（用户/助手交互）
- 可配置历史记录限制（默认最多保留 2 轮）
- 自动历史修剪

### 8. **config.py - 配置管理**
集中式配置系统：
- 使用 `@dataclass` 定义配置
- 从 `.env` 文件加载环境变量
- **关键配置：**
  - Anthropic API Key 和模型选择
  - 嵌入模型规格
  - 分块参数
  - 数据库路径

### 9. **models.py - 数据模型**
Pydantic 数据模型定义：
- `Lesson` - 单个 Lesson 表示
- `Course` - 完整课程（包含多个 Lessons）
- `CourseChunk` - 用于向量存储的文本块

---

## 数据流架构

```
用户输入查询（前端）
    ↓
POST /api/query（FastAPI）
    ↓
RAG System 查询处理
    ├─ Session Manager（维护会话上下文）
    ├─ AI Generator（Claude）
    │   ├─ 系统提示词
    │   └─ 工具调用决策
    │       └─ 执行搜索工具
    │           └─ Vector Store 语义搜索（ChromaDB）
    │               ├─ Course Catalog（课程名匹配）
    │               └─ Course Content（内容检索）
    │
    └─ 生成答案 + 来源引用
        ↓
    返回前端展示
```

---

## 核心处理流程

### 1. **文档摄取流程**
```
读取 /docs 文件夹
    ↓
Document Processor 解析
    ├─ 提取元数据（课程、讲师、Lessons）
    ├─ 提取 Lesson 内容
    └─ 文本分块（800 字符，100 重叠）
    ↓
生成嵌入向量（Sentence Transformers）
    ↓
存储到 ChromaDB
    ├─ course_catalog 集合
    └─ course_content 集合
```

### 2. **查询处理流程**
```
接收用户查询
    ↓
Session Manager 加载会话上下文
    ↓
发送到 Claude AI（带工具定义）
    ↓
Claude 决策是否调用搜索工具
    ├─ 是 → 执行 CourseSearchTool
    │   ├─ 课程名称模糊匹配（course_catalog）
    │   ├─ 语义搜索内容（course_content）
    │   └─ 返回相关内容 + 元数据
    │
    └─ 否 → 直接生成回答
    ↓
Claude 基于检索内容生成最终答案
    ↓
返回答案 + 来源引用
    ↓
前端渲染展示
```

---

## 特性亮点

### 1. **语义搜索**
- 超越关键词匹配，理解查询意图
- 基于向量相似度的内容检索
- 支持自然语言查询

### 2. **工具调用 (Tool Calling)**
- Claude 自主决定是否需要搜索
- 动态执行搜索工具
- 结果自动整合到回答中

### 3. **会话上下文管理**
- 支持多轮对话
- 维护对话历史（最多 2 轮）
- 基于 Session ID 隔离会话

### 4. **精确来源追踪**
- 每个答案都附带来源引用
- 包含课程名、Lesson 标题和链接
- 可追溯答案依据

### 5. **智能文档处理**
- 自动解析课程结构
- Lesson 级别内容分割
- 保留上下文的文本分块

### 6. **批量文档加载**
- 一次性加载整个文件夹
- 自动去重（跳过已加载课程）
- 高效的批量处理

---

## API 接口设计

### POST /api/query
**请求：**
```json
{
  "query": "什么是机器学习？",
  "session_id": "optional-session-id"
}
```

**响应：**
```json
{
  "answer": "机器学习是...",
  "sources": [
    "Course: AI Fundamentals, Lesson 1: Introduction to ML",
    "Course: Deep Learning, Lesson 3: Neural Networks"
  ],
  "session_id": "generated-or-provided-session-id"
}
```

### GET /api/courses
**响应：**
```json
{
  "total_courses": 4,
  "course_titles": [
    "Course 1 Title",
    "Course 2 Title",
    "Course 3 Title",
    "Course 4 Title"
  ]
}
```

---

## 部署架构

```
运行环境：
├─ Python 3.13+
├─ uv 包管理器
└─ ChromaDB 本地存储

启动流程：
1. run.sh 脚本
2. 创建 /docs 目录
3. 启动 Uvicorn 服务器（端口 8000）
4. 自动加载课程文档
5. 前端通过 http://localhost:8000 访问
```

---

## 扩展性设计

### 1. **工具系统可扩展**
- 抽象 `Tool` 基类
- 轻松添加新工具（如网络搜索、计算器等）

### 2. **支持多种文档格式**
- 当前支持 TXT, PDF, DOCX
- 可扩展更多格式（PPT, Excel 等）

### 3. **会话管理灵活**
- 可调整历史记录长度
- 支持会话持久化（当前内存存储）

### 4. **向量数据库可替换**
- 当前使用 ChromaDB
- 可切换到 Pinecone、Weaviate 等

---

## 性能优化

1. **文档去重** - 避免重复加载相同课程
2. **文本分块** - 平衡上下文完整性和检索精度
3. **嵌入模型** - 使用轻量级 all-MiniLM-L6-v2（快速推理）
4. **温度设置** - Temperature=0 确保稳定输出
5. **Token 限制** - Max Tokens=800 控制成本

---

## 数据隐私与安全

- **本地向量存储** - ChromaDB 数据存储在本地
- **环境变量管理** - API Key 通过 .env 文件隔离
- **CORS 配置** - 支持跨域请求
- **无日志记录敏感信息** - 保护用户隐私

---

## 未来改进方向

1. **会话持久化** - 使用 Redis 或数据库存储会话
2. **用户认证** - 添加登录和权限管理
3. **多模态支持** - 支持图片、视频内容检索
4. **实时流式输出** - 改进用户体验
5. **更多工具** - 网络搜索、代码执行等
6. **部署优化** - Docker 容器化、云部署

---

## 总结

这是一个设计精良的 RAG 系统，展示了现代 AI 应用的最佳实践：
- ✅ 模块化设计，职责清晰
- ✅ 工具调用机制，AI 自主决策
- ✅ 向量数据库，语义搜索
- ✅ 会话管理，多轮对话
- ✅ 来源追踪，答案可信
- ✅ 易于扩展，技术先进

适用于课程教学、知识库问答、文档检索等多种场景。
