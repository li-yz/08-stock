# 08-stock

个股分析专家团 —— 基于 Claude Code 的多 agent 协作投研系统。

## 结构

- `CLAUDE.md` — 项目级工作流定义（6步流程、CodeBuddy→Claude Code 映射表）
- `stock-partner-team/agents/` — 1 个主理人 + 6 个子 agent 角色定义
- `stock-partner-team/agent-registry.json` — agent 注册表（可扩展）
- `stock-partner-team/skills/` — 数据工具（westock-data / westock-tool / md-to-html）
- `中芯国际/` — 案例分析示例（完整圆桌报告）

## 使用

在 Claude Code 中给出股票代码或简称即可自动触发专家团分析。支持 A 股（sh/sz）、港股（hk）、美股（us）。

## 免责声明

以上内容由 AI 基于公开信息整理生成，仅供参考，不构成任何投资建议或个股推荐。投资有风险，决策需谨慎。
