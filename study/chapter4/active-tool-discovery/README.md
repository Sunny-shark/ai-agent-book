# 实验 4-1：主动工具发现

## 实验目的

本实验比较三种工具组织方式：全量注入、一次性检索预筛选和主动工具发现。核心问题是：当 Agent 拥有上百个工具时，是否有必要在每一轮都把全部工具 schema 放入系统提示，以及按需发现工具能否在保留任务能力的同时降低上下文成本。

- 实验源码：[`chapter4/active-tool-discovery`](../../../chapter4/active-tool-discovery/)
- 正式协议：[`experiment_protocol.json`](../../../chapter4/active-tool-discovery/experiment_protocol.json)
- 本次正式运行：[`local_20260928`](../../../chapter4/active-tool-discovery/validation/experiment_4_1/local_20260928/)

## 实验环境

| 项目 | 配置 |
| --- | --- |
| 实验日期 | 2026-09-28 至 2026-09-29 |
| 操作系统 | Windows 11 |
| 本地模型服务 | Ollama 0.34.2，`http://127.0.0.1:11434` |
| 模型 | `qwen3:4b`，Q4_K_M |
| GPU | NVIDIA GeForce RTX 5060 Laptop GPU，8 GB |
| 工具服务 | perception-tools MCP，stdio 传输 |
| 工具目录 | 127 个唯一工具，完整 schema 为 50,597 tokens |
| 检索模型 | `sentence-transformers/all-MiniLM-L6-v2`，本地 CPU，384 维 |

模型、嵌入索引和工具调用均在本机运行；工具执行会访问 Yahoo Finance、DuckDuckGo、GitHub 等公开接口。实验目录未保存 API Key、Authorization 请求头或其他私密凭据。

## 实验过程

### 1. 离线机制自检

先运行无需凭据的机制实验：

```powershell
cd chapter4\active-tool-discovery
python demo.py --offline --output validation/offline-local-20260923.json
```

该路径使用本地哈希嵌入和脚本化模型，适合检查三种策略的工具暴露方式、token 统计和任务流转。它不能作为真实模型能力证据。

### 2. 准备正式实验

安装 MCP 工具依赖，启动 Ollama，并缓存正式协议指定的模型：

```powershell
python -m pip install -r ..\perception-tools\requirements.txt
ollama pull qwen3:4b
```

随后缓存 `sentence-transformers/all-MiniLM-L6-v2`。正式 runner 使用 `local_files_only=True`，运行期间不会自动切换到网络嵌入或其他模型。

### 3. 运行真实 MCP campaign

初始运行命令：

```powershell
python run_exact_experiment.py --campaign-id local_20260928
```

运行中加入了两项可审计的恢复措施：

1. MCP 工具调用超过 120 秒时写入失败收据，让 Agent 可以改选工具，而不是永久阻塞。
2. 使用 `--resume` 保留已完成任务；根据人工决定跳过持续阻塞的 arXiv 任务，并保留 GitHub 对照组的失败记录。

最终续跑命令使用：

```powershell
$env:SKIP_TASK_IDS='transformer_arxiv_download'
$env:SKIP_CONTROL_TASK_IDS='github_contributors_visualization'
python run_exact_experiment.py --campaign-id local_20260928 --resume
```

## 实验结果

### 离线机制自检

| 策略 | 精确选择 | 完成任务 | 总注入 tokens | 平均注入 tokens |
| --- | ---: | ---: | ---: | ---: |
| 全量注入 | 8/8 | 8/8 | 93,040 | 11,630.0 |
| 检索预筛选 | 4/8 | 4/8 | 8,236 | 1,029.5 |
| 主动发现 | 8/8 | 8/8 | 7,796 | 974.5 |

主动发现的总注入量约为全量注入的 `1/11.9`，同时在脚本化路由条件下保持 8/8 完成。一次性预筛选虽然同样节省 token，却在 4 个跨领域任务中漏掉后续步骤需要的工具。

### 真实模型与真实工具运行

| 组别 | 任务 | 工具选择准确率 | 是否完成 | 初始 system tokens | 动态 schema tokens | 耗时 |
| --- | --- | ---: | --- | ---: | ---: | ---: |
| 对照组 | Apple 股价与新闻 | 1.0 | 是 | 50,829 | 0 | 857.089s |
| 主动发现 | Apple 股价与新闻 | 1.0 | 否 | 1,251 | 3,939 | 820.342s |
| 主动发现 | GitHub contributors 可视化 | 1.0 | 是 | 1,251 | 4,785 | 628.261s |

真实运行验证了以下事实：

- 对照组确实把 127 个完整工具 schema 放入系统提示，单任务初始提示达到 50,829 tokens。
- 主动发现组初始提示只有 1,251 tokens，再按需求分别追加相关 schema。
- 对照组 Apple 任务成功调用 `stock_price` 与 `search_news`，得到真实 Yahoo Finance 和 DuckDuckGo 回执。
- 主动发现 Apple 任务找到了正确工具，但 `yfinance_quote` 遇到 Yahoo 429 限流，后续新闻工具也没有产生可验收结果，因此保存为失败。
- 主动发现 GitHub 任务成功调用 `github_list_contributors` 和本地 `code_interpreter`，生成了 [`contributors.svg`](../../../chapter4/active-tool-discovery/validation/experiment_4_1/local_20260928/treatment/github_contributors_visualization/contributors.svg)。

### 验收结论

本次 `summary.json` 的最终状态为 **`failed`**。这不表示实验代码无法运行，而是表示本次运行没有满足正式协议要求的“两个组各完成三个任务”：

- `transformer_arxiv_download` 按人工决定在两个组中跳过；
- GitHub 对照组在拿到真实 contributors 数据后没有产生可视化输出，失败记录被保留且不再重试；
- 主动发现 Apple 任务因公开服务限流和检索失败未完成。

因此，本次成果应表述为：**离线机制实验完整通过，真实 MCP campaign 部分跑通并保留了真实成功、失败与跳过证据，但未通过完整正式验收。**

## 学习心得

1. **工具 schema 也是上下文成本。** 工具数量增加时，全量注入会在每轮重复处理大量与当前步骤无关的 schema；本次真实对比为 50,829 对 1,251 个初始 tokens。
2. **主动发现更适合多阶段任务。** Agent 可以在遇到“股票数据”“仓库元数据”“本地绘图”等能力缺口时分别检索，而不必在任务开始前准确预测全部工具。
3. **选对工具不等于任务完成。** 主动发现 Apple 轨迹的工具选择准确率为 1.0，但公开接口的 429 和检索失败仍让任务无法完成。Agent 评测必须同时检查工具选择、执行回执、最终答案和产物。
4. **失败轨迹需要保留。** 超时、限流、提前结束和没有产物都属于真实 Agent 系统的一部分。删除失败重跑容易制造“只展示成功样本”的偏差。
5. **正式验收与机制理解应分开。** 离线实验能稳定说明上下文组织和检索时机；真实实验则额外受到弱模型行为、网络、公共 API 稳定性和本机性能影响，两类证据不能互相替代。

## 复现证据与完整性

- 离线结果：[`offline-local-20260923.json`](../../../chapter4/active-tool-discovery/validation/offline-local-20260923.json)
- 正式汇总：[`summary.json`](../../../chapter4/active-tool-discovery/validation/experiment_4_1/local_20260928/summary.json)
- 文件清单：[`manifest.json`](../../../chapter4/active-tool-discovery/validation/experiment_4_1/local_20260928/manifest.json)
- 对照组失败记录：[`failure.json`](../../../chapter4/active-tool-discovery/validation/experiment_4_1/local_20260928/control/github_contributors_visualization/failure.json)
- `summary.json` SHA-256：`8e5408ca9ec6308b02dd7b3e968f6fb8bab28af2fa68306e07923b869141449a`
- `manifest.json` SHA-256：`23fdf60e672bc48c78992b8d1c5e9ba1eaa1984f4b9dbbbc2b30cd2d50cd1978`

上传时应提交笔记、runner 的超时/跳过支持以及完整 campaign 证据；不要提交 Ollama 模型文件、Hugging Face 缓存、`.env` 或终端中的私人代理配置。
