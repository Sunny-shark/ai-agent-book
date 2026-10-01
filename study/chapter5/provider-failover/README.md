# 实验 5-1：跨厂商的轨迹接管

## 实验目的

本实验验证一条执行到一半的 Agent 轨迹能否从一个模型厂商迁移到另一个厂商继续完成。实验比较三种轨迹处理方式：

- `naive`：原样转交思考内容和厂商凭证；
- `strip`：删除全部思考与凭证；
- `neutral`：使用中立轨迹格式保存工具调用与结果，只迁移可读内容，丢弃厂商专属凭证并重建调用 ID。

实验任务需要依次获取机票、住宿、餐费和汇率数据。在完成两个工具调用后注入连续的 429/503 故障，再切换到另一家模型继续执行。

- 实验源码：[`chapter5/provider-failover`](../../../chapter5/provider-failover/)
- 书中协议：[`book/chapter5.md`](../../../book/chapter5.md)

## 实验环境

| 项目 | 配置 |
| --- | --- |
| 实验日期 | 2026-10-01 |
| 操作系统 | Windows 11 |
| Python | 3.13.11 |
| HTTP 客户端 | `requests` 2.34.2 |
| 测试框架 | `pytest` 9.1.1 |
| 计划模型 | Kimi `kimi-k3`、Gemini `gemini-3.5-flash`、Anthropic `claude-haiku-4-5-20251001` |
| 实际尝试 | Kimi ↔ Gemini 两个方向，每个方向三种策略，共 6 个单元 |

正式协议需要三家厂商凭据，从而覆盖六种切换方向。本机没有 Anthropic 凭据，因此本次只能尝试 Kimi 与 Gemini 子集，不能视为完整的 18 单元正式复现。

## 实验过程

### 1. 离线测试

安装依赖后执行：

```powershell
cd chapter5\provider-failover
python -m pytest tests -q
```

结果为 `23 passed in 1.85s`。离线测试覆盖：

- 外来厂商凭证的保留与删除；
- 中立格式将思考转为普通文本；
- Gemini 工具调用的降级处理；
- 工具调用 ID 重铸与结果配对；
- 流式响应切断与参数拼接；
- 重复副作用指纹和确定性工具结果。

### 2. 首次真实运行

运行两厂商子集：

```powershell
python run_handoff.py `
  --pairs kimi:gemini gemini:kimi `
  --out validation/runs/exp5-1-handoff-local-20261001
```

6 个单元都在首次请求前终止。原因是 runner 只读取进程环境变量，不会自动加载仓库根目录的 `.env`。该失败没有被覆盖，而是作为环境加载失败证据保留。

### 3. 显式加载变量后重试

第二次运行只把本地 `.env` 中与 Kimi、Gemini 相关的变量加载到当前进程，并写入独立目录：

```powershell
python run_handoff.py `
  --pairs kimi:gemini gemini:kimi `
  --out validation/runs/exp5-1-handoff-local-20261001-v2
```

随后生成汇总：

```powershell
python summarize.py validation/runs/exp5-1-handoff-local-20261001-v2
```

## 实验结果

### 离线机制

离线测试 23/23 通过，说明中立轨迹、厂商渲染、凭证处理、ID 重铸和失败判定逻辑在本地按预期工作。

### 真实 API 运行

| 源厂商 | 目标厂商 | 策略 | 源厂商首个请求 | 是否进入接管 | 数据完整 | 答案正确 |
| --- | --- | --- | --- | --- | --- | --- |
| Kimi | Gemini | `naive` | HTTP 401 `Invalid Authentication` | 否 | 否 | 否 |
| Kimi | Gemini | `strip` | HTTP 401 `Invalid Authentication` | 否 | 否 | 否 |
| Kimi | Gemini | `neutral` | HTTP 401 `Invalid Authentication` | 否 | 否 | 否 |
| Gemini | Kimi | `naive` | HTTP 400 `API_KEY_INVALID` | 否 | 否 | 否 |
| Gemini | Kimi | `strip` | HTTP 400 `API_KEY_INVALID` | 否 | 否 | 否 |
| Gemini | Kimi | `neutral` | HTTP 400 `API_KEY_INVALID` | 否 | 否 | 否 |

第二次运行确实到达厂商 API，并保留了厂商返回的原始错误体。由于 Kimi 和 Gemini 的本地凭据均已失效，所有单元都在源厂商第一步终止，没有完成两个工具调用，也没有触发故障注入和跨厂商切换。因此：

- `handoff_ok`：三种策略均为 0/2；
- `data_complete`：三种策略均为 0/2；
- `answer_correct`：三种策略均为 0/2；
- 输入与输出 token 均为 0；
- 无法根据本次运行比较 `naive`、`strip` 和 `neutral` 的接管效果。

## 验收结论

本次实验结果为 **失败**，失败阶段是凭据鉴权，而不是轨迹转换或接管阶段。

本次结果只能支持以下结论：

1. 离线实现与测试全部通过；
2. runner 成功向真实厂商端点发送请求并保存真实 401/400 错误；
3. 本地凭据失效，导致任务尚未形成可供迁移的中间轨迹；
4. 未覆盖 Anthropic，也未满足“三条策略各覆盖六种厂商组合”的正式门槛。

仓库自带的历史 canonical evidence 显示中立策略曾在 6/6 厂商组合中切换成功，但那是维护者在 2026-08-24 产生的参考证据，不属于本次本机运行结果。

## 学习心得

这次实验让我认识到，Agent 的可迁移性不仅取决于消息文本，还取决于厂商专属的思考凭证、工具调用格式和调用 ID。中立轨迹的意义是把可移植的语义与不可移植的协议细节分开保存，切换厂商时再重新渲染。实验也提醒我，恢复机制本身必须建立在可用的基础设施上；凭据失效时，系统甚至无法产生可恢复的中间状态。因此，实验记录不能只保留成功轨迹，环境加载错误和真实鉴权失败同样是定位问题的重要证据。

## 复现证据与完整性

- 环境变量未加载的首次失败：[`exp5-1-handoff-local-20261001`](../../../chapter5/provider-failover/validation/runs/exp5-1-handoff-local-20261001/)
- 真实厂商鉴权失败：[`exp5-1-handoff-local-20261001-v2`](../../../chapter5/provider-failover/validation/runs/exp5-1-handoff-local-20261001-v2/)
- 第二次运行汇总：[`manifest.json`](../../../chapter5/provider-failover/validation/runs/exp5-1-handoff-local-20261001-v2/manifest.json)
- 首次运行 manifest SHA-256：`70f29d8ac32aacaef23fe089a99f6ba418457e53e3cdad9b96870148908095aa`
- 第二次运行 manifest SHA-256：`ed01d044523149ffd8b1649f3ee138471879aff0344967c1b3b7e730a00c72f0`

证据文件只保存请求体、响应体和评测结果，不保存 HTTP Authorization 或 API Key。重新复现时需要配置三家有效凭据，并重新运行完整六方向实验。
