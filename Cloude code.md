#cloude
# 配置cloude code
## 1.安装Claude code
### 1.1 使用windows自带工具irm安装
```
irm https://claude.ai/install.ps1 | iex
```
## 1.2 也可以使用npm安装(需要电脑安装nodejs,下载路径：`https://nodejs.org/en/download`)
```
npm install -g @anthropic-ai/claude-code
```
## 2.配置环境变量(以Deep Seek为例)
```
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_AUTH_TOKEN=${DEEPSEEK_API_KEY}
API_TIMEOUT_MS=600000
ANTHROPIC_MODEL=deepseek-chat
ANTHROPIC_SMALL_FAST_MODEL=deepseek-chat
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```
#### 模型配置说明
env 对象 (环境变量配置)
这个部分定义了工具运行时所需的环境变量，主要用于配置与 AI 服务的连接和模型选择。
`ANTHROPIC_AUTH_TOKEN`
	认证令牌。这是调用 API 所必需的密钥。值 sk-58* 是一个被掩码处理的 API Key（通常以 sk- 开头）。******************* 表示这里隐藏了真实密钥的后半部分，出于安全考虑不应明文暴露。
`ANTHROPIC_BASE_URL`
	API 基础 URL。这指定了 AI 服务的具体接入点。这里设置为 https://api.deepseek.com/anthropic，意味着这个工具实际上是通过 DeepSeek 的 API 来使用兼容 Anthropic 格式的模型（如 deepseek-chat 和 deepseek-reasoner）。这是一种常见的做法，通过一个统一的接口来访问不同的大模型提供商。
`API_TIMEOUT_MS`
	API 请求超时时间。单位是毫秒（ms）。这里设置为 600000 ms，即 10 分钟。这是一个非常长的超时时间，表明该工具可能用于处理需要长时间运行的、复杂的代码生成或分析任务，防止在模型“思考”或生成长内容时被过早断开连接。
`ANTHROPIC_MODEL`
	默认主模型。当工具需要一个通用的、平衡的模型时，会使用这里指定的模型。这里设置为 deepseek-chat，即 DeepSeek 的聊天模型。
`ANTHROPIC_SMALL_FAST_MODEL`
	小型快速模型。用于执行简单、快速的任务（如单行代码补全、简短回答），以节省成本和加快响应速度。这里同样指向了 deepseek-chat，可能意味着在当前配置下，没有区分更小或更快的模型，或者 deepseek-chat 本身就被用作默认的快速模型。
`ANTHROPIC_DEFAULT_OPUS_MODEL`
	Opus 级别默认模型。“Opus”是 Anthropic 对其最强大模型的内部代号（如 Claude 3 Opus）。这里映射到 deepseek-reasoner，表明 deepseek-reasoner 被当作是 DeepSeek 系列中最强大、最适合复杂推理任务的模型。当工具需要最强能力时，会调用此模型。
`ANTHROPIC_DEFAULT_SONNET_MODEL`
	Sonnet 级别默认模型。“Sonnet”是 Anthropic 对其平衡性能和成本模型的代号（如 Claude 3 Sonnet）。这里映射到 deepseek-chat，意味着 deepseek-chat 被视为在能力和速度/成本之间的一个良好平衡点。
`ANTHROPIC_DEFAULT_HAIKU_MODEL`
	Haiku 级别默认模型。“Haiku”是 Anthropic 对其最快、最轻量级模型的代号（如 Claude 3 Haiku）。这里同样映射到 deepseek-chat，和 SMALL_FAST_MODEL 情况类似，可能表示没有更小的模型可用，或者统一使用 chat 模型。
`CLAUDE_CODE_SUBAGENT_MODEL`
	Claude 代码子代理模型。当主工具需要委派一个专门的“子任务”（例如，在代码库中查找文件、执行特定命令）给另一个AI代理时，会使用这个模型。这里设置为 deepseek-chat。
`CLAUDE_CODE_MAX_OUTPUT_TOKENS`
	Claude 代码最大输出令牌数。“令牌”（Token）是模型处理文本的基本单位，大约相当于单词的一部分。这里设置为 32000，是一个非常高的上限，允许模型生成非常长的代码文件、详细的文档或复杂的分析报告。这直接关联到 API_TIMEOUT_MS 的长超时设置。
## 3.进入项目目录就可以使用Claude了
```
claude
```
## 4.登录
登陆时选择第二个登陆方式在到要收钱的步骤时关闭就可以登陆成功
若是登陆没有成功就需要配置claude code的配置文件，配置文件一般位于`C:\Users\{用户名}\.claude.json`
打开配置文件后添加或者修改（一般之前登录失败后就会留下有设置选项，此时就需要对原有的配置进行修改）
```
{
  "hasCompletedOnboarding": true
}
```

## 5.如果需要使用多种模型
使用工具cc-switch
```
https://github.com/farion1231/cc-switch
```

