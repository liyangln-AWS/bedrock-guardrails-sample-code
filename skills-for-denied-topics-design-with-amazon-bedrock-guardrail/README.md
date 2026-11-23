# Amazon Bedrock Guardrails Denied Topics 设计 Skill

这是一个 Agent Skill，帮助 AI 编程助手（如 Kiro）为 Amazon Bedrock Guardrails 设计、评审、测试 Denied Topics（拒绝主题），并排查误拦和漏拦。

内容基于 AWS 博客 [Best practices for writing Denied Topics in Amazon Bedrock Guardrails](https://aws.amazon.com/blogs/machine-learning/best-practices-for-writing-denied-topics-in-amazon-bedrock-guardrails/) 和 Amazon Bedrock 官方文档，并补充了一次实测结果。

## 能做什么

- 根据 must-block（必须拦截）和 must-pass（必须放行）清单，从零设计一组 denied topics
- 判断每条需求该用 Denied Topics，还是 Content filter、Prompt attack、Word filter、Sensitive information filter
- 评审已有的 guardrail 配置，指出命名、定义、示例、主题重叠、顺序等问题
- 构建评测集，用重复测试验证效果
- 定位误拦、漏拦、结果不稳定的原因，并给出修改顺序
- 规划上线：detect 模式观察、版本发布、错误拦截的兜底

## 目录结构

```
skills-for-denied-topics-design-with-amazon-bedrock-guardrail/
├── SKILL.md                                  工作流程（agent 的入口文件）
├── README.md                                 本说明
├── references/
│   ├── how-denied-topics-work.md             机制、字段限制、tier、语言支持、API 格式
│   ├── policy-selection.md                   策略选型：哪些需求不该用 Denied Topics
│   ├── writing-topics.md                     名称、定义、示例、锚定、排除的写法
│   ├── topic-set-and-testing.md              主题集管理、评测、上线
│   ├── review-and-troubleshooting.md         反模式、评审清单、症状排查
│   └── worked-example-anycompany-bank.md     完整案例和实测结果
└── templates/
    ├── topic-design-worksheet.md             设计记录表
    └── eval-set-template.csv                 评测集格式和 42 条示例用例
```

`SKILL.md` 只包含工作流程，具体写法和案例都在 `references/` 中。agent 会按 `SKILL.md` 中的相对链接按需读取这些文件，所以**必须安装整个文件夹**，并保持内部结构不变。

## 安装

### 1. 获取文件

```bash
git clone https://github.com/liyangln-AWS/bedrock-guardrails-sample-code.git
```

### 2. 复制到 skills 目录

**Kiro**，任选其一：

```bash
# 用户级：所有项目可用
mkdir -p ~/.kiro/skills
cp -R bedrock-guardrails-sample-code/skills-for-denied-topics-design-with-amazon-bedrock-guardrail ~/.kiro/skills/

# 项目级：只在当前项目可用
mkdir -p <项目目录>/.kiro/skills
cp -R bedrock-guardrails-sample-code/skills-for-denied-topics-design-with-amazon-bedrock-guardrail <项目目录>/.kiro/skills/
```

**Claude Code**：复制到 `~/.claude/skills/`（用户级）或 `<项目目录>/.claude/skills/`（项目级）。

**其他支持 Agent Skills 格式的工具**：复制到该工具的 skills 目录，路径以其文档为准。

注意：

- 文件夹名必须与 `SKILL.md` 中的 `name` 字段一致，不要改名。
- 不要只复制 `SKILL.md`，否则 references 和 templates 的链接会失效。

### 3. 确认已加载

重启或新开一个会话，问 agent "你有哪些 skills？"，列表中应出现 `skills-for-denied-topics-design-with-amazon-bedrock-guardrail`。

## 使用

直接用自然语言描述需求，agent 会根据 skill 的描述自动加载。示例：

**从零设计**
> 我们是一家银行，要上线客户助手。合规要求不能给个性化投资建议，不能提供针对银行的法律意见。助手要能回答账户、转账、房贷问题。帮我设计 Bedrock Guardrails 的 denied topics。

**评审已有配置**
> 这是我们现在的 guardrail 配置（附 JSON 或 `aws bedrock get-guardrail` 的输出），帮我评审 denied topics 有什么问题。

**排查误拦**
> 我们的 guardrail 把 "什么是 ETF？" 也拦了，主题定义如下……帮我分析原因并修改。

**只问策略选型**
> 拦截竞品名称、越狱攻击、暴力内容，该用 denied topics 吗？

agent 会按以下流程工作，并在每一步向你确认：

1. 收集需求：must-block、must-pass、流量语言、Region、检查输入还是输出
2. 策略分流
3. 逐个设计主题
4. 审查整组主题（重叠、顺序、字段限制）
5. 构建评测集并测试
6. 每次改一处，重跑评测
7. 上线

建议提前准备好 must-block 和 must-pass 两张清单，以及一批脱敏后的真实用户请求。

### 不用 agent 也能用

- 把 `references/` 当作设计指南阅读，从 `writing-topics.md` 开始。
- 用 `templates/topic-design-worksheet.md` 记录需求和设计。
- 参照 `templates/eval-set-template.csv` 的格式整理评测集。

## 在 AWS 账号中测试的前提

设计和评审不需要 AWS 账号。如果要让 agent 创建 guardrail 并测试，需要：

- AWS CLI 或 SDK，并配置好凭证
- 所在 Region 支持 Amazon Bedrock Guardrails
- IAM 权限：`bedrock:CreateGuardrail`、`bedrock:UpdateGuardrail`、`bedrock:GetGuardrail`、`bedrock:ListGuardrails`、`bedrock:CreateGuardrailVersion`、`bedrock:ApplyGuardrail`，删除测试 guardrail 还需要 `bedrock:DeleteGuardrail`。建议只授予测试用的角色，并把 Resource 限定到 guardrail 的 ARN
- 使用 Standard tier 时，Resource 还要包含 guardrail profile，调用时要包含 profile 的所有目标 Region，见 [Permissions for using cross-Region inference with Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrail-profiles-permissions.html)

skill 要求 agent 在创建或修改 guardrail 之前，先说明账号、Region 和 guardrail 名称，并等待你确认。测试会产生 ApplyGuardrail 调用费用，按 [Amazon Bedrock 定价](https://aws.amazon.com/bedrock/pricing/) 计费。测试结束后，记得删除不再需要的测试 guardrail。

## 内容来源与标注

参考文件中每条内容都标注了来源：

| 标注 | 含义 |
|---|---|
| 【博客】 | 来自上述 AWS 博客 |
| 【文档】 | 来自 Amazon Bedrock 官方文档或 API 定义 |
| 【实测】 | 来自编写本 skill 时的一次测试（2026 年 11 月，us-east-1，Standard tier，42 条用例，每条 3 次） |
| 【建议】 | 本 skill 的经验建议，未经官方验证，需要在你的场景中测试 |

## 注意事项

- 本 skill 不是 AWS 官方产品。字段限制、语言支持、Region 支持会随服务更新变化，使用前以 [Amazon Bedrock Guardrails 官方文档](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) 为准。
- 【实测】结果只代表一次测试，用来说明"必须用自己的流量测试"，不代表 Denied Topics 的整体准确率。
- Denied Topics 的结果不完全确定。不要只靠它做关键决策，应与其他 Guardrails 策略组合使用，并让错误拦截可恢复。

## 版本

- 1.0.0：首个版本
