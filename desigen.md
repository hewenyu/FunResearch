好的，遵照您的指示。

这份需求文档整合了我们讨论过的所有核心概念：多模态输入、智能代理协作、多轮迭代深化、信源权重评估，以及最终采用的“量化评分质量门禁”机制。这可以作为您使用 Python + PostgreSQL + Redis 技术栈进行开发的一份完整、详细的蓝图。

---

### **项目需求文档：个人智能研究助理 (A.I.R.A.)**

**版本：** 2.0
**日期：** 2025年7月29日
**状态：** 基线 (Baseline)

---

### **1. 项目愿景与目标 (Vision & Goals)**

#### **1.1. 项目愿景**
构建一个高级的、具备批判性思维的个人智能研究助理（A.I.R.A.）。它旨在自动化并深化复杂课题的研究过程，通过模拟人类专家的迭代式探究、信息辨析和综合分析，产出远超传统AI助手深度的、可信的、可溯源的研究报告。

#### **1.2. 核心目标**
*   **流程自动化:** 实现从模糊问题到深度报告的全流程自动化。
*   **质量可量化:** 通过引入评分机制，建立一套客观、可控的报告质量标准。
*   **深度与可信度:** 结合多轮深化与信源加权，确保产出内容的深度与可信度。
*   **智能交互:** 实现主动的需求澄清，将用户的模糊意图转化为明确的执行任务。

---

### **2. 系统架构与技术栈 (Architecture & Tech Stack)**

*   **应用后端:** **Python 3.10+** with **FastAPI** (用于高性能异步IO)。
*   **持久化数据库:** **PostgreSQL 15+** (用于存储项目、报告、信源规则等结构化数据)。
*   **缓存与任务队列:** **Redis 7+** (用于会话管理、状态跟踪和异步任务调度)。
*   **核心AI引擎:** **Google Gemini API** (混合使用 Gemini 2.5 Pro 和 Flash)。

---

### **3. 功能需求 (Functional Requirements)**

#### **FR-1: 研究项目管理**
*   **FR-1.1:** 用户能通过Web界面创建一个新的“研究项目 (Research Project)”，并提供一个初始的研究主题或问题。
*   **FR-1.2:** 在项目的任何阶段，用户均可上传多种格式的文件（PDF, DOCX, TXT, CSV）和图片（JPG, PNG）作为补充上下文。系统需能自动提取文本内容。
*   **FR-1.3:** 每个研究项目在数据库中都有一个唯一的ID，并关联其所有相关数据（提示、报告、信源等）。

#### **FR-2: 智能代理工作流 (Agent-based Workflow)**
系统核心由一个 **Orchestrator（调度中心）** 模块驱动，该模块负责调用和协调以下智能代理。

*   **FR-2.1: Clarifier Agent (需求澄清代理)**
    *   **触发:** 新项目创建后。
    *   **模型:** Gemini 2.5 Flash。
    *   **逻辑:** 接收初始请求，分析其模糊性，生成澄清问题返回给用户。收集用户回答后，整合成一份“明确任务描述 (Clarified Prompt)”。

*   **FR-2.2: Planner Agent (规划代理)**
    *   **触发:** 需求澄清或新一轮深化开始时。
    *   **模型:** Gemini 2.5 Pro。
    *   **逻辑:** 接收任务描述，将其分解为一份详细的、步骤化的JSON执行计划。

*   **FR-2.3: Searcher Agent (加权搜集代理)**
    *   **触发:** Planner生成包含`search`的步骤后。
    *   **模型:** Gemini 2.5 Pro/Flash with Google Search Tool。
    *   **逻辑:**
        1.  并行执行搜索查询。
        2.  抓取并清洗内容。
        3.  调用内部“信源评估模块”，根据PostgreSQL中的规则为每个来源评定**权重分**和**层级 (Tier)**。
        4.  将加权后的信息（内容+元数据）存入Redis，以供后续使用。

*   **FR-2.4: Writer Agent (写作代理)**
    *   **触发:** 信息搜集完毕后。
    *   **模型:** Gemini 2.5 Pro。
    *   **逻辑:** 接收所有加权信源和写作指令，综合信息并撰写报告草稿。在写作时，应被明确指示优先采纳高权重信源的观点。输出为Markdown格式，并包含引用标记。

*   **FR-2.5: Critic Agent (批判与评分代理)**
    *   **触发:** Writer生成报告草稿后。
    *   **模型:** Gemini 2.5 Pro。
    *   **逻辑:**
        1.  接收报告草稿和评分标准。
        2.  **输出一份JSON格式的评分报告**，包含对【证据强度(40分)、逻辑结构(20分)、分析深度(20分)、时效性(10分)、需求契合度(10分)】等维度的打分和评价理由。
        3.  在评分之外，输出一份具体的“改进建议列表”。

#### **FR-3: 迭代深化循环与质量门禁 (Iterative Loop & Quality Gate)**
*   **FR-3.1:** 系统需内置一个可配置的**质量门禁分数**（如 `QUALITY_THRESHOLD = 85`）。
*   **FR-3.2:** Orchestrator在收到Critic的评分后，计算总分。
*   **FR-3.3:** 若总分 **≥** 质量门禁，则循环终止，报告被标记为最终版。
*   **FR-3.4:** 若总分 **<** 质量门禁，Orchestrator将Critic的“改进建议”转化为新任务，启动新一轮的`Planner -> Searcher -> Writer -> Critic`循环。
*   **FR-3.5:** 系统需内置一个最大循环次数（如 `MAX_ITERATIONS = 5`）以防止无限循环。

#### **FR-4: 输出与呈现**
*   **FR-4.1:** 最终报告以格式化的Markdown在前端呈现。
*   **FR-4.2:** 报告中的所有引用标记都可交互。点击后，应展示来源URL、标题、以及系统评定的**信源层级和权重分**，以建立信任。

---

### **4. 非功能性需求 (Non-Functional Requirements)**

*   **NFR-1: 性能:** 所有耗时操作（API调用、网页抓取）必须通过Redis任务队列异步执行。前端需轮询任务状态并向用户提供实时反馈。
*   **NFR-2: 模块化与可扩展性:** 后端代码应高度模块化，每个Agent实现为独立的类。信源权重规则、评分维度和Prompt模板应易于配置和扩展，无需修改核心代码。
*   **NFR-3: 配置管理:** 所有敏感信息（API密钥）和可调参数（质量门禁分数、最大循环次数）必须通过环境变量或配置文件（`.env`）管理。
*   **NFR-4: 日志记录:** 需有详细的日志记录每个研究项目的每一步流程、每个Agent的输入输出、以及每次API调用的耗时和Token消耗，便于调试和优化。

---

### **5. 数据模型设计 (Data Model)**

#### **5.1. PostgreSQL Schema**

```sql
-- 存储信源权重规则，系统的知识基础
CREATE TABLE source_credibility_rules (
    id SERIAL PRIMARY KEY,
    domain_pattern VARCHAR(255) UNIQUE NOT NULL, -- e.g., '%.nature.com', '%.gov.cn'
    tier VARCHAR(50) NOT NULL, -- 'Tier 1', 'Tier 2', 'Tier 3', 'Tier 4'
    description TEXT
);

-- 存储研究项目的主信息
CREATE TABLE research_projects (
    id SERIAL PRIMARY KEY,
    topic VARCHAR(512) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'CLARIFYING', -- CLARIFYING, RUNNING, DONE, FAILED
    quality_threshold INTEGER NOT NULL DEFAULT 85,
    max_iterations INTEGER NOT NULL DEFAULT 5,
    final_report_id INTEGER, -- 指向最终版本的报告
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 存储项目的所有上下文，包括初始提示和用户回答
CREATE TABLE project_context (
    id SERIAL PRIMARY KEY,
    project_id INTEGER NOT NULL REFERENCES research_projects(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL, -- 'user', 'assistant'
    content TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 存储每个版本的报告及其质量评分
CREATE TABLE reports (
    id SERIAL PRIMARY KEY,
    project_id INTEGER NOT NULL REFERENCES research_projects(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    content_markdown TEXT,
    is_final BOOLEAN DEFAULT FALSE,
    total_score INTEGER,
    score_details JSONB, -- 存储Critic返回的详细评分JSON
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 存储报告引用的所有信源（物料）
CREATE TABLE artifacts (
    id SERIAL PRIMARY KEY,
    project_id INTEGER NOT NULL REFERENCES research_projects(id) ON DELETE CASCADE,
    source_url TEXT NOT NULL,
    source_title TEXT,
    credibility_tier VARCHAR(50),
    extracted_content TEXT,
    retrieved_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (project_id, source_url)
);
```

#### **5.2. Redis 用途**
*   **任务队列:** 使用Redis List作为Celery或RQ的Broker，队列名为`research_tasks`。
*   **状态跟踪:** 使用Redis Hash存储正在运行的项目的详细状态，`job:{project_id}` -> `{"status": "running", "current_step": "CriticAgentReview", "iteration": 2}`。
*   **缓存:** 缓存已抓取的网页内容，避免重复抓取。`cache:url:{url_hash}`。

---

### **6. 附录：智能代理职责定义 (Appendix: Agent Roles)**

| 代理名称 | 主要职责 | 核心模型 | 产出物 |
| :--- | :--- | :--- | :--- |
| **Orchestrator** | 流程控制中心，调用和协调所有代理，管理状态机。 | (N/A - Code Logic) | 驱动整个研究流程 |
| **Clarifier Agent** | 与用户对话，将模糊需求转化为清晰指令。 | Gemini 2.5 Flash | 明确的任务描述 |
| **Planner Agent** | 将复杂任务分解为可执行的步骤计划。 | Gemini 2.5 Pro | 结构化的JSON执行计划 |
| **Searcher Agent** | 执行搜索，抓取内容，并根据规则评估信源权重。 | Gemini Pro/Flash + Tools | 加权后的信息集合 |
| **Writer Agent** | 综合加权信息，撰写结构化的报告草稿。 | Gemini 2.5 Pro | 带引用的Markdown文本 |
| **Critic Agent** | 审查草稿，进行多维度量化评分，并提出改进建议。 | Gemini 2.5 Pro | 评分JSON和改进列表 |