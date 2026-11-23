---
name: skills-for-denied-topics-design-with-amazon-bedrock-guardrail
description: Design, review, test and troubleshoot Denied Topics in Amazon Bedrock Guardrails. Covers requirements intake (must-block and must-pass lists), choosing Denied Topics versus content filters, prompt attack, word filters and sensitive information filters, writing intent-based topic names, action-scoped definitions and boundary examples, context anchors and exclusions, managing overlap, count and order across topics, Standard versus Classic tier limits and language support, building an evaluation set, and rollout. Use when a user wants to block a business-specific subject in a Bedrock application, asks to design or review denied topics or a guardrail configuration, or reports that a guardrail blocks legitimate requests or misses requests it should block. 适用于 Bedrock Guardrails 拒绝主题（Denied Topics）的设计、评审、测试，以及误拦、漏拦排查。
version: 1.0.0
tags:
- bedrock
- guardrails
- denied-topics
- generative-ai-safety
---

# Amazon Bedrock Guardrails Denied Topics 设计

帮助客户设计 Denied Topics：拦住业务要求拦的请求，放过应用本该处理的请求，并用测试证明这两点。

## 适用场景

- 从零为一个 Bedrock 应用设计 denied topics
- 评审客户已有的 guardrail 配置
- 排查误拦（正常请求被拦）和漏拦（该拦的没拦）
- 判断某个需求该用 Denied Topics 还是其他 Guardrails 策略

## 先理解这几点

1. 每个主题的 name、definition、examples 作为一个整体评估。名称本身是很强的信号，用领域词命名（Finance、Legal）会把同领域的正常请求也拉进来。
2. 匹配按语义，不按关键词。和主题语义接近的输入也可能被拦。
3. 一个 guardrail 内的所有主题一起评估。增删主题、调整顺序都可能改变已经测过的输入的结果。官方文档也写明了顺序会影响结果。
4. 结果不完全确定。处于边界的输入，同一请求重复调用可能结果不同。
5. 公开的示例设计不能直接照搬。本 skill 按 AWS 博客的最终设计实测，23 条应放行的请求中仍有 4 条被拦；只调换主题顺序，就有 3 条用例的结果改变。详见 [worked-example-anycompany-bank.md](references/worked-example-anycompany-bank.md)。

## 工作流程

按顺序执行。每一步完成后再进入下一步。

### 1. 收集需求

- 你必须先拿到两张清单，再开始写主题：
  - **must-block**：每一条注明来源和理由（法规、合规、法律风险、品牌政策）。
  - **must-pass**：应用的核心用途，加上与 must-block 语义接近的正常请求。没有这张清单就发现不了误拦。
- 你必须确认：流量语言；Region 和数据驻留要求（决定 tier 和跨区域 profile）；检查输入、输出还是两者；被拦后用户看到什么、能否转人工。
- 你应该向客户要一批脱敏后的真实请求，用来构建评测集。
- 用 [templates/topic-design-worksheet.md](templates/topic-design-worksheet.md) 的 A 部分记录。

### 2. 策略分流

读 [references/policy-selection.md](references/policy-selection.md)，把每条 must-block 分到合适的策略。

- 只有业务特有、能用一段话描述的内容类别才用 Denied Topics。
- 你不得用 Denied Topics 处理通用安全类别、越狱和提示注入、固定词表、个人信息识别，因为原生策略专门为这些信号设计，效果更好，多出来的主题还会干扰其他主题。
- 你不得用 Denied Topics 把应用"限制在业务范围内"（例如"除银行业务外都拦"），因为这是否定式定义，官方文档明确不建议。范围控制交给系统提示和应用路由。

### 3. 逐个设计主题

读 [references/writing-topics.md](references/writing-topics.md)。每个主题按以下顺序完成，记录在 worksheet 的 B 部分：

1. 用一句话写出要拒绝的用户意图（只用于诊断，不写进主题）。
2. 用表达该意图的具体行为命名，2 到 5 个英文词。
3. 按模板写定义：动作开头 + 具体对象 + 限定条件 + 一两个子情形。
4. 写满 5 条边界示例，覆盖不同子情形和句式，不含具体国家、产品、公司名。
5. 概念本身有歧义时，加情境或目的锚定；必要时用双条件。

### 4. 审查整组主题

读 [references/topic-set-and-testing.md](references/topic-set-and-testing.md) 的"主题集管理"部分，并逐项过 [references/review-and-troubleshooting.md](references/review-and-troubleshooting.md) 的检查清单。

- 语义重叠的主题优先合并。
- 相似主题在列表中不要相邻。
- 核对硬限制：名称字符集和长度、定义长度（取决于 tier）、示例数量和长度、主题数量。

### 5. 构建评测集并测试

读 [references/topic-set-and-testing.md](references/topic-set-and-testing.md) 的"评测"部分，评测集格式参考 [templates/eval-set-template.csv](templates/eval-set-template.csv)。

- 评测集必须包含三类：明确应拦、明确应放行、靠近边界。
- 你必须对每条用例至少重复调用 3 次，不得凭单次结果判断主题有效，因为结果有不确定性。
- 在客户账号中创建或修改 guardrail 之前，你必须说明账号、Region、guardrail 名称，并得到确认。
- 评测集使用真实流量时，你必须先脱敏，因为请求中可能含有个人信息。

### 6. 迭代

按 [references/review-and-troubleshooting.md](references/review-and-troubleshooting.md) 的症状表定位原因。

- 每次只改一处（一个名称、一个定义或一组示例），改完重跑全部评测集，对比前后差异。
- 新需求优先扩充已有主题的示例，而不是新增一个相近的主题。

### 7. 上线

- 先用 detect 模式（action 设为 NONE）观察真实流量，确认后再切换为 BLOCK。
- 发布带版本号的 guardrail 版本，应用固定引用该版本。
- 被拦时给用户可恢复的路径（换个问法、转人工），并记录拦截原因用于审计和改进。

## 交付给客户的内容

- 策略分流表：每条 must-block 由哪个策略处理，为什么。
- 每个主题的 name、definition、examples，以及设计理由。
- 主题顺序和理由。
- 评测集和结果，单独列出结果不稳定的用例。
- 残留风险，以及需要业务方决定的边界用例。

## 参考文件

| 文件 | 何时读 |
|---|---|
| [how-denied-topics-work.md](references/how-denied-topics-work.md) | 需要了解机制、字段限制、tier、语言支持、API 格式时 |
| [policy-selection.md](references/policy-selection.md) | 第 2 步，判断用哪个策略 |
| [writing-topics.md](references/writing-topics.md) | 第 3 步，写名称、定义、示例、锚定、排除 |
| [topic-set-and-testing.md](references/topic-set-and-testing.md) | 第 4 到 7 步，管理主题集、评测、上线 |
| [review-and-troubleshooting.md](references/review-and-troubleshooting.md) | 评审已有配置、排查误拦和漏拦 |
| [worked-example-anycompany-bank.md](references/worked-example-anycompany-bank.md) | 需要完整示例，或向客户说明"为什么必须自己测" |
| [templates/topic-design-worksheet.md](templates/topic-design-worksheet.md) | 记录需求、主题设计、审查和测试 |
| [templates/eval-set-template.csv](templates/eval-set-template.csv) | 评测集格式和示例用例 |

## 来源和标注

- AWS Machine Learning Blog: [Best practices for writing Denied Topics in Amazon Bedrock Guardrails](https://aws.amazon.com/blogs/machine-learning/best-practices-for-writing-denied-topics-in-amazon-bedrock-guardrails/)
- Amazon Bedrock User Guide：[Denied topics](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-denied-topics.html)、[Safeguard tiers](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tiers.html)、[Supported languages](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-supported-languages.html)、[Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-cross-region-support.html)、[Word filters](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-word-filters.html)、[Prompt attacks](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-prompt-attack.html)、[Detect mode](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-harmful-content-handling-options.html)

参考文件中的标注含义：【博客】来自上述博客；【文档】来自官方文档或 API 定义；【实测】来自本 skill 编写时的一次测试；【建议】是本 skill 的经验建议，未经官方验证，需要在客户场景中测试。字段限制、语言和 Region 支持会随服务更新变化，交付前以最新官方文档为准。
