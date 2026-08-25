---
name: litellm-config-update
description: 当 ModelScope 模型列表变化时，按新闻早中晚报的经济性优先策略更新 LiteLLM config.yaml：先用免费强模型，再用免费小参数/快速模型，最后才用收费供应商兜底。
---

# LiteLLM Config Update

Use this skill when updating `/home/ubuntu/litellm/config.yaml` model routing, especially after refreshing the ModelScope `/v1/models` list or when the user asks to repartition `pro-model`, `flash-model`, or `backup-model`.

## User Intent

This LiteLLM instance is mainly for daily Chinese news morning/noon/evening reports. Reliability requirements are moderate; economy is the priority.

The model-group semantics are:

- `pro-model`: free strong models first, primarily ModelScope API-Inference.
- `flash-model`: free smaller/faster models after strong free models are unavailable, primarily ModelScope API-Inference.
- `backup-model`: paid or non-free providers last, used only after free routes fail.

Keep this ordering unless the user explicitly changes the objective.

## ModelScope Constraints

ModelScope API-Inference is a free, non-commercial, non-SLA service with dynamic quota and concurrency limits. It is intended for developer experience, not high-concurrency online production.

Operational rules:

- Treat ModelScope as economical best-effort capacity.
- Prefer single-concurrency usage: keep `rpm: 1` and `max_parallel_requests: 1` for ModelScope deployments unless the user explicitly asks otherwise.
- Use only exact model IDs returned by `https://api-inference.modelscope.cn/v1/models` for the configured account.
- In LiteLLM, write ModelScope models as `openai/<exact_model_id>` with `api_base: https://api-inference.modelscope.cn/v1/` and `api_key: os.environ/MODELSCOPE_API_KEY`.
- Do not invent aliases or shortened names. For example, if `/v1/models` returns `deepseek-ai/DeepSeek-V4-Flash-0731`, do not configure `deepseek-ai/DeepSeek-V4-Flash`.
- Remove or replace ModelScope models that no longer appear in `/v1/models`; they commonly fail with `has no provider supported`.

## Group Assignment Rules

### `pro-model`

Use free ModelScope models with stronger expected quality for Chinese news summarization, synthesis, and reasoning. Prefer large instruction models and strong current flagship models.

Good candidates from prior availability checks included:

- `ZhipuAI/GLM-5.2`
- `Qwen/Qwen3-235B-A22B-Instruct-2507`
- `deepseek-ai/DeepSeek-V4-Pro-0813`
- `deepseek-ai/DeepSeek-V4-Pro`
- `MiniMax/MiniMax-M3`

Keep weights conservative enough that one weak or dynamically limited model does not dominate. If uncertain, use roughly equal weights among the best 3-5 models.

### `flash-model`

Use free ModelScope models that are smaller, faster, cheaper, or better suited for fallback summaries and rewrites. These should be lower-cost alternatives after `pro-model` is exhausted or limited.

Good candidates from prior availability checks included:

- `deepseek-ai/DeepSeek-V4-Flash-0731`
- `Qwen/Qwen3-Next-80B-A3B-Instruct`
- `Qwen/Qwen3-30B-A3B`
- `Qwen/Qwen3-14B`
- `Qwen/Qwen3-8B`
- `stepfun-ai/Step-3.5-Flash`
- `stepfun-ai/Step-3.7-Flash`
- `meituan-longcat/LongCat-Flash-Lite`
- `ZhipuAI/GLM-4.7-Flash`

A ModelScope model may appear in both `pro-model` and `flash-model` only when there is a deliberate reason, but avoid pointless duplication because provider-level limits are shared.

### `backup-model`

Use paid or non-free providers only. This group exists to spend money only after the free ModelScope route fails.

Typical backup providers in this repo include:

- Z.ai OpenAI-compatible endpoints using `ZAI_API_KEY`
- SiliconFlow OpenAI-compatible endpoints using `SILICONFLOW_API_KEY`
- custom GPT endpoint using `CUSTOM_GPT_API_KEY`

Do not add ModelScope deployments to `backup-model` under the current cost-first policy unless the user explicitly requests it.

## Fallback Policy

Preserve this order unless the user asks to change it:

```yaml
litellm_settings:
  fallbacks:
    - pro-model: ["flash-model", "backup-model"]
    - flash-model: ["backup-model"]
```

This means: free strong models -> free small/fast models -> paid providers.

## Update Workflow

When asked to update config from a new ModelScope list:

1. Query ModelScope `/v1/models` with the configured `MODELSCOPE_API_KEY`, or use the model list supplied by the user.
2. Compare the returned exact IDs against current `config.yaml` ModelScope deployments.
3. Identify invalid ModelScope IDs currently configured and propose replacements.
4. Assign candidates using the group rules above.
5. Present a concise diff or proposed model-group list before changing the file if the user asked for advice only.
6. If the user asked to apply changes, edit only `config.yaml` unless another file is clearly required.
7. Validate YAML syntax after editing.
8. Restart LiteLLM only after the user has asked for changes to take effect or has already implied operational application.
9. Verify live `/model/info` after restart when possible.

Never expose API keys in the response. If command output includes secrets, summarize with redaction.

## Verification

Minimum verification after editing:

```bash
python3 -c "import yaml, pathlib; data=yaml.safe_load(pathlib.Path('config.yaml').read_text()); print(type(data).__name__); print(len(data['model_list']))"
```

For live verification after restart, query `/model/info` with the LiteLLM master key and confirm the expected models appear in the expected groups.
