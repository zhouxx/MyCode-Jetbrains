# MyCode

**English** | [中文](README.zh_CN.md)

A coding assistant inside your JetBrains IDE. You send a message in the IDE; it calls the model you configured and uses the IDE's own tools to read code, edit files, and run commands. No separate CLI to install.

The tool window sits on the right side of the IDE, titled **MyCode**. In the chat input, set the engine to **MyCode**.

> The source code of this project is not published. This repository is used for releasing versions, documenting usage, and collecting bug reports.
> **Free to use, but not open source** — no charge for personal or commercial use; redistribution and modification are not permitted.
> See [LICENSE](LICENSE) for the full terms; a Chinese reference translation is at [LICENSE.zh_CN](LICENSE.zh_CN).

## Requirements

- JetBrains IDE 2023.3 (`233`) through 2026.3 (`263.*`) — IDEA, PyCharm, WebStorm, GoLand, and others
- Git plugin (bundled with JetBrains IDEs; nothing to install)

## Sending a message

1. Open **MyCode** on the right side.
2. Make sure the engine is **MyCode**, and pick a model. If you don't have a model yet, do "Configuring models" below first.
3. Type your question and send it.
4. To bring in the code you're looking at: select it in the editor and press `Ctrl+Alt+K` (macOS: `Cmd+Alt+K`).

Editing files, running commands, and reaching outside the workspace each raise a permission prompt. Choose allow, deny, or remember the decision for this time / this session / this workspace / all workspaces. The default mode is to ask.

## Configuring models

**Settings → Model Configuration**, or edit the file directly:

- macOS / Linux: `~/.mycode/config.json`
- Windows: `%USERPROFILE%\.mycode\config.json`

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

| Field | Meaning |
| --- | --- |
| provider id | A key under `providers`. Starts with a letter or digit; may contain `.` `_` `-` |
| `protocol` | `openai` or `anthropic`. If omitted, an OpenAI-compatible API is assumed |
| `baseUrl` | API root URL |
| `apiKey` | The key. Some gateways use `authToken` instead |
| `models` | Model id → `context` (context tokens) and `output` (per-response output limit) |

Saving from the settings page rewrites `modelMap` (model id → provider id) based on the model ids, and leaves the other top-level fields alone. The model list in the input box comes from here. With several providers configured, the model that appears first in the config file becomes the default.

If a model name carries the `1m` context suffix, its context is counted as 1 million tokens.

If the file doesn't exist, MyCode still tries the old location `~/.codemoss/config.json`, looking for a `mycode` or `myagent` section. Once you write a new config, `~/.mycode/config.json` is authoritative.

## What it can do

Within a single turn, the model can call these tools:

| Tool | Purpose |
| --- | --- |
| `read` / `grep` / `glob` | Read files, search contents, find files by name |
| `edit` / `write` | Modify an existing file, write a whole file |
| `bash` | Run commands in the project directory |
| `search_symbol` / `inspect_symbol` / `find_symbol_usages` | Query symbols, definitions, and references via the IDE index |
| `rename_symbol` | Rename using the IDE's refactoring engine |
| `check_errors` | See the IDE's errors for the current file |
| `webfetch` | Fetch a web page |
| `Skill` | Load a skill's instructions and follow them |
| `askuserquestion` | Ask you a multiple-choice question |

In Java projects, the symbol tools go through the IDE's PSI. Commands and file writes always pass through permissions first.

## Permissions

Permissions look at what a call is doing, not at the tool's name. A single call can touch several capabilities at once; all of them must pass for it to run, and one denial is enough to block it.

| Capability | Examples |
| --- | --- |
| `fs.read` | Reading files, listing directories, searching; read-only commands like `ls` / `cat` / `git status` |
| `fs.write` | Modify, create, delete, move; redirected writes |
| `exec` | Anything that starts a process: `npm`, `python`, `git push`, commands that can't be parsed |
| `net` | `webfetch` and any command that reaches the network |

Default policy:

- Reads inside the workspace are allowed directly.
- Writes inside the workspace, command execution, and network access ask you first.
- Writes to sensitive paths — `.git`, `.ssh`, `.env`, shell config — always ask.
- Dangerous commands (interpreters, package managers, `git push` / `commit` / `reset`, and similar) always ask.

The usual choices in the dialog:

- **Just this once**: allow this call, don't remember it.
- **This session**: expires when the session ends.
- **This workspace**: written to `~/.mycode/permission/workspaces/<id>.json`.
- **All workspaces**: written to `~/.mycode/permission/settings.json`.

Config merge order: built-in defaults → global `settings.json` → current workspace file. When rules collide, deny beats ask, and ask beats allow.

`permissions.defaultMode` defaults to `ask`. Keep it on `ask` for everyday use.

## Skills

A skill is a Markdown file with a description. The model reads it in with the `Skill` tool when it needs to, or you can invoke it from the input box with `/name`.

Locations:

- All projects: `~/.mycode/skills/<name>/SKILL.md`
- Current project only: `{project root}/.mycode/skills/<name>/SKILL.md`

On a name collision, the project skill overrides the user skill. Skills you turn off in **Settings → Skills** are moved to the adjacent `skills-disabled/`, and the agent won't load them.

```markdown
---
name: review
description: Review the current changes against a checklist and call out behavior and test gaps.
---

Look at the git diff first, then give your conclusions against the checklist below…
```

`name` uses lowercase letters, digits, and single hyphens, up to 64 characters, and can't start or end with a hyphen. If you omit `name`, the directory name is used. `description` goes into the skill list, so write it in a way that makes clear when the skill should be used.

## Where data lives

```
~/.mycode/
  config.json              # providers, models, compaction, tracing
  models-dev.json          # model catalog cache
  permission/
    settings.json          # shared across all workspaces
    workspaces/
      <workspaceId>.json   # current project only
  skills/                  # enabled user skills
  skills-disabled/         # user skills turned off in settings
  sessions/                # session jsonl
  traces/                  # logs, when tracing is on

{project root}/.mycode/
  skills/
  skills-disabled/
```

`workspaceId` is the first 16 hex characters of the SHA-256 of the normalized project root path. The real path is recorded in that file's `workspace.root`. Change machines or move the directory and the id changes, so it never reads another project's permission file.

Session files live under `~/.mycode/sessions/<project path>/`, where non-alphanumeric characters in the path become `-`. For example, `D:\Projects\MyProject` maps to the directory name `D--Projects-MyProject`, containing `<sessionId>.jsonl` files.

## When the context grows

As a conversation approaches the model's context limit, MyCode automatically summarizes it and tries to restore recently read file excerpts. This is on by default. To change the behavior, add this to `config.json`:

```json
"compact": {
  "enabled": true,
  "bufferTokens": 8000
}
```

`bufferTokens` is how many tokens short of the context ceiling compaction starts, 8000 by default. Set `enabled` to `false` to turn automatic compaction off. An overly long single tool result is also truncated within its turn, so one `grep` can't fill the window.

## Tracing

No traces are written by default. To diagnose why a turn stalled, turn it on in `config.json`:

```json
"tracing": {
  "enabled": true
}
```

Files are written to `~/.mycode/traces/mycode-<id>.json`. `outputDir` changes the directory.

## Keyboard shortcuts

| Action | Windows / Linux | macOS |
| --- | --- | --- |
| Send selected code to the input box | `Ctrl+Alt+K` | `Cmd+Alt+K` |
| Quick-fix with AI | `Ctrl+Shift+Q` | `Cmd+Shift+Q` |

## Reporting issues

Open an issue in this repository, and include:

- Your IDE name and version
- Your MyCode version (visible in **Settings → Plugins**)
- Steps to reproduce
- Relevant logs (**Help → Show Log in Explorer**)

When diagnosing a turn that gets stuck, turn on tracing as described above and reproduce once more — the log will be far more informative.

## License

**MyCode is free to use, but it is not open source.** You may install and use it on any number of machines you own, at no charge for either personal or commercial purposes. You may not redistribute it, modify it, decompile it, incorporate it into another product, or use it to provide a service to third parties.

The full terms are in [LICENSE](LICENSE), with a Chinese reference translation at [LICENSE.zh_CN](LICENSE.zh_CN). Third-party open-source components and their attributions are listed in [NOTICE](NOTICE).
