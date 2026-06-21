# 商品竞品分析 Agent 协作工作台

这是一个独立 Web App。当前 Demo 场景为“空气炸锅商品竞品分析”，用于展示品牌产品经理、电商运营和商品规划人员如何通过 5 个业务 Agent + 贯穿式 QA Guardrail 完成 SKU 竞品分析。

系统保留 FastAPI 后端、Agent 脚本、统一 LLM/Search/Fetch 客户端、前端工作台和 demo 产物机制。质量控制已重构为阶段 QA 验收和轻量最终交付 Gate。当前版本不包含真实爬虫、实时价格监控、销量预测或平台账号接入。

## 项目结构

```text
frontend/index.html
backend/server.py
backend/clients/llm_client.py
backend/clients/search_client.py
backend/clients/fetch_client.py
backend/models/schemas.py
backend/services/run_manager.py
backend/services/artifact_loader.py
backend/services/version_store.py
scripts/planning_agent.py
scripts/collecting_agent.py
scripts/structuring_agent.py
scripts/comparing_agent.py
scripts/writing_agent.py
scripts/quality_agent.py
scripts/orchestrator.py
data/demo/
output/
logs/
```

## 当前 Demo 覆盖

- 任务输入：自然语言任务、目标品类、报告用途、目标产品定位、决策问题、候选 SKU、分析维度、来源范围。
- 分析计划：推荐竞品 SKU、推荐维度、来源范围、缺失信息和可调整项。
- 协作执行：5 个业务 Agent 状态、阶段 QA 状态、SKU 字段整理、价格口径冲突和局部回退模拟。
- 对比结果：商品对比表、关键发现、机会点象限、证据链抽屉。
- QA Guardrail：阶段 PASS / WARNING / BLOCKED、统一 QA 详情抽屉、责任 Agent、修复动作和局部重跑范围。
- 最终报告：1-10 节商品规划输入文档，包含 Roadmap 和“不建议做什么”。

## 安装依赖

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## 配置环境变量

```bash
cp .env.example .env
```

要真实运行全流程，请编辑 `.env`，填入两个 Key：LLM 用于结构化、分析和报告生成；Tavily 用于从公开网页检索证据。默认示例已经设置为严格 live 模式，缺少任一 Key 时，创建任务会返回清晰的 503 配置错误，而不是伪装成真实结果。

```text
PIPELINE_MODE=live
OPENAI_API_KEY=...
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4o-mini
TAVILY_API_KEY=...
```

`OPENAI_BASE_URL` 与 `OPENAI_MODEL` 支持 OpenAI-compatible 服务。仅在课堂展示时，将 `PIPELINE_MODE=demo`；若希望未配置时自动使用资料包，则设为 `auto`。生产前还应把自己的前端域名加入 `CORS_ORIGINS`。

配置后先执行预检（不会发送请求，也不会显示密钥）：

```bash
python3 -m scripts.preflight
```

## 启动后端

```bash
uvicorn backend.server:app --reload --host 127.0.0.1 --port 8000
```

然后打开：

```text
http://127.0.0.1:8000/
```

可用以下接口确认服务已识别为 live（返回中只会显示 true/false，不会泄露 Key）：

```bash
curl http://127.0.0.1:8000/api/health
```

真实运行可通过工作台接入，或调用既有 API：

```bash
curl -X POST http://127.0.0.1:8000/api/runs \
  -H 'Content-Type: application/json' \
  -d '{
    "industry":"空气炸锅商品竞品分析",
    "target_companies":["美的空气炸锅","九阳空气炸锅","苏泊尔空气炸锅"],
    "dimensions":["容量与规格","核心功能","价格与促销","用户差评与痛点"],
    "report_usage":"新品立项",
    "source_scope":["电商商品页","品牌官网","价格与促销","用户评价"],
    "price_range":"300-500 元"
  }'
```

也可以直接打开：

```text
frontend/index.html
```

## 运行分析

前端工作台会引导用户完成：

```text
任务输入 -> 生成分析计划 -> 确认计划 -> 采集与结构化 -> 对比分析 -> 报告生成 -> 最终交付 Gate
```

后端 API 仍保留：

- `POST /api/runs`：启动一次多 Agent 任务
- `GET /api/runs/{run_id}/status`：读取运行状态
- `GET /api/runs/{run_id}/logs`：读取流水日志
- `GET /api/runs/{run_id}/agent/{agent_id}`：读取某个 Agent 产物
- `GET /api/runs/{run_id}/result`：读取最终报告和关键产物
- `GET /api/runs/{run_id}/artifacts`：列出产物文件
- `POST /api/runs/{run_id}/feedback`：记录用户修改意见
- `POST /api/runs/{run_id}/rerun`：从指定 Agent 生成下一版

## Demo 产物

`data/demo/` 中的资料已经切换为空气炸锅商品竞品分析：

- `planning_result.json`：SKU 范围、维度、来源和待确认问题
- `collected_sources.json`：商品页、价格、评价和结构化来源库
- `structured_schema_table.json`：SKU Schema 表和字段风险
- `comparison_analysis.json`：对比矩阵、关键发现、机会点和证据链
- `report_draft.md`：报告初稿
- `report_metadata.json`：章节、证据链和风险披露
- `quality_report.json`：历史质检样例，当前前端已改为阶段 QA Guardrail 展示
- `competitive_report_final.md`：最终商品竞品分析报告
- `workflow_state.json`：本阶段新增的共享任务状态、局部重跑规则和版本结构样例

## Agent 职责

- Planning Agent：理解商品分析任务，收敛 SKU 范围和分析维度。
- Collecting Agent：收集公开商品页、价格促销、问答和评价资料。
- Structuring Agent：整理统一 SKU Schema，标记缺失字段和冲突字段。
- Comparing Agent：生成商品对比矩阵、关键发现和机会点优先级。
- Writing Agent：生成带证据链和风险边界的报告初稿。
- QA Guardrail：不作为左侧独立业务 Agent 节点；在每个业务阶段后自动验收，返回 PASS / WARNING / BLOCK，输出透明化"规则检验明细"（checked_rules），支撑"主界面显示报错规则、侧边抽屉折叠通过规则"交互。支持内嵌式三动作决策：采纳 AI 修复并重跑、忽略警告强制放行、输入自定义修改意见。

## 本阶段协作机制

- 总控逻辑在前端工作台中模拟，负责调度当前 Agent、判断是否需要用户确认、计算局部重跑范围并生成报告版本。
- SKU 冲突回路：结构化 Agent 发现苏泊尔规格 / 套餐冲突，用户点击“重新确认商品规格”后，仅重跑采集、结构化、相关对比和报告第 4、9 章，并生成 V2。
- 证据不足回路：分析 QA Guardrail 发现“清洁困难机会点”评价证据不足，用户点击“补充评价证据”后，仅重跑评价处理、相关机会点、报告第 6、9 章和最终交付 Gate，并生成 V3。
- 版本机制保留 V1、V2、V3 的生成时间、修改原因、变更章节和差异摘要。

## 当前限制

- Demo 链接使用 `example.com`，不可作为真实引用。
- 不执行真实平台爬取、登录、价格监控或销量预测。
- 用户补充资料和局部重跑在前端可模拟状态，接入真实数据源后可替换为后端流程。

## Batch 2：任务规划与信息采集闭环

本次交付统一了前两个业务 Agent 的输入、输出和交接合同：

- 首页字段统一映射到任务规划数据，不再由 `demoTask`、`taskState`、`editableTask` 各自维护互相冲突的值。
- 任务规划结果区分“用户提供”与“Agent 补充”，并在用户完成任务、竞品、维度和来源确认后生成 `contract_version: 2.0` 的采集交接合同。
- 信息采集严格读取已确认的 SKU、分析维度、来源计划和附件摘要，展示规划修订号和范围一致性。
- 每个 SKU 明确标记为已采集、带缺口、缺失、失败或冲突；来源记录绑定平台、采集时间、可信度和支撑字段。
- 采集阶段的普通价格/规格冲突作为结构化阶段的待处理输入，不再错误地等同于采集失败；只有 SKU 身份不明或完全缺失来源时才阻断。
- Demo 中可对非阻断信息缺口执行预设补充，生成来源 `S006` 并将采集 QA 从 WARNING 更新为 PASS。
- 首页附件仍为会话内元数据和本地预览，当前不会上传或解析文件内容。

## Batch 3.0：信息与视觉收口

本批次在字段级交互开发前统一了页面信息层级和质检语义：

- 信息采集主页面不再默认展示任务交接大卡，交接合同仍保留在工作流状态中。
- 五个阶段统一使用“待质检 / 质检中 / 已通过 / 带风险通过 / 已阻断”状态。
- 任务规划在三步确认前不生成质检结论；未来阶段不会在 QA 总览中提前显示结果。
- 采集阶段区分无阻断警告与阻断异常；结构化阶段将上游风险与本阶段新增问题分开。
- 全站收敛为冷灰蓝背景、白色卡片、单一主蓝色和三类语义状态色。
- Agent 不再使用五种身份色；状态图标改为轻量线性 SVG，不使用 Emoji。
- 报告章末内容改为决策性总结，最终交付入口统一为“查看风险与分析边界”。

新增前端覆盖文件：

```text
frontend/batch3_0.css
frontend/batch3_0.js
```

## Batch 3A Lite：关键结构化字段面板

本批次遵循“只新增、不删除或折叠原有内容”的原则：

- 保留原有商品卡片堆叠；
- 在商品卡片之后新增“关键结构化字段”可点击表格；
- 原有四类量表继续完整展开；
- 点击待确认、冲突或规划字段后，右侧栏在原有内容上方新增字段说明和处理入口；
- 字段处理只写入新增面板与现有版本记录，不改动原有商品卡片和四类量表。

新增覆盖文件：

```text
frontend/batch3a_lite.css
frontend/batch3a_lite.js
```

## Batch 3A.1：排版与模拟重跑热修复

- 结构化阶段右侧上下文改为覆盖式抽屉，不再压缩中央表格。
- 新增结构化表由 9 列压缩为 6 列摘要表，原字段和点击能力保留。
- 信息采集、结构化和对比分析的 QA 决策界面统一。
- 对比分析“采纳 AI 建议”现在会真实更新 Demo 证据、参照竞品权重和 QA 状态，并停留在对比分析阶段等待用户确认。

新增覆盖文件：

```text
frontend/batch3a1.css
frontend/batch3a1.js
```

## Batch 3A.2：低操作量人机协作

- QA 异常默认收束为“AI 推荐摘要 + 一键应用”，不再要求用户逐项审批全部 Warning。
- Blocker 保留人工确认，但结构化根问题可一次应用统一推荐方案并模拟重跑。
- “查看并调整”继续保留规则级采纳、保留风险和自定义能力。
- 结构化字段点击后在单元格旁显示快捷操作浮层，只有查看完整依据时才打开右侧上下文抽屉。
- 信息采集、结构化处理和对比分析的一键推荐均会真实更新 Demo 状态与 QA 结果。

新增覆盖文件：

```text
frontend/batch3a2.css
frontend/batch3a2.js
```

## Batch 3A.2｜低操作量 Human-in-the-loop

本版本将采集、结构化与对比分析中的多项异常默认汇总为一键 AI 推荐方案；高级用户可按需展开逐项调整。结构化关键字段支持在单元格旁直接处理，右侧抽屉仅承担详细依据与来源说明。六列结构化表已取消内部嵌套框线和固定行高，降低留白并保持表格对齐。

## Batch 3B Lite

新增评分解释、数据口径标识与机会点轻量调整。默认路径不增加人工审批；评分与机会点仅在用户需要时展开解释或覆盖 AI 推荐。详见 `BATCH3B_LITE_CHANGELOG.md`。
