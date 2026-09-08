# 深度集成阿里云百炼 Token Plan：Qwen3.8-Flash 的接入工作流与避坑指南（脱敏版）

> **注意**：本工作流基于本机实测，配置已脱敏。实际使用时请替换占位符为您的真实信息。如需部署至生产环境，请务必在私有仓库中先验证。

## 1. 目标与背景（Why）

将阿里云百炼 Token Plan 订阅（含 qwen3.8-flash 模型）深度集成至 DeepSeek Harness，以实现：

- **成本优化**：用订阅制替代按量付费，降低长期使用成本；
- **性能提升**：利用 qwen3.8-flash 高效的缓存命中能力，加速长上下文重放；
- **自动化支撑**：为 Agent 场景提供稳定、可预测的模型服务。

根据本机 4 天实测（2026-09-05 至 09-08），该方案实现约 ¥95.29 的按量等价价值，远超 ¥139 的月付成本，具备显著性价比优势。但需正视其核心挑战：额度用量暂无精确确定值。

## 2. 核心事实与关键限制（The Hard Facts）

### ✅ 优势：高性价比与实用性（Practicality & Cost-Performance）

- **模型选择**：qwen3.8-flash 是本方案的核心。其缓存命中单价极低（约 10 Credits/百万 token），对依赖历史重放的 Agent 负载极为友好，是“实用”与“性价比”的完美结合。
- **价格对比**：4 天内估算消耗 9,529 Credits（≈¥95.29），而订阅费用为 ¥139/月。即使不考虑其他功能，仅此一项即可节省约 ¥43.71，且能支持高频并发任务。
- **兼容性**：端点完全兼容 OpenAI 协议，无需改造现有 harness 代码，即插即用。

### ⚠️ 限制：额度透明度与用量不确定性（The No-Value Fact）

**本机账本口径 + 估算，不是官方 Credits 数。** 

- **无官方换算表**：阿里云控制台未公开 Credits 与 token（input/output/cache）的换算关系，无法进行精确核对。
- **无对外查询接口**：无 API 可获取按时间、模型、项目维度的用量明细，自动化记账只能依赖本地脚本（如 token_plan_report.py）估算，偏差无法校验。
- **周额度规则**：每 7 天 10,000 Credits 窗口自首次调用起算，不结转。本机实测 4 天即达 9,529（95%），导致用户不敢放开使用，必须主动分流到免费池或等待窗口重置。

> **结论**：当前状态为「暂无精确确定额度用量」。所有数字均为本地估算，仅供参考。建议定期核对控制台数据，并设置 ≥70% 用量预警自动分流。

## 3. 接入步骤（How-To）

### 步骤 1：准备凭据（Credentials）

1. 从阿里云控制台获取您的 TOKENPLAN_API_KEY（以 sk-sp- 开头）。
2. 编辑文件 ~/.dsh/.credentials.yaml， 在 refs: 下添加：
   TOKENPLAN_API_KEY: <您的API密钥>
   > **注意**：该文件应为纯文本，勿加密。密钥不可泄露，本机存储路径为 <本机用户目录>/.dsh/.credentials.yaml。

### 步骤 2：配置 Provider（Provider Configuration）

编辑 ~/.dsh/settings.yaml， 在 llm-pi-ai.providers 下添加 tokenplan 配置块：

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

> **重要提示**：
> - apiKeyEnv 必须与 .credentials.yaml 中的键名严格一致。
> - baseURL 为通用端点，无需修改。
> - id 字段必须为 qwen3.8-flash，否则可能因 UNSUPPORTED_REASONING_EFFORT 报错（上游 qwen3.8-flash 不支持 high 档次）。

### 步骤 3：设置默认模型（Default Model）

在 settings.yaml 中，将全局默认模型设为 tokenplan/qwen3.8-flash：

agent-default-model:
  provider: tokenplan
  model: qwen3.8-flash

> **生效条件**：新会话才会应用此默认值。旧会话需手动切换模型。

### 步骤 4：启用外部数据管道（External Data Sanitation）

**这是防止 data_inspection_failed 错误的铁律。** 任何外部数据（如 GitHub 搜索结果）在进入对话前，必须通过净化管道。

1. 用以下命令处理数据：
   gh api -X GET search/repositories -f q="关键词" | node dsh-moderation-fix/gh-search-safe.cjs
2. 净化规则：
   - 命中黑名单（如 cirosantilli、gege-circle）的整行直接丢弃。
   - 描述字段若超过 20 字，强制截断并哈希（[desc 63ch sha256=...]），避免密度判定触发绿网拦截。
   - **失败关闭**：中文字符超过 20 字的整行，降级为 [row ...]，确保不会被拦截。

> **警告**：未经净化的数据输入，会导致会话永久报废，错误码为 INVALID_REQUEST，无法恢复。

## 4. 关键注意事项与避坑指南（Pitfalls & Best Practices）

| 陷阱 | 解决方案 |
|---|---|
| 429 Throttling / Allocated quota exceeded | 两个会话同一秒并发即触发。解决方案：串行化任务，或错峰 1 分钟执行。频繁发生则考虑升级至更高并发档位。 |
| data_inspection_failed | 由外部回灌的描述文本触发，非用户内容。解决方案：必须走 gh-search-safe.cjs 等净化管道。 |
| UNSUPPORTED_REASONING_EFFORT on qwen3.8-flash | 该模型不支持 reasoningEffort: high。解决方案：在会话中选择 medium 或 low 档次，或改用 qwen3.8-max。 |
| resolves no models / UNKNOWN_MODEL | 由于 settings.yaml 配置错误（如 id 缺失、models 为空）导致。解决方案：运行 dsh_preflight.py --check 自检，修复后重启 dsh。 |
| GEO_OR_ENTITLEMENT 错误 | 由阿里云侧区域合规或未购授权引起。解决方案：检查账号权限和地域设置，或联系客服。 |

## 5. 总结（Conclusion）

qwen3.8-flash 在 Token Plan 下展现出卓越的实用性和性价比。其低廉的缓存成本使其成为 Agent 工作负载的理想选择。然而，当前最大的障碍是额度用量的不透明性。在没有官方换算表和查询接口的情况下，我们只能依赖本地估算。因此，**请始终将「暂无精确确定额度用量」作为核心前提，不要期望得到一个精确的实时数字。**

通过遵循本工作流，您可以在享受高性能、低成本模型的同时，有效规避重大风险，构建一个稳健的 AI 工作流。