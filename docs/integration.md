# 模型接入说明

## 先确定这三个值

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
- **Translation model**：填写接口接受的模型 ID。
- **API Key**：填写翻译服务密钥。
- **Target language**：默认 `en`，可选 `jp / cn / fr`。
- **Reasoning effort**：默认 `low`；接口不支持时选 `default`。

`/config` 也可以修改非敏感选项。语言变化自动重新加载插件，不需要反复手动启动。

## 接口需要支持什么

插件向 **API URL 原样发起 POST**，不会自动补 `/v1` 或其他路径。

```http
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

请求形状：

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

## 环境变量和 .env

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

解压 Release 中的 `prompt-translator-v0.1.0.zip`，把 `prompt-translator` 文件夹放到 `~/.claude/skills/`。下次启动 Claude Code 时自动加载。

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
| HTTP 401 / 403 | Key、接口与账号权限是否匹配 |
| HTTP 404 | 是否填了完整请求路径 |
| HTTP 400 | 模型 ID、`max_tokens` 和 `reasoning_effort` 是否被接口接受；可把推理档位设为 `default` |
| HTTP 429 | 检查服务额度或限流，稍后重试 |
| 译文过长 | 缩短输入；本版本输出预算为 1024 tokens |
| 路径或代码标记未完整保留 | 模型没有完整保留标记；换翻译模型或缩短输入 |
| 没有加载提示框 | 用 `claude plugin list` 检查插件状态；确认当前打开的是会话，不是后台任务列表 |
| 出现 `Your conversation moved to the background` | 这是 Claude Code 的后台会话视图。选中对应会话按 Enter，或按 Esc 返回；普通会话可在 `/config` 关闭 `leftArrowOpensAgents` |

最后一项是 Claude Code 自身的导航行为，插件不会修改这些快捷键设置。

## 已验证的范围

开发基线为 Claude Code 2.1.291。插件的请求、密钥读取、语言路由、代码片段保护、失败拦截和提示框状态有自动检查。GLM 端点已完成真实翻译和 Claude 收到译文的回读验证。

接口形状兼容不等于每个供应商、模型和桌面端都已测试。完整 Windows / Claude Desktop 视觉效果未在本项目中验证。

## 官方参考

- [插件安装与配置](https://code.claude.com/docs/en/plugins/install)
- [userConfig 字段与安全存储](https://code.claude.com/docs/en/plugins-reference#user-configuration)
- [Mods API](https://code.claude.com/docs/en/plugins/mods/api)
- [后台会话导航](https://code.claude.com/docs/en/agent-view#switch-sessions-without-leaving-the-terminal)
