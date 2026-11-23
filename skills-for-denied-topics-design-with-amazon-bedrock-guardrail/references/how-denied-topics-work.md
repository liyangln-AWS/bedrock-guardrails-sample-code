# Denied Topics 的机制与限制

## 主题的组成和字段限制

| 字段 | 限制 | 说明 |
|---|---|---|
| name | 1 到 100 字符，只允许英文字母、数字、空格和 `- _ ! ? .`【文档】 | 中文、逗号、撇号都会被 API 拒绝。中文应用也必须用英文命名主题 |
| definition | Classic tier 最多 200 字符，Standard tier 最多 1,000 字符【文档】 | 常见合适长度 100 到 300 字符【博客】 |
| examples | 最多 5 条，每条最多 100 字符【文档】 | 字段可选，但建议用满 5 条【博客】 |
| 主题数量 | 每个 guardrail 最多 30 个【文档】 | 主题越多越容易互相干扰【博客】 |
| type | 固定为 `DENY`【文档】 | |
| guardrail name | 最多 50 字符，只允许字母、数字、`-`、`_`【文档】 | |

## 评估方式

- **三部分一起评估。** guardrail 把 name、definition、examples 当作一个整体，而不是只看定义。名称的影响常常比预期大：领域类名称会让 guardrail 联想到整个领域的词汇，即使定义已经收窄【博客】。
- **按语义匹配。** 不是关键词匹配。语义接近主题的输入也可能被拦，所以领域式的主题会误拦同领域的正常请求【博客】。
- **所有主题一起评估。** 每次请求都把全部主题和输入一起评估。两个主题语义相近时会互相干扰：命中归到哪个主题难以预测，组合起来还可能拦下单个主题都不会拦的输入【博客】。
- **顺序有影响。** 同一个输入在一种主题顺序下被拦、换一种顺序后放行，是可能的【文档】【博客】。
- **一次只报一个主题。** 本 skill 的测试中，每条被拦的输入都只有一个主题标记为 `detected: true`，即使它同时符合两个主题【实测】。所以按主题统计的命中率依赖归因，而归因会受重叠和顺序影响。

## 结果不完全确定

Denied Topics 用语言模型理解语义，存在两种波动【博客】：

1. **重复同一请求。** 输入、配置都不变，连续调用两次，偶尔一次拦、一次放。大多数输入结果稳定，波动集中在语义处于主题边缘的输入。
2. **主题列表变化。** 新增、删除、调整顺序都可能改变之前测过的输入的结果。主题之间语义重叠、定义宽泛时更明显。

定义越精确（动作式、具体对象、清晰锚定、边界示例），落在边缘区的输入越少，两种波动都会减少【博客】。

## 输入、输出和动作

- 每个主题可以分别对输入（`inputEnabled`）和输出（`outputEnabled`）启用【文档】。
- 动作（`inputAction`、`outputAction`）可选 `BLOCK` 或 `NONE`。`NONE` 即 detect 模式：不拦截，只在返回结果里报告检测情况，用于上线前评估【文档】。
- 文档对示例的描述是"a prompt or continuation"。主题检查输出时，可以放一条模型回答式的示例【建议】。

## Safeguard tier

| | Standard | Classic |
|---|---|---|
| 定义长度上限 | 1,000 字符 | 200 字符 |
| Denied Topics 语言 | 支持范围广，简体中文为 "Optimized and supported" | 仅英语、法语、西班牙语 |
| 跨区域推理 | 必须使用 | 不支持 |
| 代码类内容 | 支持检测代码注释、变量名、字符串中的内容 | 不适用 |

【文档】

- Standard tier 必须配置 `crossRegionConfig.guardrailProfileIdentifier`，按源 Region 选择 profile，例如 `us.guardrail.v1:0`、`eu.guardrail.v1:0`、`apac.guardrail.v1:0`。请求可能被路由到同一地理范围内的其他 Region，需要确认是否符合客户的数据驻留要求【文档】。
- Denied Topics 的 tier 在 `topicPolicyConfig.tierConfig` 中设置；Content filter 和 Prompt attack 的 tier 在 `contentPolicyConfig.tierConfig` 中单独设置【文档】。
- 支持 tier 的 Region 列表见 [Safeguard tiers](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html)。

## 语言支持

- Denied Topics：Classic 只支持英语、法语、西班牙语；Standard 支持范围广，包括简体中文和繁体中文【文档】。
- Word filter：文档只列出英语、法语、西班牙语【文档】。
- Sensitive information filter：支持中文【文档】。
- 文档强调必须用目标语言测试，不支持的语言上 guardrail 无效【文档】。
- 中文应用：主题名必须用英文。定义和示例用英文还是中文，博客和文档都没有说明，两种都要测【建议】。本 skill 的测试中，英文定义的主题拦住了一条中文投资建议请求，两条中文正常问题也放行了，但样本只有 3 条【实测】。

## 成本

每个启用的策略都会对每次请求进行评估并计费。同时启用 Denied Topics、Content filter、Prompt attack 时，按组合成本规划【博客】。价格见 [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/)。

## API 格式速查

CreateGuardrail 中的 Denied Topics 部分【文档】：

```json
{
  "name": "my-app-guardrail",
  "blockedInputMessaging": "Sorry, I can't help with that here.",
  "blockedOutputsMessaging": "Sorry, I can't help with that here.",
  "crossRegionConfig": { "guardrailProfileIdentifier": "us.guardrail.v1:0" },
  "topicPolicyConfig": {
    "tierConfig": { "tierName": "STANDARD" },
    "topicsConfig": [
      {
        "name": "Investment Recommendations",
        "definition": "Asking for a personalized recommendation to buy, sell, or hold ...",
        "examples": ["...", "...", "...", "...", "..."],
        "type": "DENY",
        "inputEnabled": true,
        "inputAction": "BLOCK",
        "outputEnabled": true,
        "outputAction": "BLOCK"
      }
    ]
  }
}
```

用 AWS CLI 单独测试一条输入。`--output-scope FULL` 会列出所有主题及其 `detected` 状态，detect 模式下也能看到检测结果：

```bash
aws bedrock-runtime apply-guardrail \
  --guardrail-identifier <guardrail-id> --guardrail-version DRAFT \
  --source INPUT \
  --content '[{"text":{"text":"Explain what an ETF is."}}]' \
  --output-scope FULL \
  --query '{action:action,topics:assessments[].topicPolicy.topics[]}' \
  --output json
```

返回中 `action` 为 `GUARDRAIL_INTERVENED` 表示被拦，`NONE` 表示放行；`topics[].detected` 表示哪个主题命中。测试模型回答时把 `--source` 改为 `OUTPUT`。
