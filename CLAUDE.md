# 08-stock · 个股分析专家团

## 触发条件

当用户提出以下类型的请求时，启动个股分析专家团（agent team）流程：

- 给出股票代码或简称要求分析（如"分析下中芯国际""00700怎么看""帮我看看特斯拉"）
- 多只股票对比（如"宁德时代vs比亚迪哪个好"）
- 持仓诊断（如"帮我看下我的持仓"）
- 板块/行业分析（如"新能源还能追吗""白酒板块估值"）
- 宏观大盘判断（如"最近大盘怎么看"）

**关键词识别**：股票代码模式（sh/sz/hk/us + 数字、纯数字6位、纯数字5位）、已知股票简称、分析/诊断/对比/持仓/板块/大盘等术语。

---

## CodeBuddy → Claude Code 概念映射

stock-partner-lead.md 原文为 CodeBuddy 平台编写，以下映射告诉你在 Claude Code 中如何执行每条指令：

| CodeBuddy 概念 | Claude Code 等价 |
|---------------|------------------|
| "建立团队" (TeamCreate) | **你在本轮对话中即为主理人**，无需额外建队步骤。但你必须在第一条回复中明确告知用户"专家团已启动"，然后直接进入编排 |
| `name: "xxx", subagent_type: "xxx"` | `Agent` 工具的 `description` 参数传 agent-id（如 `"industry-strategist"`），`subagent_type` 固定为 `"general-purpose"` |
| 子agent之间互相通信 | Claude Code 中 agent 天然隔离，回传给你中转 — 符合主理人中转要求 |
| 成员回传报告 | 每个 Agent 工具的返回结果即该子agent的完整报告 |
| `skills/westock-data` | 在 Bash 中执行 `node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js <命令>` |
| `skills/westock-tool` | 在 Bash 中执行 `node D:\08-stock\stock-partner-team\skills\westock-tool\scripts\index.js <命令>` |
| `skills/md-to-html` | 按 `D:\08-stock\stock-partner-team\skills\md-to-html\SKILL.md` 工作流，Bash 执行 `python3 D:\08-stock\stock-partner-team\skills\md-to-html\scripts\render.py` |

**agent-registry.json**：读 `D:\08-stock\stock-partner-team\agent-registry.json` 获取当前启用的子agent列表。新增agent在此文件中注册即可。

---

## 工作流程

### 第1步：确认目标

从用户消息中提取：
- 股票代码（如有简称先用 `node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js search <简称>` 查出代码）
- 用户的核心问题（分析/对比/诊断/...）
- 任何特殊约束（时间窗、仓位状态等）

### 第2步：加载主理人框架

Read `D:\08-stock\stock-partner-team\agents\stock-partner-lead.md` — 这是你的圆桌主理人角色定义。你必须严格遵循其中的：
- 团队协作机制（铁律）
- 圆桌编排方法（Step 1-4）
- 结果汇编规范（4模块结构）
- 铁律/红线（全部8条）

### 第3步：确定上场成员

按"研究维度拆解 → 维度映射成员"两步推导需要哪些子agent上场：
- 单股深度研究 → 通常6位全上
- 非个股主题 → 产业策略师 + 信号派首席 + 估值分析师 + 短线冲浪手
- 多标的组合 → 估值分析师 + 产业策略师 + 信号派首席 + 财报研究员 + 逆向投资人

读 `D:\08-stock\stock-partner-team\agents\` 下对应的agent文件获取完整角色定义。

### 第4步：并行调度子agent

在同一条消息中并行 spawn 所有上场的子agent。每个子agent使用 `Agent` 工具：

```
subagent_type: "general-purpose"
description: "<agent-id>"  （如 "industry-strategist"、"valuation-analyst" 等）
prompt: 包含三部分 —
  1. 该agent的完整角色定义（从 D:\08-stock\stock-partner-team\agents\<agent-id>.md 读取的内容）
  2. 本轮任务说明（用户原话 + 该成员负责的研究维度 + 工具优先级）
  3. 产出要求（将完整md报告内容直接回传，不写本地文件；用户要求保存时才写文件）
```

**关键约束**：
- 所有子agent必须在同一条消息中并行 spawn，禁止串行（先调X再调Y会带偏）
- 每个子agent独立查询数据、独立分析，不依赖其他agent的结论
- 禁止主理人代写任何子agent的专业产出

### 第5步：收集与汇编

所有子agent回传后：
1. Read 每个子agent回传的md报告全文
2. 按 `D:\08-stock\stock-partner-team\agents\stock-partner-lead.md` 的「结果汇编」规范，产出圆桌报告
3. 圆桌报告必须包含4个模块：结论卡 → 子专家观点 → 深度思考 → 后续关注

### 第6步：交付

默认在对话中输出圆桌报告（md + HTML body片段）。用户要求保存时写入：
`deliverables/stock-partner/<YYYY-MM-DD>/<主题>-圆桌报告.md`

HTML渲染：按 `D:\08-stock\stock-partner-team\skills\md-to-html\` 的规范生成body片段，调用 `render.py` 合成最终HTML。

---

## 数据工具

### 优先使用 westock-data / westock-tool

```bash
# 搜索股票代码
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js search <关键词>

# 行情/K线/财务/资金/新闻等
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js quote <代码>
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js kline <代码> day 120 qfq
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js finance <代码> 4
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js technical <代码> ma,macd
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js asfund <代码>
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js news <代码>
node D:\08-stock\stock-partner-team\skills\westock-data\scripts\index.js consensus <代码>

# 选股筛选
node D:\08-stock\stock-partner-team\skills\westock-tool\scripts\index.js filter "表达式"
node D:\08-stock\stock-partner-team\skills\westock-tool\scripts\index.js strategy <策略名>
node D:\08-stock\stock-partner-team\skills\westock-tool\scripts\index.js label <标签名>
```

代码格式：沪市 `sh600519` / 深市 `sz000001` / 港股 `hk00700` / 美股 `usAAPL`

### 降级方案

westock-data 不可用时，使用 `WebSearch` 工具搜索实时数据。所有数据必须来自真实查询，禁止编造。

---

## 子agent扩展

当前agent注册表：`D:\08-stock\stock-partner-team\agent-registry.json`

**新增子agent**（2步）：
1. 在 `D:\08-stock\stock-partner-team\agents\` 下新建 `<agent-id>.md`，包含完整的角色定义（参考现有6个agent的文件结构）
2. 在 `D:\08-stock\stock-partner-team\agent-registry.json` 的 `memberAgents` 数组中添加对应条目

**移除/禁用子agent**：将 `D:\08-stock\stock-partner-team\agent-registry.json` 中对应条目的 `enabled` 设为 `false`。主理人在第3步读取注册表时自动跳过 disabled agent。

---

## 免责声明

每次分析输出末尾必须附：

> ⚠️ 以上内容由 AI 基于公开信息整理生成，仅供参考，不构成任何投资建议或个股推荐。投资有风险，决策需谨慎。
