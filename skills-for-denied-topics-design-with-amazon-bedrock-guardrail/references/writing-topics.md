# 写好一个主题：名称、定义、示例、锚定、排除

## 1. 总原则：按用户意图设计，而不是按领域

主题误拦正常请求，常见原因是主题描述的是一个领域，而不是用户想从对话中得到什么。同一领域的两个问题可能意图完全不同："我的工资什么时候到账？"和"L5 工程师年薪多少？"都和薪酬有关，只有后者是薪资比较。领域式主题抹掉了这种区别【博客】。

做法【博客】：

1. **识别意图。** 用自己的话写下用户想得到、而你要拒绝的东西，例如个性化推荐、法律意见、他人联系方式、党派立场背书。这一步只用于诊断，不写进主题。
2. **用表达这个意图的动作命名。** 意图告诉你要拦什么，动作给 guardrail 一个可以判断的具体形状。
3. **定义以动作开头。** 名称和定义只要有一个是领域式的，主题就会匹配过宽；两者都描述具体动作，主题才精确。

## 2. 名称

名称对匹配范围的影响往往比预期大。宽泛的类别名（Finance、Security、Legal）会让 guardrail 联想到整个类别的词汇，即使定义已经收窄；具体的动作名（Investment Recommendations、Hacking Instructions、Suing for Damages）在读定义之前就先缩小了范围【博客】。

规则【博客】：

- 用 2 到 5 个词的具体行为短语，不用单个类别词。
- 可用的形式：
  - 动作名词：Investment Recommendations、Hacking Instructions、Election Fraud Claims
  - 具体行为名词短语：Colleague Arguments、Passport and Visa Renewal
  - 动名词加宾语：Suing for Damages、Requesting Contact Details for Named Individuals
- 自检：读出名称，问"这描述的是用户在做的事，还是用户的问题靠近的一个领域？"如果只是领域，改名。
- 避免机构类标签（Government、Company、Insurance），除非动作本身就针对该机构。例如 Government Documents 可能误命中"国家名 + 文件"的内部政策问题。
- 名称里不要重复一个有歧义的词，如果正常的教育类问题也常用这个词。见第 5 节的内幕交易案例。

官方文档的补充【文档】：

- 名称应是名词或短语，不要在名称里描述主题。
- 只允许英文字母、数字、空格和 `- _ ! ? .`，最多 100 字符。

| 改前 | 改后 | 原因 |
|---|---|---|
| Finance | Investment Recommendations | 只拦"要投资建议"这个动作 |
| Legal | Litigation and Non-Servicing Legal Advice | "non-servicing"通过对比把贷款服务类问题排除在外 |
| Fraud | Fraud Investigation Requests | 客户报告欺诈要放行（转人工），只拦"让助手调查欺诈" |
| Insider Trading Advice | Trading on Non-Public Information | 名称不再重复教育类问题会用的词 |

【博客】

## 3. 定义

定义描述用户对某个具体对象的动作，而不是这个动作所属的主题领域【博客】。

### 结构模板

```
[动作或内容类型开头] for [具体对象] by [限定条件或目的]. Includes [一两个子情形].
```

【博客】

### 规则

- **以动作或内容类型开头**【博客】：
  - 用户请求某个动作："Asking for…"、"Requesting…"、"Requests for…"、"Requests to…"、"Seeking…"、"Recommending…"
  - 用户产出的内容、披露或生成内容："Statements that…"、"Providing…"、"Generating…"、"Content that includes…"
  - 特定技术或方法："Using X to Y"
  - 不要用领域式开头："Related to X"、"Content about Y"、"Anything mentioning Z"
- **写出具体对象。** 用具体条目而不是大类，例如写"individual stocks, bonds, ETFs, or cryptocurrencies"，而不是"financial products"【博客】。
- **加区分细节。** 两个主题共用词汇时（例如都涉及"customer"），给每个定义一个独有特征，否则两者会互相干扰【博客】。
- **正面描述。** 描述主题包含什么。可以在末尾加一句简短排除，见第 6 节【博客】。
- **长度不是目标。** 定义越长不一定越好，长定义对相邻主题的措辞也更敏感。常见合适长度 100 到 300 字符；概念有歧义需要锚定时，写长是合理的【博客】。
- **不要写成指令。** 例如"Block all content associated to cryptocurrency"是指令，不是定义【文档】。
- **不要在定义里放示例句。** 示例放到 examples 字段【文档】。
- **不要用否定式定义。** 例如"All contents except medical information"【文档】。
- **不要捕获实体或词语。** 例如"Statements with a competitor name Y"。主题描述的是一个主题，guardrail 按上下文评估；词语和实体交给 Word filter 或 Sensitive information filter【文档】。

### 示例：逐步改写

```
Name: Finance
Definition: Anything related to money, banking, or financial products.
```
误拦"Where can I find my payoff quote?"：payoff 是金融词，quote 和钱有关，整个输入"related to"金融领域【博客】。

```
Name: Investment Recommendations
Definition: Anything related to investing, personal finance decisions, or securities trading.
```
名称改好了，定义仍是领域式的。"Explain what a mutual fund is"、"Why did rates rise last quarter?"、"Show me my holdings"都会被拦【博客】。

```
Name: Investment Recommendations
Definition: Asking for a personalized recommendation to buy, sell, or hold specific stocks,
bonds, mutual funds, ETFs, or cryptocurrencies for the user's own investment returns.
Includes asking whether a specific security is a good investment, requesting portfolio
allocation percentages, or asking when to enter or exit a specific position.
```
- "Asking for a personalized recommendation to…"是动作开头。
- "stocks, bonds, mutual funds, ETFs, or cryptocurrencies"是具体对象。
- "for the user's own investment returns"是限定条件，排除教育类和第三方讨论。
- "Includes…"列出两个子情形。

【博客】

注意：本 skill 的测试中，这个最终定义仍然拦下了"Explain what an ETF is."和"Show me my portfolio report."【实测】。模板是起点，不是保证，必须用评测集验证。

## 4. 示例：用边界示例，不用改写

示例告诉 guardrail 主题的边界在哪里。同一个意思的 5 种改写什么都没教；精心挑选的 5 条示例可以用很少的定义文字勾勒出主题轮廓。最有用的是靠近主题边缘的示例：足够接近一个可能的非匹配，从而告诉 guardrail 线画在哪里。这类示例称为**边界示例**【博客】。

好的边界示例同时满足三点【博客】：

1. 在主题范围内。
2. 靠近一个可能的非匹配，使用和不该匹配的输入相似的词汇或结构。
3. 和其他示例在子情形、句式、词汇上都不同。

规则【博客】：

- 用满 5 条。5 条多样的示例比 2 条显而易见的示例有效。
- 覆盖不同子情形。主题包括"买"和"卖"，两种都要有。
- 覆盖不同句式：直接提问（What should I buy?）、祈使句（Recommend a stock）、陈述句（I'm considering putting money into…）。
- 至少一条靠近边界但仍在范围内。
- 示例中不要出现国家名、产品名、竞品名、公司名，除非主题本身就针对这个实体。具名实体会让主题误命中提到同一实体的无关请求。

补充：

- 示例字段只能放应拦的内容，没有放"应放行示例"的地方【文档】。不要把一条你希望放行的边界请求放进 examples【建议】。
- 主题同时检查输出时，可以放一条写成模型回答形式的示例，例如"You should move 60% of your savings into growth stocks."【建议】
- 中文流量：可以混入中文示例，并和纯英文示例对比测试【建议】。

### 示例：改写式与边界式

改写式（差）：
```
Should I buy stocks?
What stock should I buy?
Is this a good stock to purchase?
Which stocks are good buys?
Recommend some stocks
```
测试中可靠地拦住了"Should I buy stocks?"，却放过了"Is it a good time to rotate my portfolio out of equities into bonds?"，而后者正是主题要拦的。改写全部挤在主题里的一个点上，其他地方没人守【博客】。

边界式（好）：
```
Should I buy 100 shares of a specific stock before earnings?
Is it a good time to move my portfolio into bonds?
What percentage should I allocate to cryptocurrency?
Should I sell my index fund now or hold?
Recommend three growth stocks for a 5-year horizon
```
5 个不同子情形（具体股票、资产类别转换、配置比例、卖出时机、成长股筛选），5 种不同句式。"Explain what an ETF is"（教育）和"Show me my portfolio report"（数据查询）与这些示例差别足够大，博客认为不应匹配【博客】。本 skill 的测试中这两条仍被拦，说明博客的判断也要在自己的环境中验证【实测】。

## 5. 锚定：给有歧义的概念加上下文

有些概念本身有歧义：同样的词既可能描述应拦的动作，也可能描述正常请求，取决于上下文。这时加一个明确的锚点固定上下文【博客】。

三种锚定方式【博客】：

| 方式 | 示例 |
|---|---|
| 物理或机构情境 | 护照主题锚定到"at an embassy or consulate"或"through an official government application process" |
| 目的限定 | 投资建议主题锚定到"for the user's own investment returns" |
| 双条件 | "Only applies when the request (1) explicitly mentions creating an advertising campaign AND (2) names personal identification data as the targeting criterion." |

规则【博客】：

- 锚定用正面表述（"at X"、"for Y"），不用"except…"、"but not…"。
- 主题本身已经很窄时不要加锚定。例如"insider trading"这个短语本身就无歧义，再加锚定只会引入噪声。
- 不要锚定到在大量无关请求中出现的专有名词（"in India"、"at Amazon"），锚定到类别（embassies、consulates）。
- 检查名称是否重复了有歧义的词。教育类的正常问题常常用同一个词，而名称是独立于定义的信号。双条件锚定写对了，仍可能输给一个用歧义词构成的名称。

### 案例：内幕交易

初稿：
```
Name: Insider Trading Advice
Definition: Asking whether to buy or sell a specific security based on inside information.
```
拦住了预期的请求，也拦住了"How does the SEC define insider trading?"，因为它"在问 insider trading"【博客】。

加双条件锚定：
```
Definition: Asking whether to buy or sell a specific security based on non-public material
information about the issuing company, or seeking a recommendation to trade ahead of a known
upcoming public announcement. Applies only when the request (1) names a specific security AND
(2) describes a specific piece of non-public information the user claims to possess.
```
教育类问题不满足条件 2，但测试中仍有时被拦。问题出在名称上：它重复了查询用的词【博客】。

保留锚定，改名：
```
Name: Trading on Non-Public Information
```
改名后教育类问题稳定放行，真正的内幕交易请求仍被拦。这个定义超过 300 字符是有意的：概念本身微妙，定义要承担更多工作【博客】。

注意：本 skill 的测试中，只把主题顺序反转，"How does the SEC define insider trading?"又被这个主题拦了【实测】。见 [topic-set-and-testing.md](topic-set-and-testing.md)。

### 文档与博客的说法差异

官方文档建议不要在定义中写否定或例外【文档】。博客推荐的双条件锚定（"Applies only when…"）和简短尾部排除，从字面看与文档有出入【博客】。调和方式【建议】：

- 定义的主体始终是正面描述，这一点两者一致。
- 双条件是正面地收窄范围，不是用"除了"来定义主题。
- 尾部排除只作为辅助，并按第 6 节判断是否适用。
- 两种写法都用评测集验证，不要凭文字判断。

## 6. 尾部排除：什么时候有效

一句简短的尾部排除（例如"Fraud reports for routing to a human agent are not in scope."）不是解决所有误拦的通用办法【博客】。

- **有效的情况：** 被排除的请求和主题在类别上不同：意图不同、提问者不同、内容类型不同。这时尾部排除是正确的第一选择【博客】。
- **不可靠的情况：** 正常请求的措辞和定义的正面描述几乎一样，只是对象不同。排除句要和一段跟它很像的措辞竞争，往往输掉【博客】。

措辞几乎一样时，改用以下三种之一。只有测试表明尾部排除输给了近似匹配时，才换用这些方法【博客】：

1. 双条件锚定，在定义中直接点名容易混淆的情况（第 5 节）。
2. 改名，把区分点放进名称，让 guardrail 读定义之前就看到（第 2 节）。
3. 上游意图路由：在 Guardrails 之前把有歧义的请求交给专门的流程。

本 skill 的测试中，Fraud Investigation Requests 的尾部排除没有挡住"I want to report a fraudulent charge on my account."和"My card was skimmed at a gas station, what do I do?"【实测】。报告欺诈和要求调查欺诈用词高度重合，属于不可靠的情况。

## 7. 一个主题的完成标准

- [ ] 能用一句话说出要拒绝的意图
- [ ] 名称是 2 到 5 个英文词的具体行为短语，不是领域词
- [ ] 定义以动作或内容类型开头，有具体对象和限定条件
- [ ] 定义没有指令、否定式主体、关键词清单、实体捕获、示例句
- [ ] 5 条示例覆盖不同子情形和句式，至少一条靠近边界，没有具名实体
- [ ] 有歧义的概念加了正面锚定，且名称没有重复歧义词
- [ ] 符合当前 tier 的长度限制
