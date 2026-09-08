# 深度集成阿里云百炼 Token Plan：Qwen3.8-Flash 接入指南（精简版）

> ✅ 一键启动 | ⚠️ 关键警告 | 💰 高性价比 | 🔒 无精确用量 

## 1. 一句话总结（核心价值）

将阿里云百炼 Token Plan 订阅（含 qwen3.8-flash 模型）接入 DeepSeek Harness，可实现 成本降低约 ¥43.71/月，并利用其极低的缓存命中成本（约 10 Credits/百万 token）大幅提升 Agent 工作负载效率。但请务必正视其核心挑战：额度用量暂无精确确定值。

## 2. 为什么选择 Qwen3.8-Flash？（关键优势）

| 特性 | 说明 |
|---|---|
| ✅ 极致性价比 | 4 天内估算消耗 9,529 Credits（≈¥95.29），远超 ¥139 的月付成本，节省约 ¥43.71。 |
| ✅ 超低缓存成本 | 缓存命中单价仅约 10 Credits/百万 token，对依赖历史重放的 Agent 负载极为友好。 |
| ✅ 完全兼容 | 端点兼容 OpenAI 协议，无需改造现有代码，即插即用。 |
| ⚠️ 限制 | 无官方换算表、无对外查询接口、周额度不结转，导致无法精确监控用量。 |

## 3. 必须知道的三大警告（避坑指南）

> ❗ 警告 1：额度用量不可精确确定
> 阿里云控制台未公开 Credits 与 token（input/output/cache）的换算关系，也无对外用量查询接口。所有数字均为本地估算，仅供参考。建议设置 ≥70% 用量预警自动分流到免费池或等待窗口重置。
>
> ❗ 警告 2：外部数据必须净化
> 任何外部数据（如 GitHub 搜索结果）在进入对话前，必须通过净化管道（gh-search-safe.cjs）。否则会触发 data_inspection_failed 错误，导致会话永久报废。
>
> ❗ 警告 3：reasoningEffort 不支持
> qwen3.8-flash 不支持 high 档次。若在会话中选择 high，会报错 UNSUPPORTED_REASONING_EFFORT。请改用 medium 或 low 档次，或切换至 qwen3.8-max。

## 4. 快速开始（1分钟上手）

# 1. 准备凭据：从阿里云控制台获取 API Key
# 2. 在 ~/.dsh/.credentials.yaml 中添加：
TOKENPLAN_API_KEY: <您的API密钥>

# 3. 在 ~/.dsh/settings.yaml 中配置：
llm-pi-ai:
  providers:
    tokenplan:
      displayName: 阿里云 Token Plan
      apiKeyEnv: TOKENPLAN_API_KEY
      api: openai-completions
      baseURL: https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1
      models:
        - id: qwen3.8-flash
          name: Qwen3.8-Flash·TokenPlan

# 4. 设置默认模型：
agent-default-model:
  provider: tokenplan
  model: qwen3.8-flash

# 5. 启用外部数据管道（防止被拦截）：
gh api -X GET search/repositories -f q="关键词" | node dsh-moderation-fix/gh-search-safe.cjs

## 5. 详细文档与源码（进阶参考）

- 完整工作流文档: qwen-tokenplan-workflow.md
- GitHub 仓库: https://github.com/mengge237/dsh-qwen-tokenplan-workflow
- 相关技能: qwen-token-plan-30d, dsh-local-runbook, upstream-feedback-archive

> 📌 提示：本工作流基于本机实测，配置已脱敏。实际使用时请替换占位符为您的真实信息。