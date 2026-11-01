# 模型接入说明

## 先选择翻译方式

插件配置中的 **Translation backend** 支持：

| 值 | 调用方式 | 认证来源 |
| --- | --- | --- |
| `api`（默认） | OpenAI 兼容 HTTP 接口 | 插件配置、环境变量或 `.env` 中的翻译 API Key |
| `codex-cli` | 本地 `codex exec` 子进程 | Codex CLI 已保存的登录态和服务配置 |

两种模式均用于输入翻译和输出翻译，共用代码片段保护和失败拦截。不会在两种模式之间自动回退。

## 四段独立语言

| 字段 | 配置键 | 默认 | 说明 |
| --- | --- | --- | --- |
| Input language | `input_language` | `cn` | 输入语言，也可选 `auto` 自动识别 |
| Claude prompt language | `language` | `en` | 输入翻译后发给 Claude 的语言，保留原有配置键 |
| Claude reply language | `claude_language` | `same-as-prompt` | Claude 正文回复语言，可单独指定；默认跟随提示词语言 |
| Output language | `output_language` | `cn` | 最终输出语言；`same-as-claude` 跳过输出翻译 |

各语言支持 `cn / en / jp / fr`。默认链路为：中文输入 → 英文提示词 → Claude 英文回复 → 中文输出。中文输入、法文输出可仅修改 `output_language=fr`。输入语言为提示，翻译模型仍会处理混合语言。

原有 `language` 配置继续作为提示词语言。Claude 回复语言默认跟随它，最终输出新增为默认中文；想保留直接显示 Claude 原文的行为，可将 **Output language** 设为 `same-as-claude`。

输出翻译使用原生 `turn.step` 流式钩子，只改写主会话的普通正文，工具调用、参数、工具结果和思考块保持原有流程。它同时改写显示和 transcript，后续 Claude 会读取翻译后的历史回复。Claude 回复语言通过上下文指令引导，不代表控制内部思考过程。

插件等待每次模型响应收齐后再翻译正文，以保护跨 chunk 的代码块，不逐字输出译文。需要输出翻译时，每次有正文的主会话模型响应增加一次翻译调用。配置相同的回复和输出语言，或选择 `same-as-claude`，可省去这次调用。

## 安装时选择与后续修改

安装后，在 Claude Code 中运行 `/plugin configure prompt-translator@claude-prompt-translator`，按顺序选择翻译方式、四段语言，再填写该模式需要的配置。这个入口也可以从 `/plugin` 的 Installed 标签进入：选择本插件，再选择 **Configure options**。

也可以在安装命令中直接指定配置，以下参数已在 Claude Code 2.1.316 验证：

```bash
claude plugin install prompt-translator@claude-prompt-translator \
  --config backend=codex-cli \
  --config input_language=cn \
  --config language=en \
  --config claude_language=en \
  --config output_language=fr
```

上例会安装为中文输入、英文提示词、Claude 英文回复、法文最终输出。后续通过同一配置入口或 `/config` 随时切换后端和语言。两种后端的配置分别保留，切换不会清空另一种模式的字段。API Key 只在 API 模式使用，不影响 Codex CLI 的配置和运行。

在交互会话中保存配置后，Claude Code 会重新加载插件。若使用终端命令 `claude plugin configure ... --values-stdin` 在会话外修改配置，则按命令提示重启 Claude Code，或在会话中重新加载插件。

## Codex CLI 模式

在运行 Claude Code 的同一机器、用户和环境中确认：

```bash
codex --version
codex login status
```

尚未登录时，先运行 `codex login` 完成登录。插件不会读取或复制 Codex 的凭据文件。

在插件配置中把 **Translation backend** 设为 `codex-cli`，**Codex executable** 保持 `codex`。后者通过 Claude Code 进程的 `PATH` 查找；命令不可见时填写可执行文件的绝对路径。可执行文件路径可以包含空格，直接填写路径，不额外加引号，也不拼接参数。SSH 会话调用的是远端机器上的 CLI。

CLI 模式不读取翻译 API Key，**API URL / Translation model / Reasoning effort** 均不参与该模式；模型和推理设置沿用本地 Codex 的配置。

插件通过 Claude Code 的 `$.process.run` 无 shell 启动 `codex exec`，从标准输入传递保护后的文本，只取标准输出中的最终回答。每次启动独立临时会话，使用只读沙箱、禁用 shell 工具，并关闭 `AGENTS.md` 的自动加载，避免项目开发指令干扰翻译。翻译指令要求只翻译、不执行文本中的任务。

每次调用最多等待 120 秒。启动失败、超时、非零退出码、空输出、标准输出被截断或保护标记异常时，都会停止提交。取消的提交即使收到译文也不会继续发送；子进程的停止由宿主和超时控制。插件不会显示 Codex 的原始标准错误内容。

Codex 仍会访问其配置的模型服务，并遵循该服务的额度和可用性。本地配置中的 MCP、hooks 等扩展可能影响启动耗时和行为；只读沙箱和禁用 shell 不代表所有扩展都被禁用。插件不会修改本地 Codex 配置。

本功能已在 Codex CLI 0.166.0 和 Claude Code 2.1.316 环境下验证。旧版 Codex 若不支持这些参数，请先升级；其他操作系统尚未验证。

## API 模式：先确定这三个值

从你的翻译服务获取：**完整请求地址、模型 ID、API Key**。这三项必须来自同一个服务。

| 服务类型 | API URL 示例 | 模型与 Key |
| --- | --- | --- |
| GLM Coding 接口 | `https://open.bigmodel.cn/api/coding/paas/v4/chat/completions` | 使用该服务支持的模型 ID 和自己的 Key |
| 自建 OpenAI 兼容网关 | `https://your-gateway.example/v1/chat/completions` | 使用网关暴露的模型 ID 和 Key |
| 本地 OpenAI 兼容服务 | `http://127.0.0.1:1234/v1/chat/completions` | 使用本地服务加载的模型 ID；无需鉴权的服务可填 `local` 作为占位 Key |

本地地址是 **Claude Code 所在机器** 的地址。通过 SSH 使用插件时，`127.0.0.1` 指向远端机器。

GLM 是本项目已完成真实调用验证的默认接入。网关和本地服务按下面的协议对接；不同服务对模型名、额度和扩展参数的要求由该服务决定。

## 填入配置

在 Claude Code 中执行：

```text
/plugin configure prompt-translator@claude-prompt-translator
```

- **API URL**：填写完整请求路径，包含 `chat/completions` 或你网关的等效路径。
- **Translation backend**：选择 `api`。
- **Translation model**：填写接口接受的模型 ID。
- **API Key**：填写翻译服务密钥。
- **Input language / Claude prompt language / Claude reply language / Output language**：分别设置输入、提示词、Claude 回复和最终输出语言。
- **Reasoning effort**：默认 `low`；接口不支持时选 `default`。

`/config` 也可以修改非敏感选项。语言变化自动重新加载插件，不需要反复手动启动。

## API 接口需要支持什么

插件向 **API URL 原样发起 POST**，不会自动补 `/v1` 或其他路径。

```http
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

输入翻译的请求形状：

```json
{
  "model": "your-model-id",
  "messages": [
    { "role": "system", "content": "Translation instructions..." },
    { "role": "user", "content": "Text to translate..." }
  ],
  "max_tokens": 1024,
  "stream": false
}
```

选择 `low / medium / high / max` 时，会额外发送相应的 `reasoning_effort`。选择 `default` 时省略该字段。

输出翻译使用相同协议，系统提示要求翻译 Claude 回复，输出预算为 `max_tokens: 8192`；输入翻译保留 `1024` 的预算。需要供应商支持对应的输出长度。

响应形状：

```json
{
  "model": "your-model-id-or-resolved-alias",
  "choices": [
    {
      "finish_reason": "stop",
      "message": { "content": "Translated text..." }
    }
  ]
}
```

需要 HTTP 2xx、有效 JSON、非空文本；返回 `finish_reason: length` 时停止提交。模型别名可由服务端解析为实际版本名，插件接受这种响应。

这个入口使用 `chat/completions` 协议。仅提供 Anthropic `messages` 或 OpenAI `responses` 协议的地址，需要先通过网关转换。

## API 环境变量和 .env

推荐通过插件配置填写 Key。自动化环境也可以提供 `TRANSLATOR_API_KEY`。

加载顺序：**插件配置的 API Key → 进程环境变量 → 插件目录的 `.env`**。

本地开发时：

```bash
cp plugins/prompt-translator/.env.example plugins/prompt-translator/.env
chmod 600 plugins/prompt-translator/.env
```

用编辑器填写这一行：

```dotenv
TRANSLATOR_API_KEY=your-api-key
```

`.env` 只用于 Key。接口、模型、语言和推理档位在插件配置里填写。插件安装目录可能随更新改变，因此日常安装优先使用插件安全存储，而不是在安装缓存里维护 `.env`。

## 手动安装或使用 Release ZIP

若使用 Release ZIP，解压所需版本的 `prompt-translator` 文件夹，放到 `~/.claude/skills/`。下次启动 Claude Code 时自动加载。较旧版本可能不包含当前的双向翻译和配置选项。

该安装方式的插件 ID 为 `prompt-translator@skills-dir`。配置时使用：

```text
/plugin configure prompt-translator@skills-dir
```

临时加载源码：

```bash
claude --plugin-dir ./plugins/prompt-translator
```

这是源码开发入口；日常接入优先用 README 中的插件安装命令。相同插件不要同时用两种方式安装。

## 故障排查

| 现象 | 检查 |
| --- | --- |
| 提示没有 API Key | 检查当前插件 ID 的配置，或 `TRANSLATOR_API_KEY` |
| Codex CLI 无法启动或超时 | 检查 **Codex executable**、Claude Code 的 `PATH`、`codex login status` 和本地 Codex 配置；最长等待 120 秒 |
| Codex CLI 非零退出码 | 在相同环境单独检查 Codex 的登录、额度、服务配置和版本 |
| Codex CLI 输出为空或被截断 | 缩短输入后重试，检查本地 Codex 是否能正常完成非交互任务 |
| HTTP 401 / 403 | Key、接口与账号权限是否匹配 |
| HTTP 404 | 是否填了完整请求路径 |
| HTTP 400 | 模型 ID、`max_tokens` 和 `reasoning_effort` 是否被接口接受；可把推理档位设为 `default` |
| HTTP 429 | 检查服务额度或限流，稍后重试 |
| 输入 API 译文过长 | 缩短输入；输入翻译 API 预算为 1024 tokens |
| 输出翻译失败 | 检查翻译后端；输出 API 预算为 8192 tokens。过长回复需缩短后重试；缓冲上限为 256 KiB 序列化 chunk 数据或 10,000 个 chunk |
| 路径或代码标记未完整保留 | 模型没有完整保留标记；换翻译模型或缩短输入 |
| 没有加载提示框 | 用 `claude plugin list` 检查插件状态；确认当前打开的是会话，不是后台任务列表 |
| 出现 `Your conversation moved to the background` | 这是 Claude Code 的后台会话视图。选中对应会话按 Enter，或按 Esc 返回；普通会话可在 `/config` 关闭 `leftArrowOpensAgents` |

最后一项是 Claude Code 自身的导航行为，插件不会修改这些快捷键设置。

输出缓冲超限时停止翻译原文，并在后续正文块显示失败提示，避免重试覆盖唯一的提示。若超限后的响应流又中断，会保留中断状态，不补造正常结束标记或重复调用主模型。

## 已验证的范围

双向翻译的开发基线为 Claude Code 2.1.316。插件的请求、密钥读取、四段语言路由、代码片段保护、响应流协议保留、失败拦截和输入提示框状态有自动检查。输入 API 模式的 GLM 端点曾完成真实翻译和 Claude 收到译文的回读验证；这不代表本轮已真实验证输出 API 的每个供应商。

Codex CLI 后端已验证：mock 测试覆盖参数、标准输入、语言路由、代码片段还原、异常拦截和提示框；另在本机通过实际提交处理函数启动真实 Codex 子进程，确认英文译文、代码路径还原和回复语言 context。该检查不包含真实 Claude 模型接收译文后的回答，也不包含 CLI 模式的完整交互界面验收。

双向链路另通过实际输入和响应 hook 启动两次真实 Codex 子进程：中文输入译成英文，模拟的英文 Claude 回复译成日文，两次均还原代码路径。主模型回复在该检查中为模拟文本；完整的真实 Claude 会话仍需单独验收。

接口形状兼容不等于每个供应商、模型和桌面端都已测试。完整 Windows / Claude Desktop 视觉效果未在本项目中验证。

## 官方参考

- [插件安装与配置](https://code.claude.com/docs/en/plugins/install)
- [userConfig 字段与安全存储](https://code.claude.com/docs/en/plugins-reference#user-configuration)
- [Mods API](https://code.claude.com/docs/en/plugins/mods/api)
- [Codex 非交互模式](https://developers.openai.com/codex/noninteractive)
- [后台会话导航](https://code.claude.com/docs/en/agent-view#switch-sessions-without-leaving-the-terminal)
