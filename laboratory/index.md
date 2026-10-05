三层极简架构方案

一、整体结构

```
┌─────────────────────────────────────────────┐
│  L3 · 范畴层                                 │
│  不同方法论的本体如何共存、映射、切换          │
├─────────────────────────────────────────────┤
│  L2 · 本体层                                 │
│  定义”世界由什么构成“——实体、属性、关系       │
├─────────────────────────────────────────────┤
│  L1 · 主数据层                               │
│  存实际数据实例，可追溯、可审批、可版本化      │
└─────────────────────────────────────────────┘
         ▲
         │ 写入 / 读取
         │
┌─────────────────────────────────────────────┐
│  LLM 知识库（驱动引擎）                       │
│  非结构化数据 → 抽取 → 对齐本体 → 写入主数据  │
│  人类确认 → 审批通过 → 正式入库               │
└─────────────────────────────────────────────┘
```

数据流向：非结构化输入 → LLM 抽取 → 对齐本体 → 写入主数据（待审批）→ 人类确认 → 正式入库。

人类确认是 LLM 知识库与三层之间的唯一关卡。LLM 的产出默认是 draft 状态，人类审批后才变为 active。

二、三层详细说明

L1 · 主数据层

做什么：存实际的实体实例数据。每条记录有状态、版本、来源、审批人。

本地环境选型：SQLite（单文件，零运维，支持 JSON 字段）。

核心表结构（最小集）：

```sql
— 实体实例表
CREATE TABLE entity (
    id TEXT PRIMARY KEY,
    category_id TEXT NOT NULL,        — 属于哪个范畴
    entity_type TEXT NOT NULL,        — 属于哪个本体类
    data JSON NOT NULL,               — 实例数据（JSON）
    status TEXT DEFAULT ’draft‘,      — draft / active / retired
    version INTEGER DEFAULT 1,
    source TEXT,                      — 来源（LLM抽取 / 人工 / 系统）
    created_by TEXT,
    approved_by TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

— 关系表
CREATE TABLE relation (
    id TEXT PRIMARY KEY,
    from_entity TEXT REFERENCES entity(id),
    to_entity TEXT REFERENCES entity(id),
    relation_type TEXT NOT NULL,      — 本体中定义的关系名
    data JSON,
    status TEXT DEFAULT ’draft‘,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

— 变更请求表（人类审批的核心）
CREATE TABLE change_request (
    id TEXT PRIMARY KEY,
    entity_id TEXT,
    action TEXT NOT NULL,             — create / update / merge / retire
    proposed_data JSON,
    human_edit JSON,                  — 人类审批时的修正
    risk_level TEXT,                  — low / medium / high
    decision TEXT DEFAULT ’pending‘,  — pending / approved / rejected
    decided_by TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

L2 · 本体层

做什么：定义”世界由什么构成“。用 OWL 或 JSON Schema 描述实体、属性、关系、约束。

本地环境选型：Turtle/OWL 文件 + Apache Jena（本地嵌入式推理），或更轻量的 JSON Schema。

最小本体示例（自媒体场景）：

```turtle
@prefix : <http://example.org/mdm#> .

:Content a owl:Class ;
    rdfs:label ”内容“ .

:Account a owl:Class ;
    rdfs:label ”账号“ .

:Material a owl:Class ;
    rdfs:label ”素材“ .

:publishedOn a owl:ObjectProperty ;
    rdfs:domain :Content ;
    rdfs:range :Account .

:usesMaterial a owl:ObjectProperty ;
    rdfs:domain :Content ;
    rdfs:range :Material .

:title a owl:DatatypeProperty ;
    rdfs:domain :Content ;
    rdfs:range xsd:string .

:status a owl:DatatypeProperty ;
    rdfs:domain :Content ;
    rdfs:range xsd:string .
```

约束：用 SHACL 定义校验规则，写入主数据时自动检查。

L3 · 范畴层

做什么：管理不同方法论的本体如何共存。每个范畴是一个独立的本体，声明自己的本体承诺。

本地环境选型：SQLite 表 + JSON 配置文件。

核心表结构：

```sql
— 范畴注册表
CREATE TABLE category (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,               — 如”流程导向“”实体导向“
    methodology TEXT,                 — process / entity / event / capability
    ontology_file TEXT,               — 对应本体文件路径
    commitment TEXT,                  — 本体承诺声明
    status TEXT DEFAULT ’active‘,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

— 范畴间映射表（函子）
CREATE TABLE category_mapping (
    id TEXT PRIMARY KEY,
    source_category TEXT REFERENCES category(id),
    target_category TEXT REFERENCES category(id),
    mapping_type TEXT,                — equivalent / weak / perspective_diff
    mapping_rule TEXT,                — 映射表达式
    confidence REAL,
    approved_by TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

— 冲突记录表
CREATE TABLE conflict (
    id TEXT PRIMARY KEY,
    source_category TEXT,
    target_category TEXT,
    assertion_a TEXT,
    assertion_b TEXT,
    conflict_type TEXT,               — perspective_diff / true_conflict
    resolution TEXT,
    resolved_by TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

范畴层的作用：当元智能体需要整合不同方法论的结论时，查 category_mapping 做翻译；翻译不了时，查 conflict 看是否已有人类裁决；都没有时，生成一条 change_request 等人类确认。

三、LLM 知识库：驱动引擎

这是把非结构化数据”灌入“三层结构的引擎。

工作流程

```
非结构化输入（文档/对话/网页）
        │
        ▼
┌───────────────────┐
│  LLM 抽取          │  用本体作为 schema 约束，让 LLM 输出结构化 JSON
│  → 识别实体        │
│  → 识别属性        │
│  → 识别关系        │
│  → 匹配到本体类    │
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  本体校验          │  SHACL 校验：类型对不对、必填字段有没有
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  写入主数据        │  status = ’draft‘，生成 change_request
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  人类确认          │  审批通过 → status = ’active‘
│                   │  审批拒绝 → 丢弃或退回修改
│                   │  参数修正 → human_edit 记录修正内容
└───────────────────┘
```

本地技术选型

组件 选型 说明
本地 LLM Ollama + Qwen2.5 / Llama3 本地跑，无需联网
向量库 ChromaDB 或 SQLite + sqlite-vec 存非结构化原文的向量，做语义检索
抽取框架 LangChain 或 LlamaIndex 编排”检索→抽取→校验→写入“流程
本体校验 pySHACL Python 库，本地校验
推理 Apache Jena 或 owlready2 本地 OWL 推理

LLM 抽取的 Prompt 模板（核心）

```python
EXTRACT_PROMPT = ”“”
你是一个主数据抽取引擎。根据以下本体定义，从文本中抽取实体和关系。

## 本体定义
{ontology_ttl}

## 当前范畴
{category_name}: {category_commitment}

## 输入文本
{raw_text}

## 输出要求
返回 JSON 数组，每个元素格式：
{{
  “entity_type”: “本体中的类名”,
  “data”: {{“属性名”: “值”}},
  “relations”: [
    {{“type”: “关系名”, “target_entity_type”: “类名”, “target_data”: {{...}}}}
  ],
  “confidence”: 0.0-1.0,
  “source_span”: “原文片段”
}}

只输出 JSON，不要解释。
“”“
```

人类确认的最小实现

不需要复杂的审批引擎。一张 change_request 表 + 一个简单的 Web 页面即可：

· 页面列出所有 pending 的变更请求
· 每条显示：LLM 抽取的原始内容、本体校验结果、风险等级
· 人类可以：批准 / 拒绝 / 修正后批准
· 修正内容写入 human_edit 字段
· 批准后自动更新 entity.status = ’active‘

四、一次完整的数据流示例

场景：自媒体智能体从一篇草稿中抽取”内容“主数据。

Step 1 · LLM 抽取

输入草稿：

”这篇《AI时代的MDM》准备发在公众号上，用了封面图A和配图B。“

LLM 输出：

```json
[
  {
    ”entity_type“: ”Content“,
    ”data“: {”title“: ”AI时代的MDM“, ”status“: ”draft“},
    ”relations“: [
      {”type“: ”publishedOn“, ”target_entity_type“: ”Account“, ”target_data“: {”name“: ”公众号“}},
      {”type“: ”usesMaterial“, ”target_entity_type“: ”Material“, ”target_data“: {”name“: ”封面图A“}},
      {”type“: ”usesMaterial“, ”target_entity_type“: ”Material“, ”target_data“: {”name“: ”配图B“}}
    ],
    ”confidence“: 0.92,
    ”source_span“: ”这篇《AI时代的MDM》准备发在公众号上...“
  }
]
```

Step 2 · 本体校验

pySHACL 检查：Content 类必须有 title（有）、status 必须是枚举值（有）、publishedOn 的 range 必须是 Account（是）。通过。

Step 3 · 写入主数据

· entity 表插入一条 Content，status = ’draft‘
· entity 表插入/关联 Account（如果已存在则复用）
· entity 表插入/关联两个 Material
· relation 表插入三条关系
· change_request 表插入一条待审批记录

Step 4 · 人类确认

审批人看到这条请求，确认标题、账号、素材都正确，点击批准。

Step 5 · 正式入库

· entity.status 更新为 active
· change_request.decision 更新为 approved
· 写入审计日志

Step 6 · 范畴切换（如果需要）

如果元智能体要做流程分析，查 category_mapping，发现当前是”实体导向“范畴，需要映射到”流程导向“范畴。函子规则把”内容“映射为”发布活动“，把”素材“映射为”活动资源“。如果映射不存在，生成一条 change_request 等人类定义映射。

五、最小技术栈清单

层 组件 选型
L1 主数据 存储 SQLite
L2 本体 本体编辑 Protégé（设计时）
L2 本体 推理校验 Apache Jena / pySHACL
L3 范畴 存储 SQLite（与 L1 同库不同表）
LLM 知识库 本地 LLM Ollama + Qwen2.5
LLM 知识库 向量检索 ChromaDB
LLM 知识库 编排 LangChain
人类确认 Web UI FastAPI + 简单 HTML
事件通知 本地 SQLite trigger + 轮询

全部本地运行，无需 Docker、Kafka、Neo4j、DataHub。一台开发机即可。

六、与你之前方案的关系

之前的五层方案 现在的三层方案
L1 数据接入 + L2 事实源 合并为 L1 主数据
L3 本体语义 保留为 L2 本体
L4 元认知 保留为 L3 范畴
L5 应用消费 简化为人类确认 Web 页
Kafka + Debezium + DataHub + Neo4j 全部去掉，用 SQLite + Ollama + ChromaDB

核心保留：范畴注册、函子映射、冲突标注、人类确认。
核心砍掉：CDC 事件流、元数据管理平台、图数据库、微服务编排、多级审批引擎。

这样三层各司其职：主数据存事实，本体定结构，范畴管共存，LLM 做抽取，人类做确认。 一台机器，一个 SQLite 文件，跑通完整链路。