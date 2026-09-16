# 第 1 章学习档案：上下文与 Agent 基础

## 当前进度

- 已完成：[Context 上下文消融实验](context/README.md)
- 已记录：[Search-Codegen 深度研究闭环实验](search-codegen/README.md)（本次因 DashScope 凭据缺失未发起 API 调用，待配置后重跑）
- 进行中：阅读第 1 章其余内容并补充实验。
- 本章总结：待本章学习完成后填写。

## 本章实验索引

| 实验 | 模型/提供商 | 状态 | 结论摘要 |
| --- | --- | --- | --- |
| [Context：上下文消融实验](context/README.md) | DeepSeek / `deepseek-v4.1-flash` | 已完成 | 历史和工具结果缺失会导致重复调用或无法完成任务；本次未观察到去除已保留推理后的正确性退化。 |
| [Search-Codegen：搜索与代码生成](search-codegen/README.md) | DashScope / `qwen3.7-plus` | 待重跑 | 本次在 API 调用前检测到 `credential_missing`；已记录验收条件、失败边界与复现步骤。 |

## 本章总结

### 核心概念

- 待补充：上下文、轨迹、工具调用与工具结果之间的关系。

### 学习心得

- 待补充：完成本章全部实验后，归纳最重要的理解与局限。

### 后续问题

- 待补充。
