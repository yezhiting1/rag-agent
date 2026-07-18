<img width="1536" height="762" alt="2" src="https://github.com/user-attachments/assets/ec3ea483-aa3d-4706-adf6-f76755657e4f" />






# RAG 智能体
基于 EdgeOne Makers 构建的检索增强对话智能体：依托本地 PDF 知识库回答问题，回复附带文献引用，通过 SSE 流式输出内容。后端采用 Python 版 OpenAI Agents SDK 开发。
技术框架：OpenAI Agents SDK · 应用分类：文件处理・开发语言：Python

[![Deploy to EdgeOne Makers](https://cdnstatic.tencentcs.com/edgeone/pages/deploy.svg)](https://edgeone.ai/makers/new?template=rag-agent&from=within&fromAgent=1&agentLang=python)

<!-- ![preview](./assets/preview.png)  TODO: confirm -->

## 项目概述

一套可直接投入企业使用的完整 RAG 模板：只需将 PDF 文件放入`public/prepare-rag/files/`, 执行一条脚本即可生成知识库；智能体回答内容均附带精确到文档页码的引用来源。
检索层基于本地文件系统构建，无需部署向量数据库、额外中间服务，代码逻辑清晰易读。你只需替换内置加载器，整套检索生成链路均可无缝复用。

1-回复附带文献引用：所有回答内容均可通过 search_document、fetch_pages 工具溯源至对应文档与页码
2-流式输出 + 工具执行可视化：前端界面实时推送 tool-input-available/tool-output-available 事件，用户可直观查看智能体正在读取哪些参考资料
3-文件系统知识库：prepare_rag_data.py 解析 PDF 并输出至 agents/_data/{文档ID}/pages/{页码}.txt；请求阶段由 _loader.py 安全读取文件，内置路径穿越防护
4-会话持久记忆：通过 context.store.openai_session(对话ID) 在单次对话内保存多轮上下文；无状态云函数 /history 可在页面刷新后恢复完整聊天记录
5-即时终止响应：调用 /stop 接口执行 context.utils.abort_active_run()，可在流式输出中途中断大模型调用

## 环境变量配置

| Variable | Required | Description |
|----------|----------|-------------|
| `AI_GATEWAY_API_KEY` | Yes | Model gateway API key. Use your Makers Models API Key, or any OpenAI-compatible provider key. |
| `AI_GATEWAY_BASE_URL` | Yes | Gateway base URL. For Makers Models, use `https://ai-gateway.edgeone.link/v1`. |
| `AI_GATEWAY_MODEL` | No | Model ID. Defaults to `@makers/deepseek-v4-flash` (a free built-in model). |

本模板完全兼容 OpenAI 接口标准，可对接 EdgeOne Makers Models 或其他任意兼容服务商。

### 获取 AI_GATEWAY_API_KEY 步骤

1. 打开 [Makers Console](https://edgeone.ai/makers/new?s_url=https://console.tencentcloud.com/edgeone/makers).
2. 登录账号并开通 Makers 功能
3. 进入 Makers → 模型服务 → API 密钥，创建专属密钥
4. 将生成密钥填入环境变量 AI_GATEWAY_API_KEY

内置 @makers/deepseek-v4-flash 为免费模型，设有调用额度上限，适合原型验证；生产环境可接入自有付费大模型（自带密钥接入模式）。

## 本地开发环境
前置依赖：Node.js ≥ 18、Python ≥ 3.10、EdgeOne 命令行工具（安装命令：npm i -g edgeone）

```bash
npm install
pip install -r agents/requirements.txt
pip install -r public/prepare-rag/requirements.txt
cp .env.example .env       # then fill in AI_GATEWAY_API_KEY / AI_GATEWAY_BASE_URL

# Drop PDFs into public/prepare-rag/files/, then build the knowledge base
npm run prepare-rag

edgeone makers dev
```

Local agent metrics & traces are exposed at `http://localhost:8080/agent-metrics`.

## Project Structure

```text
rag-agent/
├── agents/                          # Stateful EdgeOne Makers Agent Functions (Python)
│   ├── chat/index.py               # POST /chat — streaming RAG chat
│   ├── chat/_stream.py             # SSE streaming utilities (private)
│   ├── stop/index.py               # POST /stop — abort active agent run
│   ├── rag-stats/index.py          # POST /rag-stats — knowledge base stats
│   ├── _agent.py                   # RAG Agent definition (private)
│   ├── _tools.py                   # search_document, fetch_pages tools (private)
│   ├── _loader.py                  # Filesystem knowledge base reader (private)
│   ├── _model.py                   # LLM configuration (private)
│   ├── _data/                      # Generated knowledge base (gitignored)
│   └── requirements.txt            # Python agent dependencies
├── cloud-functions/                 # Stateless EdgeOne Makers Python cloud functions
│   ├── history/index.py            # POST /history — load conversation messages
│   └── _logger.py                  # Logger utility
├── public/prepare-rag/              # PDF → structured text pipeline
│   ├── prepare_rag_data.py
│   ├── requirements.txt
│   └── files/                      # Drop your source PDFs here
├── src/                             # React + Vite frontend
│   ├── App.tsx                     # Root component
│   ├── api.ts                      # SSE stream client
│   └── components/
│       ├── RagChat.tsx             # Chat UI with streaming + tool visibility
│       ├── CitationCard.tsx        # Source citation display
│       └── KnowledgeBaseSummary.tsx
├── package.json
├── edgeone.json                     # framework=openai-agents-sdk, agents.timeout=300, sandbox.timeout=300
└── vite.config.ts
```

> Files prefixed with `_` are private modules — not exposed as public routes.

## How It Works

`agents/` runs in **conversation mode**: requests carrying the same `Markers-Conversation-Id` HTTP header are sticky-routed to the same agent instance, sharing the same in-memory state and the same EdgeOne sandbox. That stickiness is what lets `/chat` (the SSE stream) and `/stop` (the abort) reach the same running task. `/stop` deliberately receives the conversation id in the request body — never in the header — so the cancel signal doesn't collide with the live SSE stream.

End-to-end:

1. **Build the knowledge base (offline)** — `prepare_rag_data.py` reads PDFs from `public/prepare-rag/files/` and writes `agents/_data/{docId}/meta.json` + `pages/{n}.txt` (plus an optional `structure.json` page-tree). `agents/_data/index.json` is the document manifest.
2. **Request entry** — `POST /chat` (handled by `agents/chat/index.py`) pulls history via `context.store.openai_session(conversation_id)` and starts an OpenAI Agents SDK run for the `_agent.py` agent definition.
3. **LLM ↔ tools loop** — the agent has access to two tools defined in `_tools.py`:
   - `search_document(query)` — retrieves candidate pages from the local knowledge base
   - `fetch_pages(doc_id, pages)` — reads exact page text for citation
   The agent runs up to 6 turns; `_loader.py` enforces path-traversal-safe filesystem reads under `Path(__file__).parent / "_data"`.
4. **Streaming** — the handler emits SSE events `start`, `text-start`, `text-delta`, `text-end`, `tool-input-available`, `tool-output-available`, `finish`, `error`. The UI's `useAgentStream` reducer turns those into chat bubbles + citation cards.
5. **Stats / history / stop** — `POST /rag-stats` (in `agents/`) returns knowledge-base metadata; `POST /history` (in `cloud-functions/`) reads `context.agent.store.get_messages()` to rehydrate after a refresh; `POST /stop` cancels the live run.

Sandbox credentials are injected by the runtime — no local sandbox config is needed. Per `edgeone.json`, both the agent and its sandbox have a 300-second timeout (`agents.timeout`, `agents.sandbox.timeout`).

The bundled sample knowledge base includes:

- **EdgeOne-Pages-Platform-Guide.pdf** — platform architecture, `context.store`, SSE streaming, deployment.
- **Building-RAG-Applications.pdf** — RAG patterns, retrieval strategies, citations, evaluation.

## Resources

- [EdgeOne Makers Agents — Documentation](https://pages.edgeone.ai/document/agents)
- [EdgeOne Makers — Quick Start](https://pages.edgeone.ai/document/agents-quick-start)
- [Makers Models](https://pages.edgeone.ai/document/models)

## License
MIT.



