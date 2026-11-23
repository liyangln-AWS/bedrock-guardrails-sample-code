# 主题集管理、评测与上线

## 一、主题集管理

guardrail 把所有主题一起评估，不是逐个评估。两个主题语义相近时会互相干扰：命中归到哪个主题难以预测，组合起来还可能拦下单个主题都不会拦的输入。一个写得很好的主题出错，原因常常不在它本身，而在相邻的主题【博客】。

### 规则

- **上线前审查整组主题的重叠。** 每加一个主题，都要问它的定义是否和已有主题共用词汇或对象【博客】。
- **注意主题数量。** 主题越多，干扰的可能越大，尤其是多个主题共用词汇或覆盖相关领域时。如果定义本身没问题但主题之间互相串扰，减少重叠，或把主题分到多个 guardrail【博客】。
- **一个 guardrail 内保持语义多样。** 如果有几个相关主题（例如三个都和金融相关），考虑它们是否应该放在同一个 guardrail 里【博客】。
- **重叠时优先合并，不要拆分。** 合并两个共用对象和动作的主题，比在它们之间画一条细线更安全【博客】。
- **新需求优先扩充已有主题的示例。** 新增主题会改变 guardrail 对整组主题的理解，扩充示例不会【博客】。
- **相似主题在列表中不要相邻。** 两个措辞相似的主题挨在一起时，guardrail 更容易把输入归到最近看到的那个。在它们之间插入一个无关主题【博客】。
- **顺序会影响结果。** 同一个输入在一种顺序下被拦、另一种顺序下放行是可能的【文档】。确定顺序后固定下来，任何调整都要重跑评测。

### 案例：宽主题加窄主题

合规团队想在 Investment Recommendations 旁边再加一个更宽的 Financial Advice，指望宽主题兜住窄主题漏掉的请求【博客】：

```
Topic A  Name: Financial Advice
         Definition: Requests for advice about money, investing, or personal finance decisions.
Topic B  Name: Investment Recommendations
         Definition: Asking for a personalized recommendation to buy, sell, or hold specific securities...
```

测试"Should I put more into my 401(k)?"时，随 API 请求中的主题顺序不同，命中可能归到 A、B 或两者，某些顺序下被拦【博客】。

处理：删除 Financial Advice，由 Investment Recommendations 承担具体动作。合并后，同一输入无论主题顺序如何都得到相同结果【博客】。

401(k) 这条本身仍未完全解决："Should I put more…"的句式和真正的交易建议相同，所以 Investment Recommendations 仍会匹配。这是合并解决不了的残留边界情况，需要业务方决定是否拦，或者用第 6 节的替代方法处理【博客】。本 skill 的测试中这条稳定被拦【实测】。

### 案例：调整顺序

原顺序【博客】：
1. Investment Recommendations
2. Trading on Non-Public Information
3. Litigation and Non-Servicing Legal Advice
4. Fraud Investigation Requests

1 和 2 都涉及证券，相邻放置会让边界输入的归因不稳定。调整后【博客】：
1. Investment Recommendations
2. Litigation and Non-Servicing Legal Advice
3. Trading on Non-Public Information
4. Fraud Investigation Requests

调整后，"Should I trade my Example Corp shares based on rumor?"更稳定地归到 Trading on Non-Public Information【博客】。

本 skill 的测试中，把调整后的顺序整体反转，有 3 条用例的结果发生变化，另有 3 条被拦用例换了命中主题【实测】。顺序的影响确实存在，而且无法事先推断，只能测。

## 二、评测

### 评测集的组成

| 类别 | 内容 | 作用 |
|---|---|---|
| 明确应拦 | 每个 must-block 都要有，覆盖不同子情形、句式、措辞 | 衡量漏拦 |
| 明确应放行 | 应用的核心用途，取自真实流量 | 衡量误拦 |
| 靠近边界 | 和某个主题共用词汇但应放行的请求，以及刚好在主题内的请求 | 改动后最容易变化，最能暴露问题 |
| 交给其他策略的 | 越狱、暴力、竞品名等 | 确认分流后的策略生效，且不被主题抢走归因 |
| 输出侧 | 模型回答形式的文本 | 主题检查输出时需要 |
| 其他语言 | 每种流量语言都要有 | 验证语言支持 |

【博客】【建议】

- 边界用例最重要。每个主题至少准备 3 到 5 条"像它但应放行"的请求【建议】。
- 业务方还没决定的边界用例（例如 401(k)）单独标记为待定，跑出结果后交给业务方决定，不计入准确率【建议】。
- 使用真实流量前必须脱敏。
- 格式参考 [../templates/eval-set-template.csv](../templates/eval-set-template.csv)，其中是银行案例使用的 42 条用例。各列含义：

| 列 | 含义 |
|---|---|
| id | 用例编号，唯一 |
| source | `INPUT`（用户请求）或 `OUTPUT`（模型回答），对应 ApplyGuardrail 的 `source` 参数 |
| text | 要评估的文本 |
| expected | `BLOCK`（应拦）、`PASS`（应放行）、`REVIEW`（待业务方决定，不计入准确率） |
| expected_policy | 应拦时预期由哪个策略或主题命中，例如 `topic:Investment Recommendations`、`content:VIOLENCE`、`content:PROMPT_ATTACK`、`word:custom` |
| category | 分组，例如 `must-block:investment`、`must-pass:mortgage`、`boundary`、`lang:zh` |
| notes | 备注，例如用例来源 |

### 测试方法

- **用 ApplyGuardrail 测 guardrail 本身。** 不经过模型，只评估给定文本，结果里能看到每个策略、每个主题是否命中。命令见 [how-denied-topics-work.md](how-denied-topics-work.md) 的"API 格式速查"【文档】。
- **每条用例至少重复 3 次。** 一次的结果不能证明正确。每次都拦记为"拦"，每次都放记为"放"，有拦有放记为"不稳定"。不稳定的用例按错误处理，并说明它处于主题边缘【博客】【建议】。
- **记录命中的是哪个主题。** 被拦了但命中的主题不对，说明主题之间有重叠，将来改动时容易出问题【建议】。
- **每次改动后重跑全部评测集。** 改名、改定义、新增主题、调整顺序都可能改变之前通过的用例【博客】。
- **每次只改一处。** 否则无法判断是哪处改动导致了变化【建议】。
- **对比前后结果。** 列出由对变错、由错变对、换了命中主题的用例【建议】。

### 结果怎么看

| 指标 | 含义 |
|---|---|
| 应拦用例的稳定拦截率 | 每次都拦的比例 |
| 应放行用例的稳定放行率 | 每次都放的比例 |
| 不稳定用例数 | 落在主题边缘的输入数 |
| 误拦归因 | 每个主题造成了几条误拦 |
| 归因错误 | 被拦但命中的不是预期主题的用例 |

通过标准由客户的业务方和合规团队确定，在测试前定好【建议】。

## 三、上线

- **先用 detect 模式。** 把主题的动作设为 NONE，guardrail 不拦截，只在结果中报告检测情况。用真实流量观察一段时间，确认误拦和漏拦在可接受范围内，再改为 BLOCK【文档】。
- **发布版本。** 测试用 DRAFT，上线前发布带版本号的版本，应用固定引用该版本。改动先在 DRAFT 上测，通过后再发布新版本【文档】【建议】。
- **让错误拦截可恢复。** 高风险主题被拦时转人工；允许用户换个问法；记录拦截原因，用于审计和改进主题集【博客】。
- **不要只靠 Denied Topics。** 关键决策和 Content filter、Prompt attack、Sensitive information filter 组合使用【博客】。
- **持续评估。** 把线上新出现的误拦、漏拦加入评测集，每次改动后重跑【建议】。
