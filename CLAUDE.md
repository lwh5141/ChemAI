# CLAUDE.md

ChemAI（智辅化学）——面向高中化学教学的 AI 教学平台。三端（教师端 / 学生端 / 家长端），核心闭环：**OCR 批改 → 障碍诊断 → 自适应练习 → 间隔复习 → 学情分析**。差异化在于回答"**为什么错**"（三维障碍根因诊断），而非"谁错了"。

动手前先读：`CONTEXT.md`（领域术语）、`DESIGN.md`（前端视觉约束）、`docs/adr/`（架构决策）。设计文档在 `docs/设计文件/`，总体开发方案见 `5.AI Coding工程实践/AI coding自然语言版/00-总体开发方案.md`。

## 项目概览

| 项 | 内容 |
|---|---|
| 三端 | 教师端（桌面 ≥1280px，6 页）/ 学生端（移动 430×900，6 页）/ 家长端（移动 430×900，2 页）——共 14 页 |
| 四条数据链 | 组织链（学校→年级→班级→学生）、教学链（考试→题目→作答）、家校链（家长↔学生）、身份链（账户↔教师/学生/家长） |
| 部署形态 | 单校部署优先；桌面端 pywebview 打包三个 .exe，服务端可选 Docker |
| 规格管理 | `openspec/`（propose → apply → archive） |
| 决策记录 | `docs/adr/`——决策不记录等于没发生 |

## 技术栈

**后端**：FastAPI 0.109.2 + Uvicorn 0.27.1（原生 async / 自动 OpenAPI / SSE 原生支持）；SQLAlchemy 2.0.25 + Alembic 1.13.1，24 个模型以 33 号文档为权威口径；SQLite + WAL（开发）/ MySQL 可选（生产），三库分离（主库 / Agent 检查点 / 长期记忆）

**Agent 与数据**：LangGraph `create_react_agent` **单 Agent**（路由准确率 87%；v0 Pydantic AI 约 70%、v1 多 Agent 约 75%，故弃用多 Agent）；AsyncSqliteStore + AsyncSqliteSaver 做检查点；ChromaDB 0.4.22 + DashScope text-embedding-v3（1024 维）

**LLM 路由**（主链路三级回退 + 视觉补充）

| 优先级 | Provider | 定位 |
|---|---|---|
| 主链路 | qwen-turbo | 意图分类、工具选择、通用辅导、复杂 JSON；延迟 1.8s / 可用率 99.9% |
| 备选 | deepseek-chat | 批量生成与回退；化学满分、成本最低 |
| 兜底 | MiMo-V2.5 | 联网搜索与图像请求；唯一原生支持视觉 + 联网 |
| 视觉（不在回退链内） | qwen-vl-ocr / glm-4v | 图片识别、OCR、文档解析复核 |

回退链每级 3 次重试 + 指数退避（1s → 2s → 4s）。不使用海外模型与自部署开源模型。

**其他**：APScheduler 3.10.4（每日练习 08:00 UTC / 预警 00:00 UTC / OCR 轮询 5s）；python-jose JWT（HS256，access 24h + refresh 7d）+ passlib[bcrypt] 1.7.4 + RBAC 中间件；化学配平用自定义系数平衡算法 + RDKit 2024.3.1（**必须确定性算法**，LLM 配平仅约 80%）；Playwright ≥1.40；Pytest 8.0.0；Docker + Compose（python:3.11-slim）

**前端**：教师端 React 19 + Vite 8 + Tailwind 4 + antd 6 + ECharts 6 + KaTeX/mhchem + marked（选 SPA 因 Agent 流式渲染与多 Tab 工作台交互复杂）；学生端与家长端是**无构建步骤**的轻量静态页；桌面端 pywebview + 内嵌 Uvicorn（127.0.0.1:8000）+ PyInstaller 6.x

**OCR 降级链**：PP-OCRv6（答题卡默认，免费无网络依赖）→ 百度 AI 教育 OCR（在线降级）→ MinerU（PDF 与 LaTeX 标准化）→ VLM（GLM-4V / MiMo）兜底。PDF 优先 MinerU，图片优先 PP-OCRv6，混合文档并行后合并。

## 目录结构

```
ChemAI/
├── CLAUDE.md / CONTEXT.md / DESIGN.md
├── openspec/                            # 规格驱动开发
├── docs/{设计文件, adr, agents}/
├── app/                                 # FastAPI 应用（根目录直挂，不套 backend/ 层级）
│   ├── models/                          # 24 个 SQLAlchemy 模型
│   ├── api/  services/  middleware/  utils/
│   └── mcp/                             # MCP 工具服务器（16 工具 + 3 端点）
├── agent/{tools, channel, prompts}/      # 42 领域工具 + 5 浏览器工具 / SSE 通道 / Prompt 模板
├── chem_skills/                         # 10 个化学技能模块（Agent 工具底层实现）
├── frontend/{pages, js, css, m, legacy/teacher/pages}/
├── data/{exam_questions, knowledge_graph, chromadb}/
├── alembic/versions/                    # 数据库迁移
└── scripts/  evals/  tests/  logs/      # run_evals.py / baseline.json / pytest / audit.log
```

> **路径口径**：代码根目录就是项目根目录。文档里出现 `backend/app/...`（43/58 号残留）按 `app/...` 解读，`app/routers/` 按 `app/api/` 解读——**不要另建 `backend/`**。树根名 `ChemAI/` 是本机实际目录名，文档里的 `chemai-backend/` 指同一处。

## 常用命令

```bash
uvicorn app.main:app --reload --port 8000   # 项目根目录下
alembic upgrade head                        # 应用迁移
pytest tests/ -v                            # 全量测试
python scripts/run_evals.py --tier l1       # Evals 分层
python scripts/run_evals.py --tier all --run-slow --compare evals/baseline.json   # 含 L3
npm run dev                                 # frontend/ 下
git push no-mistakes <branch>               # 唯一推送方式
```

不加 `--run-slow` 时 L3 慢跑层默认跳过。

## 工具链（九层）

```
思考层：/office-hours /plan-ceo-review /plan-eng-review     ← Gstack
对齐层：/grill-with-docs → CONTEXT.md + ADR                 ← Matt Skills
规格层：/opsx:propose → /opsx:apply → /opsx:archive         ← OpenSpec
实现层：写测试(RED) → 生成实现(GREEN) → 重构                 ← TDD
质量层：L1 单元 → L2 集成 → L3 质量 → 基线对比               ← Evals
流程层：/review /code-review /cso /qa /ship /retro          ← Gstack + Matt
门禁层：/no-mistakes + git hooks + CI                       ← no-mistakes
骨架层：branch commit tag merge revert push                 ← Git
原生层：Plan 模式 Checkpoint Hooks Subagents                ← Claude Code
```

## 开发流程（每阶段 10 步）

`git checkout -b phase-N/<主题>` → `/opsx:explore` → `/opsx:propose` →（仅架构重阶段 `/plan-eng-review`）→ `/opsx:apply`（内含 TDD：写测试 RED → 生成实现 GREEN → pytest → 提交）→ `/code-review` → `/qa` → `/opsx:archive` → `git merge --no-ff` + tag → `git push no-mistakes`

| 阶段 | 文档 | 内容 | 版本 |
|---|---|---|---|
| 一 | 42 | 环境搭建 + 项目骨架 | v0.1.0 |
| 二 | 43 | 数据模型与认证（24 模型 / JWT / RBAC） | v0.2.0 |
| 三 | 44-46 | 出题工作台 + 题库管理 + 障碍诊断引擎 | v0.3.0 |
| 四 | 47-51 | 自适应练习、复习、学情面板、OCR、学生端、家长端 | v0.4.0 |
| 五 | 52 | Evals 评测体系（Golden + 基线 + CI） | v0.5.0 |
| 六 | 53-57 | Agent 架构 + 30 个工具实现（注册表 42 领域 + 5 浏览器）+ Persona + GuardState + MCP | v0.6.0 |
| 七 | 58-59 | 全链路验收 + 灰度发布 + 复盘 | v1.0.0-rc.1 → v1.0.0 |

架构重阶段：43 / 45 / 46 / 49 / 52 / 53。
可并行：47/48/49、50/51、54/55/56。必须串行：44→45→46、53 最先、58→59。
轻任务（改文案、调样式）不走 10 步：`/grill-me` → `/implement` → `/code-review` → commit。做下一个模块前跑 `/graphify`。

## 质量门禁

- **TDD**：写测试（RED）→ 实现（GREEN）→ pytest 全绿
- **Evals**：L1 单元 ≥95%（pre-commit）· L2 集成 ≥90%（每个工具组）· L3 质量 ≥70%（每日收工与 push 前）
- **推送**：只走 `git push no-mistakes`（review → test → docs → lint → push → PR → CI）
- **劣化**：≤3% 记录后继续；3-5% `/investigate` 修复；>5% revert 或修复后重跑
- **一次性配置**：43 号首阶段配 git hooks（pre-commit 跑 pytest，commit-msg 校验提交格式）

## 编码约定

**命名**：Python 模块 `snake_case` / 类 `PascalCase` / 常量 `UPPER_SNAKE_CASE`；前端组件 `PascalCase` / 函数 `camelCase`。模型名与 33 号文档实体名一致（`StudentAnswer` / `ReviewTask` / `WarningLog`）。
API 前缀 `/api/<模块>`：`/api/ocr`、`/api/diagnosis`、`/api/practice`、`/api/exam-bank`、`/api/review`、`/api/panel`、`/api/warning`、`/api/agent`；MCP 走 `/api/mcp`（34 号的「无前缀」是残留）。枚举值以 `CONTEXT.md` 为准，不自创。

**项目规范**：代码注释与文档用**中文**；Python 遵循 **PEP 8**；模型统一 SQLAlchemy ORM，API 遵循 RESTful，开发用 TDD；化学方程式一律用 `$...$` 包裹的 LaTeX；AI 生成内容必须过审核引擎才可输出。

**数据**：SQLite 不原生支持 JSON，引擎创建时配 `json_serializer` / `json_deserializer`。`students.barrier_type` 必须三键齐全（三值和为 1.0），聚合时缺失的键补 0。多租户隔离沿「学校→年级→班级」链路过滤，教师只见本校、且只限自己任教的班级。批量写入每条独立提交，诊断在全部保存成功后统一触发。

**化学红线**：系数配平必须 **100% 正确**（不可协商），用确定性算法而非 LLM。用户可见方程式须同时满足四维：**系数配平 / 反应条件 / 产物稳定性 / 分子结构**。化学式渲染统一走 KaTeX + mhchem，归一化属审核前置格式化与出题后处理，不是响应阶段临时做的。`blocked` 题目绝不下发学生端。

**前端**：写页面前先读 `DESIGN.md`，颜色 / 字体 / 圆角 / 组件样式不得自创。设计令牌通过 Tailwind 类名 + antd 主题集中控制。学生端与家长端视口固定 430×900。

## Git 约定

Conventional Commits：`<type>(<scope>): <subject>`。type 取 feat / fix / test / refactor / docs / chore / perf；scope 取 model / auth / review / exam / diagnosis / ocr / agent / eval / persona / guard / ui / golden-NNN。
分支 `phase-N/<主题>`（如 `phase-2/data-model`），模块内子任务可切 `feat/<主题>`。日常阶段（43-58）本地 `merge --no-ff` + tag，不开 PR；发布阶段（59）走 `/ship` 的 PR 仪式。推送统一 `git push no-mistakes`，**唯一例外**是 42 号的首次推送。
架构级决策必须写 ADR 到 `docs/adr/YYYY-MM-DD-<主题>.md`：背景 / 决策 / 理由（对比至少一个替代方案）/ 结果。

## 行为准则

1. **先想再写**——动手前说出假设，不确定就问；有歧义并列出来，不要默默选一个。
2. **最简单方案**——不做没要求的功能、抽象、配置；不为不可能的场景写错误处理；200 行能压到 50 行就重写。
3. **最小改动**——不顺手优化相邻代码，不重构没坏的东西；只清理自己改动产生的孤儿。每行改动都要能追溯到需求。
4. **目标可验证**——把任务转成可验证目标（"修 bug" → 先写复现测试再让它通过）；多步任务先列计划与验证点。
5. **测试质量红线**（本文档自拟，据 00 号 §6.5 授权）——禁止写只读源码找字符串的测试；必须执行真实接口，断言可观察的行为、状态、输出与失败模式。

## Agent skills

> 非设计文档派生，来自本机技能包（`setup-matt-pocock-skills`），供 `to-tickets` / `triage` / `to-spec` 读取。**不要删改**。

- Issue tracker：`.scratch/` 下的本地 markdown。见 `docs/agents/issue-tracker.md`
- Triage labels：`needs-triage` / `needs-info` / `ready-for-agent` / `ready-for-human` / `wontfix`。见 `docs/agents/triage-labels.md`
- Domain docs：单项目布局，根目录 `CONTEXT.md` + `docs/adr/`。见 `docs/agents/domain.md`
