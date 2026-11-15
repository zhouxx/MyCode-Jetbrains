# 用户钩子

钩子是你自己写的 **Groovy 脚本**。它跑在插件所在的 IDE 进程里，不另起程序。脚本挂在一轮对话的六个时机上，可以看参数、拦下工具、改参数、改喂回模型的结果、往上下文补一句话，或者结束这一轮。

只想看一眼、不改对话的脚本，用 `println`。这句话进 IDE 的 `idea.log`，不会出现在聊天窗口里。

设置里的 **Hooks** 页按六个时机分组。可以按全部、全局、本项目、启用、停用筛选，也能搜索。每一组右侧的「添加到」只往这个时机复制脚本，再选全局或当前项目。点开一条可以看到它的 jar，并编辑或删除。左边的圆钮是启用和停用，只移动文件。编辑会用 IDE 打开这个 `.groovy`。删除前会确认，确认后连同名目录里的 jar 一起删掉。脚本会被复制进 `mycode` 的启用目录，原来的文件留在原地。

## 文件放哪

两套目录，都只读「启用」的那一侧。`hooks-disabled` 里的脚本不编译、不执行。

| | 启用 | 停用 |
| --- | --- | --- |
| 所有项目 | `~/.mycode/hooks/` | `~/.mycode/hooks-disabled/` |
| 仅当前项目 | `{项目根}/.mycode/hooks/` | `{项目根}/.mycode/hooks-disabled/` |

Windows 上的 `~` 是 `%USERPROFILE%`，例如 `C:\Users\你的用户名\.mycode\hooks\`。

六个时机各是一个子目录，目录名必须是下面这几个，不能自己起名：

```text
~/.mycode/hooks/
  prepareNextTurn/
  prepareNextStep/
  recoverFromModelError/
  beforeToolCall/
  afterToolCall/
  shouldStopAfterStep/
    print-edit.groovy              一个脚本
    print-edit/lib/some.jar        只这个脚本能用的 jar
```

规则：

- 只认对应目录**下一层**的 `*.groovy`。不会进子目录找脚本。
- 以 `.` 或 `_` 开头的文件跳过，用来放草稿。
- 同一目录里按文件名排序执行。想让 `deny-rm` 比 `log-edit` 先跑，把文件名排在前面，例如 `10-deny-rm.groovy`、`20-log-edit.groovy`。
- 项目和全局可以同名。两份都会跑，项目的先跑。停用只影响自己那一份。
- 改文件、开关，都是**下一条你发出的消息**才生效。正在跑的这一轮仍用打开时的那一份。

这个脚本只在 `edit` 跑完后打印参数，不改结果：

```groovy
// ~/.mycode/hooks/afterToolCall/print-edit.groovy
if (hook.call.name() == 'edit') {
    println "[MyCode hook] edit " + hook.call.input()
}
return null
```

日志在本机 IDE 的 `idea.log` 里搜 `[MyCode hook]`。用 Gradle `runIde` 启动时，日志在项目的 `build/idea-sandbox/` 下。

## 脚本怎么写

脚本不用实现接口，也不用写 `class`。入口是变量 `hook`。下面这些类已经默认导入，不用再写 `import`：

`AgentContext`、`AgentMessage`、`AgentToolCall`、`AgentBeforeToolCall`、`AgentToolResult`、`JsonObject`。

每个钩子上都能读这三项：

| 字段 | 含义 |
| --- | --- |
| `hook.cwd` | 本轮工作目录。没有时是空字符串 |
| `hook.sessionId` | 本轮会话 id |
| `hook.env` | 只读的环境变量，见后面一节 |

可以不写 `return`。没写、空文件、或者最后一条是 `if` / 赋值这种没有值的语句时，结果是「不改」。类型不对也当成不改，例如 `beforeToolCall` 里 `return "no"` 不会拦工具，也不会报错。

Groovy 会把**最后一条表达式**的值当作返回值。只想看一眼时，最后写 `return null`。`shouldStopAfterStep` 尤其要注意：最后一行若是 `hook.step.toolCalls().isEmpty()` 这种比较，会被当成「要不要结束这一轮」。

编译失败或运行中抛错：这条脚本当作没改，本轮继续。`idea.log` 里有一条 `[MyCode] groovy hook ...` 的 warn，带文件路径。一个脚本坏了，同目录其它脚本照常跑。

脚本和你本机的代码同一信任级别，能读文件系统，也能碰到 IDE 的 API。它跑在发消息的后台线程上。要读 PSI 或虚拟文件时，自己进 ReadAction。

## 六个时机

同一条链上的脚本一个接一个跑。前一个改过的内容，后一个能看见。

| 目录 | 什么时候 | 你能改什么 |
| --- | --- | --- |
| `prepareNextTurn` | 你这条消息开始时，调模型之前，只一次 | 消息列表 |
| `prepareNextStep` | 每一次调模型之前，一轮里可能多次 | 消息列表 |
| `beforeToolCall` | 某个工具真正执行之前 | 放行、换成新参数，或拦住 |
| `afterToolCall` | 这个工具已经跑完 | 喂回模型的那段结果文本 |
| `recoverFromModelError` | 这次调模型失败了 | 换一份上下文再试一次 |
| `shouldStopAfterStep` | 模型这一步返回之后、工具还没跑 | 结束整轮，或继续 |

顺序是：**当前项目里启用的脚本 → 全局启用的脚本 → 插件自己的行为**（压缩、权限、技能列表等）。项目脚本先拦住时，全局脚本和权限框都不会再跑。

`prepareNextTurn` / `prepareNextStep` 里你追加的消息，排在后面的自动压缩有可能整段换掉。

## 拦下或改写工具：`beforeToolCall`

第一个脚本看到的是模型原来的调用。前一个脚本如果换成了新参数，下一个看到的是新的。

| 方法 | 含义 |
| --- | --- |
| `hook.call.id()` | 这次调用的 id，改不了 |
| `hook.call.name()` | 工具名，例如 `bash`、`edit`、`write`，改不了 |
| `hook.call.input()` | 参数，一个 JSON 对象。键由该工具决定 |

返回值：

| 返回 | 效果 |
| --- | --- |
| `return null` 或 `return hook.allow()` | 不改，问下一个脚本 |
| `return hook.allow(json)` | 参数整份换成这份。没写上的键不保留。权限和真正执行看到的都是这份 |
| `return hook.block('原因')` | 工具不执行。原因作为失败结果交给模型。后面的脚本和权限框都不再跑。前面改过的参数作废 |

`input()` 本身可以改键，但没 `return hook.allow(新对象)` 的话，权限和执行都不认。要改参数时先 `deepCopy()`，再返回。

拦住递归删除：

```groovy
// ~/.mycode/hooks/beforeToolCall/10-deny-rm.groovy
if (hook.call.name() == 'bash') {
    def command = hook.call.input()?.get('command')?.asString ?: ''
    if (command.contains('rm -rf')) {
        return hook.block('refuse recursive delete')
    }
}
return null
```

不许改 `.env`：

```groovy
// ~/.mycode/hooks/beforeToolCall/20-deny-env.groovy
def name = hook.call.name()
if (name == 'edit' || name == 'write') {
    def path = hook.call.input()?.get('file_path')?.asString ?: ''
    def file = path.replace('\\', '/').tokenize('/').last()
    if (file == '.env' || file.startsWith('.env.')) {
        return hook.block('do not edit env files')
    }
}
return null
```

给 bash 命令追一个 `--dry-run`。下一个脚本和权限看到的是追完之后的命令：

```groovy
// {项目}/.mycode/hooks/beforeToolCall/30-dry-run.groovy
if (hook.call.name() != 'bash') {
    return null
}
def json = hook.call.input().deepCopy()
def command = json.get('command')?.asString ?: ''
json.addProperty('command', command + ' --dry-run')
return hook.allow(json)
```

常见参数名：

| 工具 | 参数 |
| --- | --- |
| `bash` | `command` |
| `read` | `file_path`，以及可选的 `offset`、`limit` |
| `edit` | `file_path`、`old_string`、`new_string`，可选 `replace_all` |
| `write` | `file_path`、`content` |

读 JSON 时用 `get('键')?.asString`。键不存在时是 `null`，所以前面加 `?: ''`。

## 改工具结果：`afterToolCall`

工具已经执行完。这条链上每个脚本看到的 `hook.call` 相同，里面的参数是 `beforeToolCall` 换过之后、真正拿去执行的那份。`hook.result` 是当前结果；前一个脚本若换过结果，这里是那一份。

| 方法 | 含义 |
| --- | --- |
| `hook.result.isError()` | 这次是不是失败 |
| `hook.result.content()` | 交给模型的文本 |

| 返回 | 效果 |
| --- | --- |
| `return null` | 不改 |
| `return hook.ok('文本')` | 成功结果，内容换成这段 |
| `return hook.error('文本')` | 失败结果，内容换成这段 |

最后一个脚本留下的文本会写进会话，并喂回模型。

把过长的 `read` 结果截短再给模型：

```groovy
// ~/.mycode/hooks/afterToolCall/truncate-read.groovy
if (hook.call.name() == 'read' && hook.result.content().length() > 8000) {
    return hook.ok(hook.result.content().take(8000) + '\n…(hook truncated)')
}
return null
```

失败时在原因后面补一句你的约定：

```groovy
// ~/.mycode/hooks/afterToolCall/annotate-edit-error.groovy
if (hook.call.name() == 'edit' && hook.result.isError()) {
    return hook.error(hook.result.content() + '\n请先 read 该文件，再按原文做一次最小修改。')
}
return null
```

## 对话开始前补一句话

`prepareNextTurn` 在整轮开始时跑一次。`prepareNextStep` 在每次调模型前再跑，工具来回多的时候会多次执行。两边能用的字段一样。

`hook.context` 是当前上下文：

| 方法 | 含义 |
| --- | --- |
| `systemPrompt()` | 系统提示。脚本换不掉它 |
| `messages()` | 当前消息，不能原地改 |
| `tools()` | 工具名单，只能看 `name()`、`description()`、`inputSchema()`，不能在这里调用工具 |
| `plusMessages(list)` | 在末尾追加，返回一份新的上下文 |
| `withMessages(list)` | 整份换成这份消息，返回一份新的上下文 |

一条消息上常用的是 `role()`（`USER` / `ASSISTANT` / `SYSTEM`）和 `text()`（把文本拼起来）。

最省事的追加是 `hook.plusUserText('…')`，它在末尾加一条 user 文本，并返回新的上下文。这条消息会进会话记录。

用户提到 README 时，补一条提醒：

```groovy
// {项目}/.mycode/hooks/prepareNextTurn/readme.groovy
def messages = hook.context.messages()
def last = messages.isEmpty() ? null : messages.last()
if (last != null && last.role().name() == 'USER' && last.text().contains('先看 README')) {
    return hook.plusUserText('请先读项目根目录的 README.md，再回答。')
}
return null
```

`prepareNextStep` 适合「每一步都看一眼」。下面这个脚本在模型还没读过文件时补一句，只追加一次，避免每一步都重复：

```groovy
// ~/.mycode/hooks/prepareNextStep/remind-once.groovy
def messages = hook.context.messages()
def already = messages.any { it.role().name() == 'USER' && it.text().contains('先列出你要改的文件') }
if (already) {
    return null
}
return hook.plusUserText('先列出你要改的文件，确认后再 edit。')
```

返回 `null` 表示消息列表保持上一份。返回的 `AgentContext` 只会换消息；系统提示和工具名单仍是原来的。

## 模型调用失败时：`recoverFromModelError`

调模型抛错时才会进这里。每个还在跑的脚本看到的是**同一份**失败现场，不是上一个脚本改过的结果。

| 字段 | 含义 |
| --- | --- |
| `hook.error` | 这次的异常。看 `message`、`getClass().simpleName` |
| `hook.context` | 失败时的上下文，用法和上一节相同 |

返回 `null` 表示没处理，问下一个脚本。返回一份 `AgentContext` 就用它重试当前这一步，后面的脚本不再跑。插件自己的超长补救排在用户脚本后面：你先返回了新上下文，它就不会再跑；你返回 `null`，它照旧。

```groovy
// ~/.mycode/hooks/recoverFromModelError/retry-shorter.groovy
def message = hook.error?.message ?: ''
if (message.toLowerCase().contains('overloaded') || message.contains('rate limit')) {
    return hook.plusUserText('上一次请求失败了。请缩短回答，先用三句话说明下一步，不要调用工具。')
}
return null
```

## 结束这一轮：`shouldStopAfterStep`

模型这一步已经返回，工具还没执行。你看到的是模型原文，不是 `beforeToolCall` 改过的参数。

| 方法 | 含义 |
| --- | --- |
| `hook.step.text()` | 模型正文，没有时是空字符串 |
| `hook.step.thinking()` | 思考内容 |
| `hook.step.toolCalls()` | 模型这次发出的调用，字段同 `hook.call` |
| `hook.step.usage()` | token 用量，可能是 `null`。有值时可读 `inputTokens()`、`outputTokens()` |

返回 `true` 结束整轮，后面的脚本不再跑，这一步里的工具也不会执行。返回 `false` 或 `null` 则问下一个脚本。

模型自己宣布做完、并且没有再调工具时收工：

```groovy
// ~/.mycode/hooks/shouldStopAfterStep/stop-when-done.groovy
if (hook.step.toolCalls().isEmpty() && hook.step.text().contains('任务完成')) {
    return true
}
return false
```

只想看用量、不结束时，最后必须是 `return false` 或 `return null`。写成下面这样会在「没有工具调用」时把整轮停掉：

```groovy
// 这行本身就是返回值
hook.step.toolCalls().isEmpty()
```

## 环境变量

不要在脚本正文里写 `${DEPLOY_TOKEN}` 这种占位符。Groovy 的 `"hello ${name}"` 是语言本身的字符串，不是环境变量展开。密钥放在 `~/.mycode/config.json` 的 `env` 里，和 Bash 用的是同一份。

脚本用 `hook.env` 读。查找顺序：

1. IDE 里手动配置的代理（`http_proxy`、`https_proxy`、`all_proxy` 等）。
2. `config.json` 里仍开着的自定义变量。关掉的读不到。
3. 本轮内置值：`MYCODE_PROJECT_DIR`、`CLAUDE_PROJECT_DIR`、`MYCODE_SESSION_ID`、`CLAUDE_SESSION_ID`。项目目录就是本轮的 `hook.cwd`。
4. 上面都没有时，再读 IDE 进程自己的环境，所以 `PATH`、`HOME` 通常还在。

钩子不是技能调用，没有 `MYCODE_SKILL_DIR`。读到的值不能写回去，也不会改 `config.json` 或 IDE 的环境。

```groovy
def token = hook.env.DEPLOY_TOKEN
def root = hook.env.MYCODE_PROJECT_DIR
def path = hook.env.PATH
def proxy = hook.env['http_proxy']
```

没有这个名字时是 `null`。名字里有特殊字符时用方括号。

## 给一个脚本加 jar

每个脚本只用自己旁边的 jar，不和别的脚本共用，这样两份不同版本不会挤在一起。

```text
~/.mycode/hooks/beforeToolCall/check-license.groovy
~/.mycode/hooks/beforeToolCall/check-license/lib/license-client.jar
```

脚本里自己 `import`。插件不会下载 Maven / Gradle 依赖，也不支持 `@Grab`。Groovy 运行时已经在插件里，不要再往 `lib/` 放 `groovy.jar`。

设置里关掉这个脚本时，`.groovy` 和旁边的 `check-license/` 目录会一起移到 `hooks-disabled`。没有这个目录时，只移动脚本文件。

```groovy
// ~/.mycode/hooks/beforeToolCall/check-license.groovy
import com.example.License

if (hook.call.name() == 'bash') {
    def command = hook.call.input()?.get('command')?.asString ?: ''
    if (!License.allowed(command)) {
        return hook.block(License.reason())
    }
}
return null
```

`License` 只是你 jar 里的类名示例。换 jar 或改脚本之后，下一条消息会重新加载。

## 和权限的关系

`beforeToolCall` 在权限确认之前。脚本 `block` 了，就不会再弹权限框。脚本 `allow(新参数)` 了，权限看的是新参数，例如你追了 `--dry-run`，问你的那次命令是追完之后的。

权限仍然会跑，除非你已经 `block`。钩子不能代替权限开关。
