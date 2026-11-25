# cook-rag

基于 **RAG（检索增强生成）** 的中餐菜谱知识库与智能问答系统。项目收录 **323 道**家常中餐菜谱（全部为 Markdown 格式，统一包含「必备原料和工具 / 计算 / 操作 / 附加内容」结构），通过向量化与混合检索，让系统能够根据用户的自然语言问题快速定位相关菜谱，并生成分步骤的详细做法、烹饪清单或菜名列表，解决"今天吃什么"的世纪难题。

支持两种使用方式：命令行交互式问答（`python main.py`）与 FastAPI HTTP 服务（`api_server.py`）。

## 功能特性

- **智能问答**：支持「今天吃什么 / 找一道菜的做法 / 按分类或难度筛选」等自然语言场景
- **查询路由**：自动识别问题类型（`list` 列表查询 / `detail` 详细步骤 / 一般咨询），分别采用菜名列表、分步指导或基础回答三种生成模式
- **查询改写**：对非列表类问题进行智能重写，提升检索命中率
- **混合检索 + RRF 重排**：向量语义检索（Pinecone + BGE-M3）与 BM25 关键词检索结合，使用 RRF（k=60）融合排序，提升菜谱召回质量
- **元数据过滤**：自动从问题中提取分类（荤菜、素菜、水产、汤品、主食、甜品、饮品、调料、预制菜等）与难度（非常困难 / 困难 / 中等 / 简单 / 非常简单）关键词，并应用于检索过滤
- **父子文档机制**：按 Markdown 标题层级分块检索子块，回答时回溯完整父文档，兼顾检索精度与上下文完整性
- **流式输出**：命令行与 API 均支持流式（SSE）返回
- **模块化架构**：数据准备、索引构建、检索优化、生成集成四个模块独立拆分，便于二次开发

## 目录结构

```
cook-rag/
├── .env                          # 环境变量（数据路径、嵌入模型、Pinecone 等，勿提交版本库）
├── api_server.py                 # FastAPI 后端服务入口，暴露知识库与问答接口
├── main.py                       # RAG 系统主类 RecipeRAGSystem + 命令行交互式问答
├── rag_modules/                  # RAG 核心模块
│   ├── __init__.py
│   ├── data_preparation.py       # 菜谱加载、Markdown 分块、元数据增强、统计、父文档回溯
│   ├── index_construction.py     # 向量索引构建（HuggingFace 嵌入 + Pinecone Serverless）
│   ├── retrieval_optimization.py # 混合检索、BM25、RRF 重排、元数据过滤检索
│   └── generation_integration.py # LLM 集成（Ollama）、查询路由/改写、多模式回答生成
└── cook/                         # 菜谱数据（Markdown + 配图）
    └── dishes/                   # 按分类分目录
        ├── aquatic               # 水产（鱼类、虾类、贝类等）
        ├── breakfast             # 早餐
        ├── condiment             # 调料
        ├── dessert               # 甜品
        ├── drink                 # 饮品
        ├── meat_dish             # 荤菜
        ├── vegetable_dish        # 素菜
        ├── soup                  # 汤品
        ├── staple                # 主食
        ├── semi-finished         # 预制菜
        └── template              # 示例菜（菜谱模板，可作编写参考）
```

## 菜谱数据说明

菜谱数据位于 `cook/dishes/` 下，共 323 篇文档，各分类数量如下：

| 分类 | 目录 | 数量 | 分类 | 目录 | 数量 |
| --- | --- | --- | --- | --- | --- |
| 荤菜 | `meat_dish` | 97 | 水产 | `aquatic` | 24 |
| 素菜 | `vegetable_dish` | 54 | 早餐 | `breakfast` | 22 |
| 主食 | `staple` | 47 | 饮品 | `drink` | 21 |
| 汤品 | `soup` | 21 | 甜品 | `dessert` | 17 |
| 预制菜 | `semi-finished` | 10 | 调料 | `condiment` | 9 |
| 示例菜 | `template` | 1 | | | |

每个菜谱为一个 `.md` 文件，文件名即菜名，标题为「`{菜名}的做法`」。推荐遵循以下统一结构（参考 [示例菜](cook/dishes/template/示例菜/示例菜.md)）：

1. **标题**：`# {菜名}的做法`；
2. **简介**：简要说明菜的特点、营养价值、预估难度与预计制作时长（含星级标注）；
3. **必备原料和工具**：列出必需食材与工具，可标注推荐品牌与可选原料；
4. **计算**：给出原料用量（质量/体积单位为毫升等统一单位）、每种配方的数量参考；
5. **操作**：详细描述全部制作流程，步骤需精确、便于复现；
6. **附加内容**：补充注意事项、安全须知、参考资料、相关图片等。

> 编写菜谱时注意：中文与英文或数字之间必须有且仅有一个空格；标题与正文之间必须有且仅有一个空行；不使用「适量」「适当」等不精确词汇；步骤中如需指代部分食材，必须明确说明。

分类与难度由 `rag_modules/data_preparation.py` 通过目录与内容正则自动识别，新增菜谱时只需放入对应分类目录即可自动纳入知识库。

## 技术栈

| 组件 | 技术选型 |
| --- | --- |
| RAG 编排框架 | LangChain（`langchain` / `langchain-community` / `langchain-text-splitters`） |
| 向量数据库 | Pinecone Serverless（AWS `us-east-1`，cosine 度量） |
| 嵌入模型 | BAAI/bge-m3（`HuggingFaceEmbeddings` 本地加载，1024 维，CUDA 推理，向量归一化） |
| 生成模型 | Ollama 本地部署 `qwen3.5:4b`（`init_chat_model` 接入） |
| 关键词检索 | BM25Retriever + RRF 重排 |
| Web 服务 | FastAPI + Uvicorn + Pydantic |
| 配置管理 | python-dotenv（`.env`） |

## 环境要求

- **Python 3.12+**（代码使用了 f-string 嵌套同类引号语法，3.12 以下无法运行）
- **NVIDIA GPU + CUDA**（嵌入模型以 `device: 'cuda'` 运行；无 GPU 需自行修改 `rag_modules/index_construction.py` 为 `cpu`）
- 本地已安装并运行 [Ollama](https://ollama.com/)，且已拉取生成模型：

  ```bash
  ollama pull qwen3.5:4b
  ```

- 有效的 [Pinecone](https://www.pinecone.io/) 账号及 API Key（免费 Serverless 索引即可）

## 快速开始

### 1. 安装依赖

项目未附 `requirements.txt`，可按下列清单安装：

```bash
pip install python-dotenv pinecone langchain langchain-community langchain-pinecone \
    langchain-huggingface langchain-ollama langchain-text-splitters \
    rank-bm25 fastapi uvicorn pydantic torch sentence-transformers
```

### 2. 配置环境变量

编辑项目根目录 `.env`：

```dotenv
# 菜谱数据根目录（包含 dishes/ 目录的 cook 目录，系统会递归加载其下所有 .md 菜谱）
DATA_PATH=<项目路径>\cook

# 嵌入模型（本地 BGE-M3 模型目录，需可访问）
EMBEDDING_MODEL_NAME=<模型路径>\BAAI--bge-m3
EMBEDDING_DIM=1024

# Pinecone 数据库
PINECONE_API_KEY=<your pinecone key>
PINECONE_INDEX_NAME=how-to-cook
```

> 首次运行时若索引不存在，系统会自动创建 Serverless 索引（维度需与 `EMBEDDING_DIM` 一致）。

### 3. 命令行交互问答

```bash
python main.py
```

启动后自动初始化系统并构建/加载知识库，随后进入交互式问答，输入 `退出` / `q` / `exit` 结束，每次提问可选择是否启用流式输出。

### 4. 启动 HTTP API 服务

```bash
uvicorn api_server:app --host 0.0.0.0 --port 8000
```

启动后服务地址为 `http://localhost:8000/`，交互式 API 文档见 `http://localhost:8000/docs`。使用前需先构建知识库，再调用问答接口：

```bash
# 构建知识库（初始化系统 + 加载菜谱文档 + 构建向量索引）
curl -X POST "http://localhost:8000/api/build-knowledge-base"

# 普通模式问答
curl -X POST "http://localhost:8000/api/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "可乐鸡翅的做法", "stream": false, "top_k": 3}'

# 流式模式问答（SSE）
curl -N -X POST "http://localhost:8000/api/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "推荐一份简单的早餐", "stream": true, "top_k": 3}'

# 查询知识库统计（菜品分类、难度分布等）
curl "http://localhost:8000/api/statistics"
```

## 接口说明

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/` | 服务根路径，返回服务基本信息 |
| GET | `/api/health` | 健康检查，返回服务状态 |
| POST | `/api/build-knowledge-base` | 构建知识库（初始化系统 + 加载文档 + 构建向量索引），返回构建统计信息 |
| POST | `/api/ask` | 智能问答，请求体 `{"question": "...", "stream": false, "top_k": 3}`；普通模式 JSON 返回，流式模式 SSE 返回 |
| GET | `/api/statistics` | 获取知识库数据统计（文档数、文本块数、分类、难度分布等） |
| POST | `/api/clear-index` | 清空向量索引（删除索引下所有向量数据，慎用） |

## 系统集成架构

系统采用分层模块化设计，问答流程如下：

```
用户问题
   │
   ├─ 1. 查询路由（list / detail / 一般）        generation_integration.query_router
   ├─ 2. 智能查询改写（list 类保持原样）          generation_integration.query_rewrite
   ├─ 3. 提取元数据过滤条件（分类、难度）         main._extract_filters_from_query
   ├─ 4. 混合检索：向量检索 + BM25 → RRF 重排     retrieval_optimization.hybrid_search
   ├─ 5. 子块回溯父文档（完整菜谱）               data_preparation.get_parent_documents
   └─ 6. 按路由类型生成回答（列表/分步/基础，支持流式）
```

各模块职责：

1. **数据准备（`DataPreparationModule`）**：递归加载菜谱目录下所有 `.md` 文件，按 Markdown 标题层级分块，增强文档元数据（来源路径、分类、菜品名、难度、doc_type），并统计文档与分类信息。
2. **索引构建（`IndexConstructionModule`）**：使用 BGE-M3 嵌入模型对菜谱分块向量化，构建 Pinecone Serverless 向量索引，支持索引清空与重建。
3. **检索优化（`RetrievalOptimizationModule`）**：提供混合检索（向量 + BM25 + RRF 重排）与元数据过滤检索能力，自动依据问题中的分类/难度关键词过滤相关文档块。
4. **生成集成（`GenerationIntegrationModule`）**：通过 Ollama 接入本地 LLM，根据查询路由类型生成回答——列表查询直接返回菜名列表，详细查询返回分步骤做法，一般查询返回基础回答。

`main.py` 中的 `RecipeRAGSystem` 将上述模块组合为完整系统，支持交互式问答与知识库构建等核心操作。

## 注意事项

- 嵌入维度（1024）与 Pinecone 索引维度必须一致，更换嵌入模型后需通过 `/api/clear-index` 清空索引重建。
- 菜谱数据是本项目的核心资产，新增菜谱请遵循上方「菜谱数据说明」的规范，放入对应分类目录即可自动纳入知识库。
- 菜谱内容仅供家庭烹饪参考，涉及食品安全（如生食、腌制、加热时间）请以权威来源为准。
