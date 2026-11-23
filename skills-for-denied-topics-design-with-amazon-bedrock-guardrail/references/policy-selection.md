# 策略选型：哪些需求该用 Denied Topics

Guardrails 有多个策略，目标都是拦截特定内容，但各自针对不同的信号。用 Denied Topics 去做另一个策略原生支持的类别，通常拦多或拦少，配置更复杂，效果更差。为 Content filter 已覆盖的类别精心写一个主题，通常不如直接用 Content filter【博客】。

## 选型表

| 要拦的内容 | 用哪个策略 | 常见错误 |
|---|---|---|
| 通用安全类别：仇恨、侮辱、色情、暴力、不当行为 | Content filter | 建一个名为 Hate Speech 或 Violence 的主题 |
| 越狱、提示注入 | Prompt attack | 建一个名为 Jailbreak 或 Instruction Override 的主题 |
| 需要一律拦截的固定词语、短语、产品名 | Word filter | 建一个名为 Blocked Terms 或 Competitor Names 的主题 |
| 个人标识：姓名、电话、身份证号、卡号 | Sensitive information filter | 建一个名为 PII Detection 的主题 |
| 业务特有的内容类别 | Denied Topics | 这是正确用法 |
| 业务特有，同时需要通用安全覆盖 | Denied Topics 和 Content filter 组合 | 只用 Denied Topics 复刻 Content filter |

【博客】

Denied Topics 适合的需求：业务特有，并能用一段简短描述表达。例如一条法规红线、一项政策承诺、一个法律风险、一条范围边界【博客】。

## 判断顺序

对每条 must-block 依次问：

1. 是通用安全类别吗？是，用 Content filter。
2. 用户是在试图改写助手的指令、或套取系统提示吗？是，用 Prompt attack。
3. 能用一份固定字符串清单完整描述吗？是，用 Word filter。
4. 是识别某类个人信息吗？是，用 Sensitive information filter。
5. 需求其实是"只允许业务范围内的话题"吗？是，交给系统提示和应用路由，不要写成"除 X 以外都拦"的主题。官方文档明确不建议否定式定义【文档】【建议】。
6. 剩下的业务特有类别，用 Denied Topics。

## 各策略要点

**Content filter**【文档】
- 类别：HATE、INSULTS、SEXUAL、VIOLENCE、MISCONDUCT，输入和输出分别设置强度。
- 强度为 HIGH 时，即使模型给出的置信度为 LOW 也会拦。不确定时先用 detect 模式观察，再调强度。

**Prompt attack**【文档】
- 只评估输入，输出强度设为 NONE。
- 覆盖越狱和提示注入；Standard tier 还能检测套取系统提示（prompt leakage）。
- 使用 InvokeModel 或 InvokeModelWithResponseStream 时，必须用输入标签包住用户输入，否则不会检测提示攻击。标签也能避免开发者写的系统提示被误判为攻击。
- 不评估工具调用结果（toolResult）和工具定义。

**Word filter**【文档】
- 自定义词条每条最多 3 个词，最多 10,000 条；另有托管的脏话列表。
- 整词、整短语匹配，不区分大小写。`competitor` 不会匹配 `competitors`，需要把变体都列进去。
- 文档只列出英语、法语、西班牙语支持，中文词条要实测。
- 可以只对输出启用。例如需求是"助手的回答里不出现竞品名"，就只检查输出，用户提问时提到竞品不受影响【建议】。

**Sensitive information filter**【文档】
- 内置多种 PII 类型，也支持自定义正则。
- 动作可选 BLOCK（拦截）或 ANONYMIZE（打码替换）。
- 支持中文。

## 组合使用

- 每个启用的策略都会对每次请求评估并计费，组合使用时按总成本规划【博客】。
- 关键决策不要只靠 Denied Topics 一个策略。和 Content filter、Prompt attack、Sensitive information filter 组合，避免任何单一策略成为唯一防线【博客】。

## 示例：银行助手的分流结果

| must-block | 策略 | 原因 |
|---|---|---|
| 个性化投资建议 | Denied Topics | 业务特有的法规红线 |
| 针对银行的诉讼和法律意见 | Denied Topics | 业务特有的法律风险 |
| 内幕交易建议 | Denied Topics | 业务特有、监管要求高 |
| 制造武器的方法 | Content filter，VIOLENCE 设为 HIGH | 通用安全类别 |
| 提示注入和越狱 | Prompt attack | 信号是"改写助手指令"，与话题无关 |
| 回答中出现竞品名 | Word filter | 精确字符串，不是语义 |

六条里只有前三条适合 Denied Topics，其余三条交给原生策略【博客】。
