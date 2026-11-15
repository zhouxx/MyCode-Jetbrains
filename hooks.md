# Hooks

**English** | [中文](hooks.zh_CN.md)

Hooks are **Groovy scripts** you write yourself. They run inside the IDE process that hosts the plugin, not as a separate program. A script attaches to one of six points in a turn. It can inspect arguments, block a tool, change the arguments, change the result sent back to the model, add a sentence to the context, or end the turn.

A script that only looks, and does not change the conversation, uses `println`. That line goes to the IDE's `idea.log`. It does not appear in the chat.

The **Hooks** page in Settings groups scripts by the six timings. You can filter by all, global, this project, enabled, and disabled, and you can search. **Add to**, on the right of a group, copies a script into that timing only; you then choose global or the current project. Open an entry to see its jars, and to edit or delete it. The round button on the left enables or disables the script by moving the file. Edit opens the `.groovy` file in the IDE. Delete asks for confirmation, then removes the script together with the jars in the directory of the same name. The script is copied into MyCode's enabled directory. The original file stays where it was.

## Where the files go

There are two directory pairs. Only the enabled side is read. Scripts in `hooks-disabled` are not compiled and do not run.

| | Enabled | Disabled |
| --- | --- | --- |
| All projects | `~/.mycode/hooks/` | `~/.mycode/hooks-disabled/` |
| Current project only | `{project root}/.mycode/hooks/` | `{project root}/.mycode/hooks-disabled/` |

On Windows, `~` is `%USERPROFILE%`, for example `C:\Users\your-name\.mycode\hooks\`.

Each of the six timings is a subdirectory. The directory name must be one of the names below. You cannot invent your own.

```text
~/.mycode/hooks/
  prepareNextTurn/
  prepareNextStep/
  recoverFromModelError/
  beforeToolCall/
  afterToolCall/
  shouldStopAfterStep/
    print-edit.groovy              one script
    print-edit/lib/some.jar        a jar only this script can use
```

Rules:

- Only `*.groovy` files in the directory **immediately under** that timing are recognized. Scripts are not searched for in subdirectories.
- Files whose names start with `.` or `_` are skipped. Use that for drafts.
- Scripts in the same directory run in filename order. To run `deny-rm` before `log-edit`, put it earlier in the sort, for example `10-deny-rm.groovy` and `20-log-edit.groovy`.
- A project script and a global script may share a name. Both run, and the project script runs first. Disabling one does not affect the other.
- File edits and enable/disable changes take effect on **the next message you send**. The turn already running still uses the copy it opened with.

This script only prints the arguments after `edit` finishes. It does not change the result:

```groovy
// ~/.mycode/hooks/afterToolCall/print-edit.groovy
if (hook.call.name() == 'edit') {
    println "[MyCode hook] edit " + hook.call.input()
}
return null
```

Search the local IDE's `idea.log` for `[MyCode hook]`. When the IDE is started with Gradle `runIde`, the log is under the project's `build/idea-sandbox/`.

## How to write a script

A script does not implement an interface and does not need a `class`. The entry point is the variable `hook`. These classes are imported by default, so you do not write `import` for them:

`AgentContext`, `AgentMessage`, `AgentToolCall`, `AgentBeforeToolCall`, `AgentToolResult`, `JsonObject`.

Every hook can read these three:

| Field | Meaning |
| --- | --- |
| `hook.cwd` | Working directory for this turn. An empty string when there is none |
| `hook.sessionId` | Session id for this turn |
| `hook.env` | Read-only environment variables. See the later section |

You can omit `return`. If you omit it, the file is empty, or the last statement is something with no value such as `if` or an assignment, the result is "do not change". A value of the wrong type is also treated as "do not change". For example, `return "no"` inside `beforeToolCall` does not block the tool, and it does not raise an error.

Groovy uses the value of the **last expression** as the return value. When you only want to look, end with `return null`. This matters especially for `shouldStopAfterStep`: if the last line is a comparison such as `hook.step.toolCalls().isEmpty()`, that comparison is taken as whether to end the turn.

If compilation fails or the script throws while running, that script is treated as having changed nothing, and the turn continues. `idea.log` gets a warn line, `[MyCode] groovy hook ...`, with the file path. One broken script does not stop the others in the same directory.

A script is trusted at the same level as code on your machine. It can read the file system and can reach the IDE's API. It runs on the background thread that sends the message. When you read PSI or a virtual file, enter a ReadAction yourself.

## The six timings

Scripts on the same chain run one after another. A later script sees what an earlier script changed.

| Directory | When | What you can change |
| --- | --- | --- |
| `prepareNextTurn` | When your message starts, before the model is called, once only | The message list |
| `prepareNextStep` | Before every model call. A turn may do this more than once | The message list |
| `beforeToolCall` | Before a tool actually runs | Allow it, replace the arguments, or block it |
| `afterToolCall` | After this tool has finished | The result text sent back to the model |
| `recoverFromModelError` | This model call failed | Swap in a new context and retry once |
| `shouldStopAfterStep` | After the model returns this step, before tools run | End the whole turn, or continue |

The order is: **enabled scripts in the current project → enabled global scripts → the plugin's own behavior** (compaction, permissions, the skill list, and so on). If a project script blocks first, the global scripts and the permission dialog do not run.

Messages you append in `prepareNextTurn` or `prepareNextStep` may be replaced wholesale by the automatic compaction that runs later.

## Block or rewrite a tool: `beforeToolCall`

The first script sees the call the model originally made. If an earlier script replaced the arguments, the next script sees the new ones.

| Method | Meaning |
| --- | --- |
| `hook.call.id()` | Id of this call. You cannot change it |
| `hook.call.name()` | Tool name, such as `bash`, `edit`, or `write`. You cannot change it |
| `hook.call.input()` | Arguments, as a JSON object. The keys depend on the tool |

Return value:

| Return | Effect |
| --- | --- |
| `return null` or `return hook.allow()` | Do not change. Ask the next script |
| `return hook.allow(json)` | Replace the arguments with this object. Keys you leave out are not kept. Permissions and the real execution both see this object |
| `return hook.block('reason')` | Do not run the tool. The reason is given to the model as a failed result. Later scripts and the permission dialog do not run. Arguments changed earlier are discarded |

You can change keys on `input()` itself, but permissions and execution ignore that unless you `return hook.allow(the new object)`. To change arguments, `deepCopy()` first, then return.

Block a recursive delete:

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

Do not edit `.env`:

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

Append `--dry-run` to a bash command. The next script, and the permission check, see the command after the append:

```groovy
// {project}/.mycode/hooks/beforeToolCall/30-dry-run.groovy
if (hook.call.name() != 'bash') {
    return null
}
def json = hook.call.input().deepCopy()
def command = json.get('command')?.asString ?: ''
json.addProperty('command', command + ' --dry-run')
return hook.allow(json)
```

Common argument names:

| Tool | Arguments |
| --- | --- |
| `bash` | `command` |
| `read` | `file_path`, and optional `offset` and `limit` |
| `edit` | `file_path`, `old_string`, `new_string`, and optional `replace_all` |
| `write` | `file_path`, `content` |

When reading JSON, use `get('key')?.asString`. A missing key is `null`, so put `?: ''` in front of the use.

## Change a tool result: `afterToolCall`

The tool has already finished. Every script on this chain sees the same `hook.call`. The arguments in it are the ones that were actually executed, after any replacement in `beforeToolCall`. `hook.result` is the current result. If an earlier script replaced the result, this is that replacement.

| Method | Meaning |
| --- | --- |
| `hook.result.isError()` | Whether this call failed |
| `hook.result.content()` | The text given to the model |

| Return | Effect |
| --- | --- |
| `return null` | Do not change |
| `return hook.ok('text')` | A successful result, with this text |
| `return hook.error('text')` | A failed result, with this text |

The text left by the last script is written into the session and sent back to the model.

Shorten an oversized `read` result before it reaches the model:

```groovy
// ~/.mycode/hooks/afterToolCall/truncate-read.groovy
if (hook.call.name() == 'read' && hook.result.content().length() > 8000) {
    return hook.ok(hook.result.content().take(8000) + '\n…(hook truncated)')
}
return null
```

On failure, append a sentence of your own after the reason:

```groovy
// ~/.mycode/hooks/afterToolCall/annotate-edit-error.groovy
if (hook.call.name() == 'edit' && hook.result.isError()) {
    return hook.error(hook.result.content() + '\nRead the file first, then make one minimal edit against the original text.')
}
return null
```

## Add a sentence before the model runs

`prepareNextTurn` runs once, at the start of the whole turn. `prepareNextStep` runs again before every model call, so a turn with many tool round-trips runs it more than once. Both expose the same fields.

`hook.context` is the current context:

| Method | Meaning |
| --- | --- |
| `systemPrompt()` | The system prompt. A script cannot replace it |
| `messages()` | The current messages. You cannot edit them in place |
| `tools()` | The tool list. You can only read `name()`, `description()`, and `inputSchema()`. You cannot call a tool from here |
| `plusMessages(list)` | Append to the end, and return a new context |
| `withMessages(list)` | Replace the messages with this list, and return a new context |

On a message, the fields you usually need are `role()` (`USER` / `ASSISTANT` / `SYSTEM`) and `text()` (the text pieces joined together).

The shortest append is `hook.plusUserText('…')`. It adds one user text message at the end and returns the new context. That message is recorded in the session.

When the user mentions the README, add a reminder:

```groovy
// {project}/.mycode/hooks/prepareNextTurn/readme.groovy
def messages = hook.context.messages()
def last = messages.isEmpty() ? null : messages.last()
if (last != null && last.role().name() == 'USER' && last.text().contains('read the README first')) {
    return hook.plusUserText('Read README.md at the project root before you answer.')
}
return null
```

`prepareNextStep` fits a check you want on every step. The script below adds one sentence while the model has not read a file yet, and appends it only once, so it is not repeated on every step:

```groovy
// ~/.mycode/hooks/prepareNextStep/remind-once.groovy
def messages = hook.context.messages()
def already = messages.any { it.role().name() == 'USER' && it.text().contains('list the files you will change') }
if (already) {
    return null
}
return hook.plusUserText('List the files you will change, and edit only after that is confirmed.')
```

Returning `null` keeps the previous message list. An `AgentContext` you return replaces only the messages. The system prompt and the tool list stay as they were.

## When a model call fails: `recoverFromModelError`

This runs only when calling the model throws. Each script that still runs sees the **same** failure. It does not see a result an earlier script changed.

| Field | Meaning |
| --- | --- |
| `hook.error` | The exception. Read `message` and `getClass().simpleName` |
| `hook.context` | The context at the time of the failure. Same use as the previous section |

Returning `null` means you did not handle it, and the next script is asked. Returning an `AgentContext` retries the current step with that context, and later scripts do not run. The plugin's own recovery for an oversized context comes after user scripts. If you return a new context first, that recovery does not run. If you return `null`, it runs as usual.

```groovy
// ~/.mycode/hooks/recoverFromModelError/retry-shorter.groovy
def message = hook.error?.message ?: ''
if (message.toLowerCase().contains('overloaded') || message.contains('rate limit')) {
    return hook.plusUserText('The last request failed. Keep the answer short: say the next step in three sentences, and do not call tools.')
}
return null
```

## End the turn: `shouldStopAfterStep`

The model has returned this step, and the tools have not run yet. You see the model's original output, not arguments rewritten by `beforeToolCall`.

| Method | Meaning |
| --- | --- |
| `hook.step.text()` | The model's text. An empty string when there is none |
| `hook.step.thinking()` | The thinking text |
| `hook.step.toolCalls()` | The calls the model made this time. The fields match `hook.call` |
| `hook.step.usage()` | Token usage. May be `null`. When present, read `inputTokens()` and `outputTokens()` |

Returning `true` ends the whole turn. Later scripts do not run, and the tools in this step do not run either. Returning `false` or `null` asks the next script.

Stop when the model says it is done and did not call another tool:

```groovy
// ~/.mycode/hooks/shouldStopAfterStep/stop-when-done.groovy
if (hook.step.toolCalls().isEmpty() && hook.step.text().contains('task complete')) {
    return true
}
return false
```

If you only want to look at usage and not end the turn, the last line must be `return false` or `return null`. Written like this, the whole turn stops whenever there is no tool call:

```groovy
// this line itself is the return value
hook.step.toolCalls().isEmpty()
```

## Environment variables

Do not write a placeholder such as `${DEPLOY_TOKEN}` in the script body. Groovy's `"hello ${name}"` is the language's own string interpolation, not environment-variable expansion. Put secrets in the `env` object of `~/.mycode/config.json`. That is the same map Bash uses.

A script reads them through `hook.env`. Lookup order:

1. Proxy values configured manually in the IDE (`http_proxy`, `https_proxy`, `all_proxy`, and similar).
2. Custom variables in `config.json` that are still enabled. Ones you turned off cannot be read.
3. Built-in values for this turn: `MYCODE_PROJECT_DIR`, `CLAUDE_PROJECT_DIR`, `MYCODE_SESSION_ID`, `CLAUDE_SESSION_ID`. The project directory is this turn's `hook.cwd`.
4. If none of the above has the name, the IDE process's own environment is read, so `PATH` and `HOME` are usually still there.

A hook is not a skill invocation, so there is no `MYCODE_SKILL_DIR`. Values you read cannot be written back, and they do not change `config.json` or the IDE's environment.

```groovy
def token = hook.env.DEPLOY_TOKEN
def root = hook.env.MYCODE_PROJECT_DIR
def path = hook.env.PATH
def proxy = hook.env['http_proxy']
```

A missing name is `null`. Use brackets when the name contains a special character.

## Add a jar for one script

Each script uses only the jars beside itself. It does not share them with other scripts, so two different versions do not collide.

```text
~/.mycode/hooks/beforeToolCall/check-license.groovy
~/.mycode/hooks/beforeToolCall/check-license/lib/license-client.jar
```

Write the `import` in the script yourself. The plugin does not download Maven or Gradle dependencies, and it does not support `@Grab`. The Groovy runtime is already inside the plugin. Do not put `groovy.jar` in `lib/`.

When you turn this script off in Settings, the `.groovy` file and the `check-license/` directory beside it move together to `hooks-disabled`. If that directory is absent, only the script file moves.

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

`License` is only an example of a class name inside your jar. After you change the jar or the script, the next message reloads it.

## How this relates to permissions

`beforeToolCall` runs before the permission prompt. If a script `block`s the call, the permission dialog does not appear. If a script `allow`s new arguments, permissions see those new arguments. If you appended `--dry-run`, the command you are asked about is the one after the append.

Permissions still run, unless you already `block`ed. A hook does not replace the permission switches.
