# MyCode

[English](README.md) | **中文**

JetBrains IDE 里的编程助手。你在 IDE 里发消息，它调用你配置的模型，并用 IDE 里的工具读代码、改文件、跑命令。不需要再装一套外部 CLI。

工具窗口在 IDE 右侧，标题是 **MyCode**。聊天输入框里把引擎选成 **MyCode**。

> 本项目源码不公开。本仓库用于发布版本、说明用法和收集问题反馈。
> **免费使用，但非开源** —— 个人和商业用途都不收费，不可再分发或修改。
> 许可条款见 [LICENSE](LICENSE)，中文参考译本见 [LICENSE.zh_CN](LICENSE.zh_CN)。

## 环境要求

- JetBrains IDE 2023.3（`233`）至 2026.3（`263.*`），IDEA / PyCharm / WebStorm / GoLand 等均可
- Git 插件（JetBrains IDE 自带，无需单独安装）

## 发一条消息

1. 打开右侧 **MyCode**。
2. 确认引擎是 **MyCode**，并选好模型。还没有模型时，先做下面的「配置模型」。
3. 输入问题后发送。
4. 想带上当前代码：在编辑器里选中，按 `Ctrl+Alt+K`（macOS：`Cmd+Alt+K`）。

改文件、跑命令、访问工作区以外的路径时，会弹出权限确认。选允许、拒绝，或记住这次 / 本会话 / 本工作区 / 所有工作区。默认模式是询问用户。

## 配置模型

设置 → **模型配置**，或直接编辑：

- macOS / Linux：`~/.mycode/config.json`
- Windows：`%USERPROFILE%\.mycode\config.json`

```json
{
  "providers": {
    "deepseek": {
      "protocol": "openai",
      "baseUrl": "https://api.deepseek.com",
      "apiKey": "sk-...",
      "models": {
        "deepseek-chat": { "context": 128000, "output": 8192 }
      }
    },
    "anthropic": {
      "protocol": "anthropic",
      "baseUrl": "https://api.anthropic.com",
      "apiKey": "sk-ant-...",
      "models": {
        "claude-sonnet-4-5": { "context": 200000, "output": 8192 }
      }
    }
  }
}
```

| 字段 | 含义 |
| --- | --- |
| 供应商 id | `providers` 的键，字母数字开头，可含 `.` `_` `-` |
| `protocol` | `openai` 或 `anthropic`。不写时按 OpenAI 兼容接口处理 |
| `baseUrl` | API 根地址 |
| `apiKey` | 密钥。有的网关用 `authToken` |
| `models` | 模型 id → `context`（上下文 token）和 `output`（单次输出上限） |

从设置页保存时，会按模型 id 重写 `modelMap`（模型 id → 供应商 id），其它顶层字段保留。输入框里的模型列表来自这里。多个供应商里，配置文件中排在最前的那个模型会当作默认模型。

模型名如果带 `1m` 上下文后缀，上下文按 100 万 token 计。

文件不存在时，仍会尝试读取旧位置 `~/.codemoss/config.json` 里的 `mycode` 或 `myagent` 段。新配置写好后以 `~/.mycode/config.json` 为准。

## 它能做什么

一轮对话里，模型可以调用这些工具：

| 工具 | 作用 |
| --- | --- |
| `read` / `grep` / `glob` | 读文件、搜索内容、按文件名查找 |
| `edit` / `write` | 改已有文件、整文件写入 |
| `bash` | 在项目目录执行命令 |
| `search_symbol` / `inspect_symbol` / `find_symbol_usages` | 用 IDE 索引查符号、定义和引用 |
| `rename_symbol` | 按 IDE 重构重命名 |
| `check_errors` | 看当前文件的 IDE 报错 |
| `webfetch` | 抓取网页 |
| `Skill` | 载入一个技能的说明并按它执行 |
| `askuserquestion` | 向你提一个选择题 |

Java 项目里，符号类工具走 IDE 的 PSI。命令和写文件都会先过权限。

## 权限

权限看的是这次调用在做什么，不看工具名字。一次调用可能同时碰到几种能力，全部通过才执行；有一条拒绝就拒绝。

| 能力 | 例子 |
| --- | --- |
| `fs.read` | 读文件、列目录、搜索；`ls` / `cat` / `git status` 这类只读命令 |
| `fs.write` | 改、建、删、移动；重定向写入 |
| `exec` | 会启动进程的命令：`npm`、`python`、`git push`、解析不了的命令 |
| `net` | `webfetch` 以及会出网的命令 |

默认策略：

- 工作区里的读取可以直接做。
- 工作区里的写入、执行命令、出网，先问你。
- `.git`、`.ssh`、`.env`、shell 配置等敏感路径上的写入，一定会问。
- 危险命令（解释器、包管理器、`git push` / `commit` / `reset` 等）一定会问。

弹窗里常见选择：

- **仅此一次**：这次放行，不记住。
- **本次会话**：关掉这次会话就失效。
- **本工作区**：写入 `~/.mycode/permission/workspaces/<id>.json`。
- **所有工作区**：写入 `~/.mycode/permission/settings.json`。

配置合并顺序：内置默认 → 全局 `settings.json` → 当前工作区文件。规则同时存在时，拒绝优先于询问，询问优先于允许。

`permissions.defaultMode` 默认是 `ask`（问你）。日常使用保持 `ask`。

## 技能

技能是一份带说明的 Markdown，模型在需要时用 `Skill` 工具读进来，也可以在输入框用 `/名字` 调用。

目录：

- 所有项目：`~/.mycode/skills/<名字>/SKILL.md`
- 仅当前项目：`{项目根}/.mycode/skills/<名字>/SKILL.md`

同名时，项目技能覆盖用户技能。在设置 → **Skills** 里关掉的技能会挪到旁边的 `skills-disabled/`，代理不会加载。

```markdown
---
name: review
description: 按清单审查当前改动，指出行为和测试缺口。
---

先看 git diff，再按下面的清单给出结论……
```

`name` 用小写字母、数字和单个连字符，最长 64，不能以连字符开头或结尾。不写 `name` 时用目录名。`description` 会进技能列表，写清楚什么时候该用它。

## 数据放哪

```
~/.mycode/
  config.json              # 供应商、模型、压缩、追踪
  models-dev.json          # 模型目录缓存
  permission/
    settings.json          # 所有工作区共用
    workspaces/
      <workspaceId>.json   # 仅当前项目
  skills/                  # 启用的用户技能
  skills-disabled/         # 在设置里关掉的用户技能
  sessions/                # 会话 jsonl
  traces/                  # 打开追踪后的日志

{项目根}/.mycode/
  skills/
  skills-disabled/
```

`workspaceId` 是项目根路径规范化之后的 SHA-256 前 16 个十六进制字符。真实路径写在该文件的 `workspace.root`。换一台机器或换一个目录，id 会变，不会去读其它项目的权限文件。

会话文件在 `~/.mycode/sessions/<项目路径>/` 下，路径里的非字母数字会换成 `-`。例如 `D:\Projects\MyProject` 对应目录名 `D--Projects-MyProject`，里面是 `<sessionId>.jsonl`。

## 上下文变长之后

对话接近模型上下文上限时，会自动做一次摘要，并尽量把最近读过的文件片段补回去。默认开启。要改行为，在 `config.json` 里加：

```json
"compact": {
  "enabled": true,
  "bufferTokens": 8000
}
```

`bufferTokens` 是距上下文顶还剩多少 token 时开始压缩，默认 8000。`enabled` 设为 `false` 则关闭自动压缩。单条工具结果过长时也会在当轮截断，避免一次 `grep` 把窗口撑满。

## 追踪

默认不写追踪。排查某一轮为什么停住时，在 `config.json` 里打开：

```json
"tracing": {
  "enabled": true
}
```

文件写到 `~/.mycode/traces/mycode-<id>.json`。`outputDir` 可改目录。

## 快捷键

| 操作 | Windows / Linux | macOS |
| --- | --- | --- |
| 发送选中代码到输入框 | `Ctrl+Alt+K` | `Cmd+Alt+K` |
| 用 AI 快速修复 | `Ctrl+Shift+Q` | `Cmd+Shift+Q` |

## 问题反馈

到本仓库提 Issue，请附上：

- IDE 名称与版本
- MyCode 版本（**Settings → Plugins** 里可见）
- 复现步骤
- 相关日志（**Help → Show Log in Explorer**）

排查某一轮卡住时，先按上面的「追踪」打开追踪再复现一次，日志会详细很多。

## 许可

**MyCode 免费使用，但并非开源软件。** 你可以在任意数量的自有设备上安装使用，个人用途和商业用途均不收费；但不得再分发、修改、反编译，也不得将其并入其他产品或用于向第三方提供服务。

完整条款见 [LICENSE](LICENSE)，中文参考译本见 [LICENSE.zh_CN](LICENSE.zh_CN)。本项目包含的第三方开源组件及其署名见 [NOTICE](NOTICE)。
