# CONTEXT.md

ChemAI（智辅化学）领域词汇表。**写代码、命名变量、开 issue、写测试之前先在这里对齐术语**——输出里出现领域概念时用本表的名字，不要漂移到同义词。

定义出自 `docs/设计文件/4.产品设计/`（19-41 号）；代码标识符与 33 号 §八 一致。需要新增术语时：要么是在发明项目没在用的语言（重新考虑），要么是真缺口（记下来交给 `/domain-modeling`）。

---

## 一、核心实体（Core Entities）

数据层 24 个实体，映射 26 张表。四条核心数据链：组织链、教学链、家校链、身份链。

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 学校 | School | `School` | 顶层容器：名称、地区、地址、电话、当前学期 |
| 年级 | Grade | `Grade` | 学校下的年级（如"高一"），含名称与学年 |
| 班级 | Class | `Class` | 年级下的班级；**数据隔离的核心边界** |
| 教师 | Teacher | `Teacher` | 归属某校，含账号状态（待审核/已通过/已拒绝）与 4 种子角色 |
| 学生 | Student | `Student` | 核心实体：障碍画像、练习追踪、6 位家长绑定码 |
| 家长 | Parent | `Parent` | 通过绑定码关联子女，一个家长可绑多个 |
| 账户 | Account | `Account` | 统一登录体系，一对一关联教师/学生/家长之一 |
| 任课关系 | Teacher-Class-Subject | `TeacherClassSubject` | 教师与班级的多对多表，标注是否班主任 |
| 亲子绑定 | Student-Parent Binding | `StudentParentBinding` | 绑定状态 + 双方关系（父/母/其他监护人） |
| 家长通知 | Parent Notification | `ParentNotification` | 推送给家长的消息。类型 5 值：`weekly_report` / `score_alert` / `learning_plan` / `reminder` / `daily_report` |
| 考试记录 | Exam Record | `ExamRecord` | 一次考试/练习/作业，归属班级，含类型、日期、参考人数、平均分 |
| 题目 | Question | `Question` | 一道完整试题：题干、选项、答案、解析、知识点、难度、来源、审核状态 |
| 学生作答 | Student Answer | `StudentAnswer` | 学生 × 题目 × 考试；含作答内容、正误、障碍标签、连续对错计数 |
| 学生提交 | Student Submission | `StudentSubmission` | 单次考试中一个学生的答题卡提交，含原始与批改后图片 |
| 上传会话 | Upload Session | `UploadSession` | 上传文件的完整生命周期追踪（状态机见 §六） |
| 批改任务 | OCR Task | `OCRTask` | 一张答题卡一个任务，含 OCR 原始结果与批改结果 |
| 题库文件夹 | Question Set | `QuestionSet` | 题目的组织容器 |
| 文件夹-题目关联 | Question Set Item | `QuestionSetItem` | 题目与文件夹的多对多关系，含排序字段 |
| 历年真题卷 | Historical Exam | `HistoricalExam` | 预置的高考真题/模拟卷：名称、来源地区、年份 |
| 历年真题 | Historical Question | `HistoricalQuestion` | RAG 的知识底座，含区分度等统计字段 |
| 知识图谱节点 | Knowledge Point | `KnowledgePoint` | 一个化学知识点：分类、PubChem 编号、关联题目数、动态错误率 |
| 障碍诊断配置 | Barrier Config | `BarrierConfig` | 教师自定义阈值：概念 3 / 审题 2 / 表述 3 / 掌握 3 |
| 复习任务 | Review Task | `ReviewTask` | 错题的艾宾浩斯安排：级别 0–5、状态 `pending` / `overdue` / `done`、升降级计数、复习历史 |
| 学情预警 | Warning Log | `WarningLog` | 一次预警：类型、级别（提示/警告/紧急）、处理状态、通知标记 |

---

## 二、学习概念（Learning Concepts）

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 最近发展区 | Zone of Proximal Development | `ZPD` | 正确率 40%–70% 为最佳学习区间；>90% 太简单、<30% 太难 |
| ZPD 难度档位 | ZPD Difficulty Level | `easy` / `medium` / `hard` | 由最近 **30 条**作答记录的正确率三档映射；`competition` 不自动分配 |
| 自适应练习引擎 | Adaptive Practice Engine | — | 消费诊断结果，为每个学生生成落在 ZPD 的个性化练习 |
| 自适应策略矩阵 | Adaptive Strategy Matrix | — | 主导障碍 → 难度偏移 / 知识点倾向 / 题型配比 的映射 |
| 间隔重复复习 | Spaced Repetition | `SpacedRepetitionEngine` | 按遗忘曲线自动安排错题复习提醒（被动系统） |
| 错题强化训练 | Wrong Question Training | `WrongQuestionTrainer` | 学生主动打开错题本、生成变式题即时强化（主动系统） |
| 艾宾浩斯复习级别 | Review Level | `review_level` | 0–5 共 6 级，间隔 1/3/7/14/30 天递增，第 5 级为已掌握 |
| 变式题 | Variant Question | `variant` | 与原题**同知识点、同难度、不同题面**的新题，默认 3 道 |
| 每日练习 | Daily Practice | `create_daily_practice` | 每天 08:00 UTC 自动生成的 10 题练习，知识点由主导障碍决定 |
| 学习计划 | Learning Plan | `students.current_plan` | 教师制定的个性化干预方案（每日任务列表）；存于 `students.current_plan` 字段而非独立表，生成后缓存 24h |
| 进步率 | Improvement Rate | `improvement_rate` | 最近一次练习正确率 − 前一次；>0.1 显著进步，<−0.05 退步需重新诊断 |
| 掌握标准 | Mastery Threshold | `mastery_threshold` | 同一知识点连续答对 N 次视为已掌握（默认 3） |
| 练习效果追踪 | Practice Effect Tracking | `/api/practice/effect/{student_id}` | 三级：基础级（两次正确率对比）/ 知识点级 / 综合级（ZPD 档位流转） |

---

## 三、诊断概念（Diagnosis Concepts）

核心差异化：从"知道谁错了"进化到"知道**为什么错**"。

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 障碍类型 | Barrier Type | `BarrierType` | 错误作答的根因三分类：概念理解型 / 审题障碍型 / 表述障碍型 |
| 概念理解型 | Concept-Comprehension Barrier | `concept` | 不理解底层概念原理——混淆相似概念、无法解释推理过程 |
| 审题障碍型 | Reading Barrier | `reading` | 读题不完整或落陷阱——遗漏关键词、答非所问，但概念其实掌握 |
| 表述障碍型 | Expression Barrier | `expression` | 理解正确但表达不规范——方程式写错、漏单位/有效数字 |
| 障碍画像 | Barrier Profile | `Student.barrier_type` | 学生三维障碍占比的 JSON（三值和为 1.0），由诊断引擎异步更新 |
| 主导障碍 | Dominant Barrier | `dominant_barrier` | 障碍画像中占比最高的类型 |
| 诊断置信度 | Diagnosis Confidence | `confidence` | LLM 把握度 0.0–1.0；≥0.8 自动采纳，0.7–0.8 标注"需关注"，<0.7 建议人工复核 |
| 薄弱知识点 | Weak Knowledge Point | `weak_knowledge_points` | 错题中错误频次最高的知识点（默认 Top 3）；**全量统计不设时间窗** |
| 教师覆盖 | Teacher Override | `/api/diagnosis/override/{student_id}` | 教师手动推翻 AI 判定；写入时指定类型占 90%、其余各 5%，并记录操作日志 |
| 规则引擎初筛 | Rule Engine Pre-classification | — | 关键词规则库快速预分类（置信度 0.5–0.7）；属增强能力，主链路以 LLM 语义分析为主 |
| 三段式诊断架构 | Three-Stage Diagnosis | — | 规则引擎初筛 → LLM 深度分析 → 综合判定 |
| 三维分析立方体 | Three-Dimensional Analysis Cube | — | 知识点 × 学生 × 时间，两两相交各揭示一种洞察 |
| 知识点错误率 | Knowledge Point Error Rate | `E(kp, c)` | 班级 c 在知识点 kp 上的答错次数 ÷ 作答总次数 |
| 学情预警 | Early Warning | `EarlyWarningService` | 三类基础规则：连续未登录 3 天 / 成绩下滑 ≥10% / 错题率过高 ≥50% |
| 预警级别 | Warning Level | `info` / `warning` / `critical` | 提示 / 警告 / 紧急；紧急由成绩下滑 ≥20% 触发 |

---

## 四、题目与考试（Questions & Exams）

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 题型 | Question Type | `choice` / `fill` / `calc` / `experiment` / `inference` | 选择 / 填空 / 计算 / 实验 / 推断；v1 仅启用 `choice` |
| 难度 | Difficulty | `easy` / `medium` / `hard` / `competition` | 简单 / 中等 / 困难 / 竞赛；竞赛级仅手动录入，不自动分配 |
| 题目来源 | Question Source | `ai` / `manual` / `daily_practice` / `ocr` | AI 生成 / 手动录入 / 每日练习 / OCR 导入，用于溯源与审核策略差异化 |
| 审核状态 | Review Status | `passed` / `warning` / `blocked` | 通过 / 警告可用 / 阻断不可用；**阻断的题目不下发学生端** |
| 四维安全审核 | Four-Dimension Safety Audit | — | 化学方程式出厂前最后一道安全门：配平 / 反应条件 / 产物稳定性 / 分子结构 |
| 系数配平审核 | Coefficient Balancing Audit | `balance` | 元素原子计数法逐元素比对两侧原子数；**100% 是 HARD RED LINE** |
| 反应条件审核 | Reaction Condition Audit | `condition` | 14 条件关键词规则库（点燃/加热/高温/催化剂/通电…），遗漏则 FAIL 或 WARNING |
| 产物稳定性审核 | Product Stability Audit | `product` | 正则检测不稳定产物（H₂CO₃、NH₄OH 应写作 CO₂↑、NH₃↑）与沉淀/气体规则 |
| 分子结构审核 | Molecular Structure Check | `structure` | 校验元素符号大小写、下标、括号匹配、离子电荷、LaTeX/mhchem 格式 |
| 硬拦截 | Hard Block | `blocked` | 配平错误——打回重生成，绝不可输出 |
| 软警告 | Soft Warning | `warning` | 条件缺失/格式不完美——标记但不拦截，输出时带"审核建议" |
| 出题工作台 | Exam Workbench | `exam-workbench` | 教师端四 Tab 单页工作台：出题工作台 / 题库管理 / 历史真题库 / 考试列表 |
| 考试类型 | Exam Type | `monthly` / `practice` / `homework` | 月考 / 练习 / 作业；月考影响学情报告，练习影响 ZPD 计算 |
| 考试状态 | Exam State | `Draft` → `AddingQuestions` → `Published` → `InProgress` → `Completed` | 考试生命周期的 5 个状态，按此顺序流转 |
| 蓝本 | Blueprint | `variant_qid` | 生成变体题时指定的源真题 |
| 检索增强生成 | RAG | — | 从历年真题库检索相似题作为 LLM 出题的参考素材 |
| RAG 上下文 | RAG Context | — | 向量检索命中 ≥3 道相似真题时才注入的参考素材；不足则不注入 |
| Agentic GraphRAG | Agentic GraphRAG | — | 在向量 RAG 之上叠加知识图谱：以 KnowledgePoint 为图节点、题型/考点为边，支撑跨知识点推理检索 |
| 试卷 | Exam Paper | `HistoricalExam` / `ExamRecord` | 一份完整试题集合；历年真题卷是 `HistoricalExam`，班级考试记录是 `ExamRecord` |
| 周报 | Weekly Report | `weekly_report` | 面向家长的周学习报告：统计概览、知识点掌握度、教师评语；由 LLM 以通俗语言生成 |
| 区分度 | Discrimination | — | 衡量题目区分优生与差生的统计指标 |
| 审核报告 | Audit Report | `AuditReport` | 四维审核的结构化 JSON 输出，含 `overall_status`（passed / blocked） |
| 化学式归一化 | Chemical Formula Normalization | — | 审核前置步骤：统一 LaTeX 箭头、裸化学式自动包裹 `$...$` |

---

## 五、Agent 概念（Agent Concepts）

单智能体 ReAct 架构，42 个领域工具 + 5 个浏览器工具。

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 角色人格 | Persona | `Persona` | 4 套配置（教师/学生/家长/辅导），决定可用工具集与 system prompt |
| 意图 | Intent | `chat` / `navigate` | 用户输入的两类归宿：`chat` 走工具执行与对话，`navigate` 直接触发页面跳转 |
| 意图分类网关 | Gateway | `Gateway` | 判定输入属于 `chat` 还是 `navigate`；准确率目标 ≥85% |
| ReAct 循环 | ReAct Loop | `create_react_agent` | Thought → Action → Observation；v2 单 Agent 递归上限 12 |
| 工具元数据注册表 | Tool Metadata Registry | `TOOL_META` | 以工具函数为键注册可用 Persona 与每轮调用上限；启动时编译期完整性断言 |
| 护栏 | Guard | `GuardState` | 四层：前置检查 → 调用限制 → 去重 → 审批门控 |
| 审批门控 | Approval Gate | `requires_approval_blocked` | `assign_adaptive_practice`、`delete_bank` 必须教师确认才执行 |
| 内联面板 | Inline Panel | `_component` | Agent 在对话流中内联渲染的面板（出题工作台、诊断图表、学生列表） |
| 页面跳转指令 | Navigate Directive | `_route` | Agent 通过工具返回值触发前端页面跳转与数据填充 |
| 页面桥接 | Page Bridge | — | `navigate` / `populate` / `actions` 三字段协议，Agent 驱动前端导航 |
| 服务端事件协议 | SSE Event Protocol | — | 事件类型（以 39 号为准）：`phase` / `text` / `tool_call` / `tool_args` / `tool_result` / `component` / `navigate` / `populate` / `action` / `exam_images` / `done`（39 号自述"10 种"，§三 实际列 11 个名） |
| Agent 阶段 | Agent Phase | `thinking` / `executing` / `planning` / `reply` / `awaiting_approval` | 分析中 / 执行中 / 规划中 / 回复中 / 等待审批 |
| 上下文管理器 | Context Manager | `ContextManager` | 三层裁剪 + token 预算分配；累计消息超 30 条触发智能裁剪 |
| 对话检查点 | Conversation Checkpoint | `checkpointer` | 短期记忆——单次对话状态快照，支撑多轮引用、审批中断恢复、异常回退 |
| 长期记忆 | Long-term Memory | — | 跨对话持久化：学生学情历史、教师偏好；新对话开始时自动加载 |
| 三层记忆架构 | Three-Tier Memory | `MemorySystem` | 记忆系统的分层设计 |
| 审计日志 | Audit Log | `AuditLogger` | JSON 结构化日志，含环形缓冲区与敏感字段脱敏，best-effort 写入 |
| Provider 回退 | Provider Fallback | — | 三级：qwen-turbo → deepseek-chat → MiMo；每级 3 次重试 + 指数退避 |
| 熔断器 | Circuit Breaker | `ProviderHealthChecker` | Provider 健康检查与熔断，仅在调用失败时触发（lazy checking） |
| 任务规划 | Planner | `plan_task` | 复杂任务拆解为步骤计划，经 SSE 推送进度卡片 |
| 工具模型上下文协议 | MCP | `/api/mcp` | 把 Agent 工具暴露为 MCP 服务器，供外部客户端调用 |
| 工具去重 | Tool Dedup | `dedup_skipped` | 相同 tool + args 的重复调用被自动跳过 |
| 化学技能模块 | Chemistry Skill Module | `chem_skills/` | Agent 工具的底层实现，10 个模块 + 共享模板 |
| 苏格拉底式辅导 | Socratic Tutoring | — | 6 个引导式辅导工具，用提问引导学生自己推导而非直接给答案 |

---

## 六、OCR 概念（OCR Concepts）

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 答题卡 OCR 批改 | Answer Sheet OCR Grading | — | 教学工具链的**入口环节**：上传 → OCR → 批改 → 入库 → 触发诊断 |
| 批改管线 | Grading Pipeline | — | 10 步：批量上传 → OCR 识别 → 学生信息提取 → 答案来源选择 → LLM 语义批改 → 教师确认 → 保存 → 触发障碍诊断 → 班级统计 → 报告生成 |
| 上传会话状态机 | Upload Session State Machine | `UploadSession` | UPLOADED → PREVIEWING → READY → IMPORTING/IMPORTED、GRADING；终态 DISCARDED / ERROR |
| 批改任务状态机 | OCR Task State Machine | `OCRTask` | pending → processing → done / failed；失败可手动重试 |
| 降级链 | Degradation Chain | — | 引擎失败时的逐级回退（图片：PP-OCRv6 → 百度 → VLM；PDF：MinerU → 百度 → VLM） |
| PP-OCRv6 | PaddleOCR v6 | `OCR_SHEET_PROVIDER=paddle` | 答题卡图片**默认引擎**——本地开源、免费、onnxruntime 推理无网络依赖 |
| 百度教育 OCR | Baidu Edu OCR | — | 在线高精度降级引擎；`handprint_mix` 模式针对手写+印刷混合 |
| MinerU | MinerU | — | PDF 试卷解析引擎，对化学式（LaTeX）、分子结构、反应方程式效果好 |
| 视觉模型兜底 | VLM Fallback | `glm-4v` / `mimo` | 最终兜底，失败时标记 `fallback_used=true` |
| 答案来源 | Answer Source | — | 三级优先级：题库匹配（最高）→ 教师录入 → LLM 自判（最低） |
| 文档解析引擎 | Document Parsing Engine | — | 按文件类型自动路由到对应引擎的调度层 |
| 批改判定编码 | Grading Result Code | `correctResult` | 百度 correct_edu 返回的**判定结果**（不是置信度）：0 未处理 / 1 正确 / 2 错误 / 3 未作答；0 标记"建议人工复核" |
| 预览 | Preview | — | OCR 先行识别、供教师确认的那一步；预览完成（READY）才允许选择"导入题库"或"批改判卷" |
| 任务轮询 | Task Polling | — | APScheduler 每 5 秒扫描 `ocr_tasks` 中最早的 `pending` 任务；前端同频轮询批次状态 |
| 低置信度标记率 | Low-confidence Rate | — | 被标记"建议人工复核"的答题卡占比，目标 <10% |
| 端到端时间 | End-to-End Time | — | 从点击上传到全部批改结果就绪，目标 <5 分钟/班 |

---

## 七、评测概念（Evaluation Concepts）

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 黄金场景 | Golden Scenario | — | 输入 + 期望行为断言的评估用例 |
| 七层评估模型 | L0–L6 Evaluation Model | `L0`–`L6` | L0 工具正确性 / L1 行为轨迹 / L2 输出质量 / L3 安全护栏 / L4 性能预算 / L5 稳定性回归 / L6 端到端任务 |
| 快跑层 | Fast Track | `--fast` | L0+L1+L4+L5，零 LLM 成本秒级；**100% 必须绿** |
| 慢跑层 | Slow Track | `--slow` | L2+L3+L6，分钟级花 token；**严重错误一票否决**，分数只做趋势 |
| 判官 | LLM-as-judge | — | 用 LLM 按 rubric 打分；判官与主模型**解耦**（用备选 Provider） |
| 判官五族 | Five Judge Families | — | 出题 / 判卷 / 诊断 / 方案 / 交互智能 |
| 校准集 | Calibration Set | — | 人工标注样本，每族 20–30 条；判官与人工一致率 ≥80% 才启用自动计分 |
| 评估先行 | Eval-first | — | 先写评估（定义"什么算好"）再开发；红 → 绿的过程即开发进度指标 |
| 预期失败标记 | expected_fail | `expected_fail` | 红评估的豁免标记；绿了会提醒移除，避免"死标记" |
| 严重错误 | Severe | `severe` | 化学错误 / 安全问题 / 端到端失败——慢跑层一票否决的依据 |
| 基线对比 | Baseline Comparison | `evals/baseline.json` | 与历史基线比对，劣化 >5% 阻断 |
| 性能预算 | Performance SLO | — | 单轮延迟 p95 ≤60s、单次运行 token ≤40k、循环数 ≤12；软警告起步 |
| 覆盖诚实 | Coverage Honesty | — | 未实现的层如实标"未实现"，不用 0/0 冒充 100% |
| 门禁 | Gate | — | `git push no-mistakes` 触发 review → test → docs → lint → push → PR → CI |

---

## 八、技术基础（Technical Foundation）

| 中文名 | 英文名 | 代码标识 | 定义 |
|---|---|---|---|
| 规格驱动开发 | Spec-Driven Development | `openspec/` | 先约定再动手：propose 提案 → apply 按规格生成代码 → archive 归档 |
| 架构决策记录 | ADR | `docs/adr/` | 记录架构级决策的背景/决策/理由/结果；**决策不记录 = 没发生** |
| 多租户数据隔离 | Multi-tenant Data Isolation | — | 沿「学校 → 年级 → 班级」链路隔离，班级是核心边界 |
| 权限矩阵 | RBAC Permission Matrix | `ROLE_PERMISSIONS` | **6 角色**（系统管理员/教务管理员/学科组长/普通教师/学生/家长）× 14 个 API 域 |
| 桌面三入口 | Three Desktop Entries | — | pywebview 打包教师端 / 学生端 / 家长端三个 .exe |
| 设计令牌 | Design Token | — | 颜色/字体/间距集中定义（Tailwind 类名 + antd 主题），保证跨页面一致 |
| 学术催化剂 | Academic Catalyst | — | 设计系统核心隐喻：降低学习反应的活化能，自身不消耗 |
