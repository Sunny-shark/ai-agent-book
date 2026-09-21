# 实验 2-1：本地 LLM 服务部署与工具调用

## 实验目的

复现 Chapter 2 的本地服务实验：让本地 `qwen3:0.6b` 通过 Ollama 在同一轮输出 Vancouver 的时间和天气两个工具调用，执行工具后将结果写回上下文，再由模型生成终止回复；同时以匹配样本比较 KV Cache 命中和未命中的首 token 延迟（TTFT）。

- 实验源码：[`chapter2/local_llm_serving/run_experiment.py`](../../../chapter2/local_llm_serving/run_experiment.py)
- 固化协议：[`chapter2/local_llm_serving/experiment_protocol.json`](../../../chapter2/local_llm_serving/experiment_protocol.json)

## 实验环境

| 项目 | 配置 |
| --- | --- |
| 实验日期 | 2026-09-21（Vancouver 工具返回日期） |
| 模型提供商 | 本地 Ollama，`http://localhost:11434` |
| 模型 | `qwen3:0.6b`，Q4_K_M，751.63M 参数 |
| 模型 digest | `7df6b6e09427a769808717c0a93cadc4ae99ed4eb8bf5ca557c90846becea435` |
| Ollama | 0.34.2 |
| Python | 3.13.11 |
| 操作系统 | Windows 11（AMD64） |
| Tokenizer | 本地缓存的 `Qwen/Qwen3-0.6B` tokenizer |
| 代码版本 | `4ac9c017e00a2943a5ba7164e88e34f955b27bcd` |

> README 推荐 Python 3.12；本次实际用 Python 3.13.11 成功执行。模型推理由本机完成，不使用云端模型 API，也没有记录 API Key。

## 实验过程

### 1. 准备本地运行环境

按照项目 README 安装 Ollama、拉取模型，并安装 runner 依赖：

```powershell
ollama pull qwen3:0.6b
python -m pip install "transformers>=4.40,<6" "PyPDF2>=3.0.0"
```

runner 为保证模板可审计，会用 `local_files_only=True` 载入 tokenizer。因此另行把 `Qwen/Qwen3-0.6B` 的 tokenizer 配置下载至本地临时目录，避免在正式运行时联网。

### 2. 执行完整实验

实际运行命令：

```powershell
cd chapter2\local_llm_serving
python run_experiment.py `
  --tokenizer /tmp/qwen3-0.6b-tokenizer `
  --output runs/exp2-1-qwen3-0.6b-20260921-local-v2
```

该命令运行以下两部分：

1. 以原始 Chat Template 发起“查询 Vancouver 当前时间和天气”的请求，检查首轮是否恰好调用 `get_current_time` 与 `get_current_temperature`；两个工具并行执行，结果加入第二轮上下文后，模型必须停止调用工具并生成答复。
2. 使用约 4096 token 的稳定系统提示做 2 次 warmup，再运行 5 对 cache hit/miss。hit 组保持完整提示逐字节相同；miss 组只在系统提示开头插入等长唯一标记，使整个前缀失效。

## 实验结果

### 工具调用闭环

| 验收项 | 结果 |
| --- | --- |
| Chat Template 特殊 token 与工具 schema 可见 | 通过 |
| 首轮含原始 `<tool_call>` 标签 | 通过 |
| 首轮恰好调用两个指定工具 | 通过 |
| Vancouver 参数正确 | 通过 |
| 两个工具并行执行且结果有效 | 通过 |
| 第二轮消费工具结果后终止 | 通过 |

首轮调用 `get_current_time(timezone="America/Vancouver")` 与 `get_current_temperature(location="Vancouver", unit="celsius")`。线程池总执行时间为 `0.0747s`；首轮 TTFT 为 `54.665s`（包含本地模型冷启动），第二轮 TTFT 为 `2.492s`。

天气工具在本次运行中无法访问 Open-Meteo，按源码的异常回退逻辑返回了 `12.4°C, rainy` 的**模拟数据**。这不影响“模型调用工具—结果写回—第二轮终止”的闭环验收，但不能把该温度当成真实天气观测。

### KV Cache 对照

| 指标 | Cache hit | Cache miss |
| --- | ---: | ---:|
| TTFT 样本（秒） | 2.099, 2.267, 2.282, 2.232, 2.243 | 2.322, 2.319, 2.529, 2.286, 2.319 |
| 平均 TTFT（秒） | **2.224** | **2.355** |

- miss / hit 为 `1.059`，即前缀失效组平均慢约 `5.9%`。
- 5/5 匹配样本中，稳定前缀的 hit 组均更快。
- 此结果支持“固定系统提示和工具定义、把动态信息后置”的设计原则；它只代表本机、该模型、该 Ollama 版本和这一次运行，不应外推为通用性能结论。

## 学习心得

1. **工具结果本身也是上下文。** 模型在第一轮只能规划调用；只有 Harness 将时间和天气结果追加到第二轮消息，模型才能据此完成回答并停止。
2. **上下文顺序会影响延迟。** 两组请求长度相同，只改系统提示开头，cache miss 仍在 5 个配对里全部更慢，说明复用的是字节级稳定前缀而非“语义相似”文本。
3. **应同时观察正确性和性能。** 本实验的工具闭环门槛全部通过，但天气数据来自回退路径；若只看最终回答，会遗漏外部观察是否真实这一关键质量维度。
4. **冷启动需与稳态分开报告。** 首轮 `54.665s` 明显高于后续缓存对照的约 2 秒 TTFT，不能将两者混为模型的稳定推理速度。

## 复现证据与完整性

- 原始逐轮证据：[`chapter2/local_llm_serving/runs/exp2-1-qwen3-0.6b-20260921-local-v2/evidence.json`](../../../chapter2/local_llm_serving/runs/exp2-1-qwen3-0.6b-20260921-local-v2/evidence.json)
- 完成清单：[`chapter2/local_llm_serving/runs/exp2-1-qwen3-0.6b-20260921-local-v2/manifest.json`](../../../chapter2/local_llm_serving/runs/exp2-1-qwen3-0.6b-20260921-local-v2/manifest.json)
- `evidence.json` SHA-256：`b0f6bed95721ca127035f9e1d7038e5476384738c67dab17d64085981177db3b`
- `manifest.json` 标记 `official_complete: true`，凭据扫描通过，且本地推理 API 成本为 `$0`。

上传前只暂存本实验的 Markdown 笔记和完整运行目录；不要上传 `.env`、模型权重、本地 tokenizer 缓存或含凭据的终端输出。
