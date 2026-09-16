# 实验 1-3：搜索与代码生成的 Deep Research 闭环

## 实验目的

本实验对应《深入理解 AI Agent》第 1 章的 `search-codegen` 项目。目标不是让模型在回答中声称“查过资料、运行过 Python”，而是通过 Responses API 让服务端实际完成两类托管工具调用：

1. 用 `web_search` 检索最新、可引用的外部信息；
2. 用 `code_interpreter` 对检索结果进行定量计算。

实验还检验一个交互要求：当“分析最近一个月比特币走势”没有指定数据源和指标时，Agent 必须先澄清需求，不能抢先调用工具；用户补充条件后，再借助 `previous_response_id` 延续同一任务。

## 本次运行环境

| 项目 | 配置 |
| --- | --- |
| 实验日期 | 2026-09-17（输出创建时间为 2026-09-16 UTC） |
| 操作系统 | Windows 11 |
| Python | 3.13.3 |
| 运行命令 | `python run_experiment_1_3.py --backends dashscope --reasoning high` |
| 目标提供商 | 阿里云百炼 DashScope Responses API |
| 目标模型 | `qwen3.7-plus` |
| 代码版本 | `6b69749427100e86771ace8538dc79036f2de82f` |

项目的官方参考路径是 OpenAI `gpt-5.6-sol`；README 也允许使用 DashScope 的等价托管 `web_search` 与 `code_interpreter` 路线进行验收。OpenRouter 仅用于诊断，不能作为通过实验的依据。

## 实验过程

运行器会依次执行两个场景：

| 场景 | 预期行为 | 关键验收证据 |
| --- | --- | --- |
| 东盟首都距离 | 搜索十个东盟国家首都的坐标，使用 Haversine 公式枚举全部 45 对，再找出最近的一对 | 已完成的 `web_search_call`、`code_interpreter_call`、可点击 URL 引用，以及正确的最近首都对 |
| 比特币技术分析 | 第一轮先询问数据源和指标；补充条件后续接并计算 MA7/MA20、RSI14、MACD、收益率和最大回撤 | 首轮没有工具调用、`previous_response_id` 延续关系、后续两类托管工具回执和指标报告 |

## 本次输出与分析

本次输出位于 `chapter1/search-codegen/validation/runs/real_20260916T173021Z/`。结论是**未完成复现，而不是实验不通过**：

| 检查项 | 本次结果 |
| --- | --- |
| DashScope 后端是否启动 | 否 |
| 错误 | `credential_missing` |
| `web_search_call` | 0 |
| `code_interpreter_call` | 0 |
| URL 引用 | 0 |
| token 用量 | 0 |
| `acceptance_passed` | `false` |

这说明运行器在发送请求前就发现 `DASHSCOPE_API_KEY` 未配置或未被当前进程读取，因此没有产生任何模型回答或工具轨迹。这里不能根据“预期最近首都对”为吉隆坡—新加坡，推断本次模型已经得到了该结论；该坐标对仅是运行器内置的独立参考值，用于之后核验真实工具执行的结果。

项目仓库中保留的 2026-07-31 历史证据显示，DashScope 的成功运行曾包含一轮托管网页搜索和代码解释器调用，并得到吉隆坡—新加坡约 316.35 km；比特币场景也曾先澄清、再继续执行。那是项目维护者提供的参考证据，不是本次本机运行的结果，不能替代本次复现。

## 学习心得

1. **工具声明不等于工具执行。** 请求中带有 `web_search` 和 `code_interpreter` 的定义，并不证明模型真的调用了它们。应以返回中的、状态为 `completed` 的工具回执为准。
2. **深度研究是可审计的闭环。** 搜索负责取得可追溯的事实，代码负责让计算可重复；引用、工具轨迹和最终结论共同构成证据链，缺任一项都无法可靠验收。
3. **澄清需求本身是 Agent 的行动。** 模糊的金融分析任务如果直接检索或计算，可能选错数据源、时间范围和指标。先澄清再用 `previous_response_id` 续接，能同时避免无效调用和丢失会话状态。
4. **失败输出也有诊断价值。** 本次的 `credential_missing` 将问题限定在凭据配置或环境加载，而非模型能力、提示词、网络检索或代码解释器。这比把“没有答案”笼统归因于模型失败更可操作。
5. **实验结论必须区分来源。** README 中的成功回执可以帮助理解验收标准，但学习记录应把“历史参考成功”和“本人本次未发起 API 调用”明确分开，避免把别人的结果当作自己的复现实证。

## 下一步复现

在 `chapter1/search-codegen/.env` 中配置有效凭据（不要提交该文件）：

```env
DASHSCOPE_API_KEY=你的百炼_API_Key
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
DASHSCOPE_MODEL=qwen3.7-plus
```

然后重新运行：

```powershell
cd D:\Ai-agent\ai-agent-book\chapter1\search-codegen
python run_experiment_1_3.py --backends dashscope --reasoning high
```

完成后应检查新生成的 `evidence.json`：`acceptance.passed` 应为 `true`，并且两个场景都应包含真实的托管工具回执。再运行：

```powershell
python -m pytest -q test_responses_agent.py
python -m py_compile agent.py config.py main.py run_experiment_1_3.py
```

> API Key 仅保存在本地 `.env`。提交前需检查 `git diff --cached`，确认没有将凭据、Authorization 请求头或其他敏感信息加入暂存区。
