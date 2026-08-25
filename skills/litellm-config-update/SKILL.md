---
name: litellm-config-update
description: 当 ModelScope 模型列表变化时，按新闻早中晚报的经济性优先策略更新 LiteLLM config.yaml：先用免费强模型，再用免费小参数/快速模型，最后才用收费供应商兜底。
---

# LiteLLM 配置更新

当需要更新 `/home/ubuntu/litellm/config.yaml` 的模型路由时使用本技能，尤其适用于刷新 ModelScope `/v1/models` 列表之后，或用户要求重新分配 `pro-model`、`flash-model`、`backup-model` 时。

## 用户目标

这个 LiteLLM 实例主要用于每日中文新闻早报、午报、晚报。可靠性要求中等，经济性优先。

模型组语义如下：

- `pro-model`：优先使用免费强模型，主要来自 ModelScope API-Inference。
- `flash-model`：免费强模型不可用后，使用免费小参数/快速模型，主要来自 ModelScope API-Inference。
- `backup-model`：最后才使用收费或非免费供应商，只在免费路线失败后兜底。

除非用户明确改变目标，否则保持这个顺序。

## ModelScope 约束

ModelScope API-Inference 是免费、非商业化、无 SLA 的服务，并且会根据平台压力动态调整额度和并发限制。它适合开发者体验和低并发任务，不适合高并发线上生产。

操作规则：

- 将 ModelScope 视为经济优先的尽力而为容量。
- ModelScope 部署优先保持单并发：除非用户明确要求，否则保留 `rpm: 1` 和 `max_parallel_requests: 1`。
- 只使用当前账号调用 `https://api-inference.modelscope.cn/v1/models` 返回的精确模型 ID。
- 在 LiteLLM 中，ModelScope 模型写成 `openai/<精确模型ID>`，并使用 `api_base: https://api-inference.modelscope.cn/v1/` 和 `api_key: os.environ/MODELSCOPE_API_KEY`。
- 不要自造别名或缩写模型名。例如 `/v1/models` 返回 `deepseek-ai/DeepSeek-V4-Flash-0731` 时，不要配置成 `deepseek-ai/DeepSeek-V4-Flash`。
- 如果当前配置中的 ModelScope 模型不再出现在 `/v1/models` 返回里，应移除或替换；这类模型通常会报 `has no provider supported`。

## 模型组分配规则

### `pro-model`

放置预期质量更高的免费 ModelScope 模型，用于中文新闻摘要、综合写作和推理。优先选择大型指令模型和当前旗舰模型。

之前可用性检查中的较好候选包括：

- `ZhipuAI/GLM-5.2`
- `Qwen/Qwen3-235B-A22B-Instruct-2507`
- `deepseek-ai/DeepSeek-V4-Pro-0813`
- `deepseek-ai/DeepSeek-V4-Pro`
- `MiniMax/MiniMax-M3`

权重应保持保守，避免某个弱模型或动态受限模型占据过多流量。如果不确定，优先在 3-5 个最强候选之间使用接近的权重。

### `flash-model`

放置免费、较小、较快或更适合兜底摘要/改写的 ModelScope 模型。这一组用于 `pro-model` 额度不足、限流或失败后的低成本替代。

之前可用性检查中的较好候选包括：

- `deepseek-ai/DeepSeek-V4-Flash-0731`
- `Qwen/Qwen3-Next-80B-A3B-Instruct`
- `Qwen/Qwen3-30B-A3B`
- `Qwen/Qwen3-14B`
- `Qwen/Qwen3-8B`
- `stepfun-ai/Step-3.5-Flash`
- `stepfun-ai/Step-3.7-Flash`
- `meituan-longcat/LongCat-Flash-Lite`
- `ZhipuAI/GLM-4.7-Flash`

只有在有明确理由时，才让同一个 ModelScope 模型同时出现在 `pro-model` 和 `flash-model`。避免无意义重复，因为同一供应商的限制通常是共享的。

### `backup-model`

只放收费或非免费供应商。这一组的目的，是在免费 ModelScope 路线全部失败后才开始花钱。

本仓库中常见的 backup 供应商包括：

- 使用 `ZAI_API_KEY` 的 Z.ai OpenAI 兼容端点
- 使用 `SILICONFLOW_API_KEY` 的 SiliconFlow OpenAI 兼容端点
- 使用 `CUSTOM_GPT_API_KEY` 的自定义 GPT 端点

在当前经济性优先策略下，不要把 ModelScope 部署加入 `backup-model`，除非用户明确要求。

## Fallback 策略

除非用户要求修改，否则保持以下顺序：

```yaml
litellm_settings:
  fallbacks:
    - pro-model: ["flash-model", "backup-model"]
    - flash-model: ["backup-model"]
```

含义是：免费强模型 -> 免费小参数/快速模型 -> 收费供应商。

## 更新流程

当用户要求根据新的 ModelScope 列表更新配置时：

1. 使用已配置的 `MODELSCOPE_API_KEY` 查询 ModelScope `/v1/models`，或使用用户提供的模型列表。
2. 将返回的精确模型 ID 与当前 `config.yaml` 中的 ModelScope 部署对比。
3. 找出当前配置中无效的 ModelScope ID，并提出替换方案。
4. 按上面的模型组规则分配候选模型。
5. 如果用户只是要求建议，先给出简明 diff 或模型组列表，不直接修改文件。
6. 如果用户要求应用变更，只编辑 `config.yaml`，除非明确需要其他文件。
7. 编辑后验证 YAML 语法。
8. 只有在用户要求生效或已经明确暗示要应用到运行服务时，才重启 LiteLLM。
9. 重启后尽量查询实时 `/model/info`，确认模型出现在预期模型组中。

不要在回复中暴露 API key。如果命令输出包含密钥，应只给出脱敏摘要。

## 验证

编辑后至少运行：

```bash
python3 -c "import yaml, pathlib; data=yaml.safe_load(pathlib.Path('config.yaml').read_text()); print(type(data).__name__); print(len(data['model_list']))"
```

如果已经重启服务，还应使用 LiteLLM master key 查询 `/model/info`，确认预期模型已加载到预期模型组。
