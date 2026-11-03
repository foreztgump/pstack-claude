# Pi tool mapping for pstack

pstack skills are written in Claude Code tool language (the `Skill` tool, the `Agent` tool, `AskUserQuestion`, Claude model names). On Pi the skills are the same files, loaded from the pstack package, and the package's Pi extension registers tools named after the Claude ones, so most names map one to one. Read this when a pstack skill names a Claude tool, a driver or bundled skill, or a Claude model. This file is Pi-specific.

This fork assumes the pi-subagents package is installed and the override sheet has `pi agent tools: off`, so pstack registers no `agent`, `send_message`, `list_agents`, or `stop_agent`. Every subagent row and section below, the model names included, is written for pi-subagents' one `subagent` tool. It is a fork-only change and not upstream pstack's mapping.

## Tool actions

| pstack / Claude action | Pi equivalent |
|------------------------|---------------|
| Read a file | `read` |
| Create a file | `write` |
| Edit a file | `edit` (exact text replacement) |
| Delete a file | `bash` (`rm`) |
| Run a shell command | `bash` |
| Search file contents / find files | `grep`, `find`, and `ls` when enabled, otherwise `bash` (`rg`, `grep`, `find`, `ls`). Pi enables only `read`, `bash`, `edit`, and `write` by default. |
| Fetch a URL | `bash` with `curl` / `wget` |
| Search the web | Pi has no web search tool. Use an MCP server that provides one. |
| Invoke a skill (the `Skill` tool, `/command`) | Skills load natively. The model reads a skill when its description matches; `/skill:<name>` forces one. |
| Dispatch a subagent (the `Agent`/`Task` tool) | `subagent({ agent, task, ... })`, with `agent` an agent file name and `task` the brief (see Subagent policy). If `subagent` is not in your tool list, call `subagents_enable({})` first. |
| Dispatch N parallel subagents in one turn | N `subagent` calls in one response. This profile runs every call in the background whatever `async` says; never pass `async: false`. |
| Wait for a subagent result | A call returns a run id at once. The completion arrives as a message (`Background task completed: ...`) naming the agent and its final text, after your current tool calls or as a new turn when you are idle. Do not poll or loop `sleep`; do other work or end your turn. When you need a result inside this turn, `bg_wait({ id })` blocks for that run, `bg_wait({ all: true })` for every active run, and `bg_wait({})` for the first to finish. |
| Continue or steer a subagent (`SendMessage`) | `subagent({ action: "steer", id, message })` to a running run, `subagent({ action: "resume", id, message })` to a finished one. `id` is the run id the launch returned. |
| List subagents | `subagent({ action: "status" })` lists the active runs, and `{ action: "status", id }` reports one. `{ action: "status", id, options: { view: "transcript", lines: 80 } }` tails its transcript, and `{ action: "list" }` lists the agent files you can dispatch. The user can watch runs with `/subagents-fleet`. |
| Stop a subagent | `subagent({ action: "stop", id })` ends a run. `{ action: "interrupt", id }` pauses it so that `resume` can continue it. |
| Track tasks (the todolist; `TaskCreate` / `TaskUpdate`, or `TodoWrite` on Claude Code) | Pi has no task-tracking tool. Keep the `todo.md` checklist poteto-mode describes for that case. |
| Ask the human a fixed-choice question (`AskUserQuestion`) | `ask_user_question`, same shape. Without an interactive UI (print or JSON mode), and in a child agent, it returns an error, so ask in plain text. When the Pi override sheet has `pi ask tool: off`, another extension provides `ask_user_question` and its behavior is that extension's; if the session declares no such tool, ask in plain text. |
| Schedule a self-paced re-invocation (`ScheduleWakeup`) | `schedule_wakeup`, same shape. A due wakeup stays pending until the session is idle, including during manual compaction. `noop: true` only labels the wakeup, which is still scheduled, and `stop: true` cancels the pending wakeup and needs no other field. The prompt arrives as a `pstack-wakeup` custom message, not a user message, so a permission check does not read it as the user's request. A `/skill:` or prompt-template prompt arrives expanded, as Pi expands one the user types. A `/loop` prompt runs the loop, and what the loop sends also arrives as custom messages. Any other command, an extension's included, arrives as plain text and does not run. In print and JSON mode, and in a child agent, scheduling returns an error, because Pi exits when the run ends and the wakeup could never fire. After a `/loop` command ends or replaces the loop during a run, that run is refused every wakeup until it settles, whatever the prompt, and Pi tells the user. No other wakeup is refused. |
| Read this workspace's session transcripts (Claude Code's `${CLAUDE_CONFIG_DIR:-$HOME/.claude}/projects/<encoded-cwd>/`) | Pi sessions live in `~/.pi/agent/sessions/--<cwd>--/` (under `$PI_CODING_AGENT_DIR` when set), where `<cwd>` is the working directory without its leading `/` and with each `/` as `-`. Subagent sessions pi-subagents started live in `<agent dir>/sessions/<encoded parent cwd>/<parent session id>/<run id>/run-0/session.jsonl`. Each file opens with a `session` header line; every later line is an entry with an `id` and a `parentId`, and a message entry is `type: "message"` with `message.role` `user`, `assistant`, or `toolResult`. A session keeps abandoned branches in the same file, so follow `parentId` back from the last entry for the conversation that happened. The same privacy rule applies: read only this workspace's directory, never a glob across `sessions/`. |

`ask_user_question` and `schedule_wakeup` come from the pstack Pi extension, which `pi install` of the pstack package loads, and `subagent`, `bg_wait`, and `subagents_enable` come from pi-subagents. A setup without pi-subagents has no subagent tool, and the fan-out skills (`interrogate`, `why`, `how`, `arena`, `reflect`) degrade to a single sequential pass. A required independent review does not degrade. It stays blocked, as poteto-mode's [Subagents](../SKILL.md#subagents) section says.

## Subagent policy

This section assumes pi-subagents 0.84.0 (see the top of this file). poteto-mode's Subagents section applies on Pi through the `subagent` tool:

- `agent` is an agent file's name, and `task` is the brief. `subagent_type` maps to `agent`: `pstack:poteto-agent`, `pstack:comment-sicko`, `pstack:thermo-nuclear-review-subagent`, and `pstack:thermo-nuclear-code-quality-review-subagent` are the agent files `poteto-agent`, `comment-sicko`, `thermo-nuclear-review-subagent`, and `thermo-nuclear-code-quality-review-subagent`, with no `pstack:` prefix. poteto-mode's note that the bare `poteto-agent` errors is Claude Code's plugin namespace; here the prefixed name is not an agent file name there. A call with no `agent` is an error, so `general-purpose` has no equivalent. Pick the profile agent whose description fits the job, a writing agent for edits and `investigator` for read-only research, and put anything else the job needs in the brief. Claude Code's `Explore` and `Plan` map the same way. An unknown `agent` errors and lists the valid names, and `{ action: "list" }` shows each agent with its description.
- Read-only has no parameter. Whether a child can edit is set by `tools:` in its agent file, so a skill that sets `readonly: true`, such as the `how` panel, dispatches a read-only agent (`explorer` or `investigator`). Check with `{ action: "list", options: { capabilities: true } }` when unsure, and say in the brief that the child must not change files.
- Children start fresh (`context: "fresh"`) in your working directory, with the brief as their only history, so keep passing file pointers, not inlined context. A child cannot start subagents of its own: this profile limits nesting to depth 1. A skill that fans out must run in the main session, and one that a child runs falls back to a single sequential pass.
- `worktree: true` runs the child in its own git worktree branched from `HEAD`. The checkout it branches from must be clean, so commit the scope first; a worktree starts from `HEAD` and lacks uncommitted changes, which is why `no-comments` runs without it. The worktree is removed when the child finishes, and its changes are captured as a patch file under `worktree-diffs/` in the run's `asyncDir`, with a handoff manifest in `artifactPaths`. Inspect and apply the patch yourself; the branch is not kept.
- Every call runs in the background: this profile forces it, because a foreground child loads no safety or permission extensions. Never pass `async: false`. The launch returns `Async: <agent> [<run id>]  Detached` and the completion notice arrives on its own, as the Wait row above says. Do not poll with `sleep`, `tail`, or repeated `status` calls.
- Stop an abandoned run with `{ action: "stop", id }` before you start its replacement. `stop` ends a top-level run for good, and `resume` cannot revive it. `steer` reaches a running child after its current tool call and returns a receipt (`delivered`, `pending`, `failed`); `{ action: "steer", id, message, options: { mode: "follow_up" } }` waits for the next turn boundary. `resume` starts a new child from the finished run's saved session, so it keeps the earlier context but is not the live process, and it needs a persisted session file.
- A permission ask from a child goes to your session and names the child's agent. The `subagent` call itself is judged at launch on its `task` text, so a brief that includes a destructive command can be denied or deferred before the child starts. That is the judge working; do not reword a brief to get past it.
- Keep the rest of the policy unchanged. Give each writing worker its own worktree, review every subagent's diff yourself.

## Model names

Skills name models by the Claude aliases in their Models sections. pi-subagents has no aliases: the `subagent` tool's `model` is an exact `provider/id` (a bare id resolves only when it is unique), so translate each alias before the call. Use the sheet's role value when it is already a full ID. Otherwise use the alias's entry in the sheet's `pi models:` line, and with no entry the column of this table for the provider the session's current model comes from, and in the `anthropic` column for any other provider:

| Alias | `anthropic` | `openai` | `openai-codex` |
| --- | --- | --- | --- |
| `opus` | `anthropic/claude-opus-5-5` | `openai/gpt-6.1-sol` | `openai-codex/gpt-6.1-sol` |
| `fable` | `anthropic/claude-fable-5-1` | `openai/gpt-6-astra` | `openai-codex/gpt-6-astra` |
| `sonnet` | `anthropic/claude-sonnet-5-5` | `openai/gpt-6.1-sol` | `openai-codex/gpt-6.1-sol` |
| `haiku` | `anthropic/claude-haiku-5-5` | `openai/gpt-6-luna` | `openai-codex/gpt-6-luna` |

A `pi models: opus=<provider/id>, sonnet=<provider/id>` line in the Pi override sheet points each alias it names at a Pi model, whatever the session's provider. It is the entry to use when the session's provider has no column above and Pi has no credentials for `anthropic`. Pi warns that Anthropic bills Claude used through Pi per token, as extra usage, even on a Claude subscription.

`subagent` has no separate effort parameter. A role's `@<level>`, or the sheet's `default effort` level, becomes a suffix on the model: `gateway/glm-5.3:high`. `session` adds no suffix, so the child runs at its agent file's `thinking:`, else at the parent's current level. This profile caps thinking at `high`, so a higher level fails the launch: keep sheet levels at `high` or below on Pi, even though `/setup-pstack` offers `xhigh` and `max`. When a skill's dispatch text says to pass an `effort`, put that level in the suffix instead.

Omitting `model` runs the agent on the model its file names, not on the parent's model. A skill's `inherit-parent` or `auto` means the parent's model, so pass the parent's `provider/id`, with the role's `@<level>` as its suffix when the role carries one (`inherit-parent @high` is `<parent provider/id>:high`). An unknown or ambiguous model fails with the closest IDs, and `{ action: "models" }` lists every ID. Diverse-model panels (`arena`, `architect`, `interrogate`, `how` critics, `reflect`) stay diverse only while their models resolve to distinct IDs. Pass each seat's own `model` on its call, whichever agent the skill's row names for that seat; an agent file's pinned model is not what makes the panel diverse. If one model family is all you can reach, vary the reasoning level and note in the verdict that diversity was reduced.

`/setup-pstack` writes the configured model list. On Pi, keep the aliases and remap them with `pi models:`.

## Session routing

The pstack Pi extension adds the poteto-mode mandate, the text the Claude Code and Codex `SessionStart` hook injects, to the system prompt at every agent start, so the mandate survives compaction. It also adds the Pi override sheet, because Pi has no include syntax for context files. The sheet is `pstack-models.md` in the Pi agent directory, which is `$PI_CODING_AGENT_DIR` when set and `~/.pi/agent` otherwise. `session hook: off` in the sheet stops the mandate. Child agents that the `agent` tool starts get the sheet but not the mandate, as Claude Code subagents see CLAUDE.md but run no `SessionStart` hook. Every session, child agents and sessions with the hook off included, also gets a pointer to this file and Claude Code's sentence on making independent tool calls in one response.

Without the extension nothing is injected. Request `poteto-mode` explicitly with `/skill:poteto-mode`, or add a standing instruction to `AGENTS.md`.

## Driver and bundled skills pstack references

The [driver policy](../SKILL.md#non-negotiables) selects the app driver. For skills and drivers named by these workflows, use these Pi equivalents:

| Skill or driver named in pstack | On Pi |
|---------------------------------|-------|
| `run` (drive a CLI/TUI to see a change work) | Pi has no `run` skill. Run the app yourself via `bash` and observe the real output. |
| `verify` (the project's `.claude/skills/verify/`, or Claude Code's bundled `/verify`) | Pi does not discover `.claude/skills/`. Read the project skill's SKILL.md by path, or add `../.claude/skills` to the `skills` list in `.pi/settings.json`. Without a project skill, drive the app as for `run`. |
| Project UI driver | Drive the UI with whatever automation you have, or hand the user a concrete manual check. Do not claim done without observing the artifact. |
| `plugin-dev:skill-development` (Claude's SKILL.md authoring guidance) | Follow Pi's skills documentation and the Agent Skills specification. Keep `name` + `description` frontmatter, name the directory after the skill, and use progressive disclosure. |
| `loop` (recurring/self-paced re-invocation, used by `babysit`) | The extension's `/loop [interval] <prompt>` command. With an interval it re-fires on that cadence. Without one the prompt runs now and the model paces itself with `schedule_wakeup`. `/loop stop` cancels, also while an iteration runs: nothing that run then schedules is armed. A `/loop` given during a run starts once that run ends. |

## Per-skill notes

No skill file carries a Pi line. poteto-mode's Platform Adaptation section points here once, and the extension's system prompt points here in every session, so read a skill's row before you follow that skill. Most skills need only the tables above. These need one more mapping:

| Skill | On Pi |
|-------|-------|
| `poteto-mode` | The todolist falls back to `todo.md`, and the Subagents defaults map through Subagent policy above: `run_in_background: true` is the profile's forced async, and `subagent_type` is `agent`. The Eval and Session pickup playbooks read transcripts from the Pi sessions directory (see Tool actions). |
| `interrogate` | Adversaries dispatch through `subagent`, one `agent` per reviewer seat and the seat's `model` translated as Model names says. A reviewer that must run code in its own checkout gets `worktree: true`, so commit the scope first (see Subagent policy). Keep the panel on distinct models (see Model names). |
| `recall` | Search the Pi sessions directory for this workspace (see Tool actions). Do not list or read `${CLAUDE_CONFIG_DIR:-$HOME/.claude}/projects/`: it holds Claude Code transcripts, not this session's history. Match on `type: "message"` entries and their `message.role`. |
| `show-me-your-work` | Check the log against this run's Pi session file (see Tool actions), not a Claude Code transcript. |
| `setup-pstack` | The Pi sheet is `pstack-models.md` in the Pi agent directory (see Session routing), and the extension loads it, so no include line is needed. List models with `pi --list-models`. For a family whose model is not listed, offer a `pi models:` line (see Model names). The role rows and the five effort levels are identical. The `session hook` line applies on Pi too, where `off` stops the extension adding the mandate. When the user runs another Pi extension that registers `ask_user_question`, add `pi ask tool: off`, so the pstack extension leaves that tool name to it; Pi refuses to start while two extensions register the same name. When the user runs pi-subagents or another Pi extension that provides the subagent tools, add `pi agent tools: off`, so the pstack extension registers none of `agent`, `send_message`, `list_agents` and `stop_agent`. When the session runs on `anthropic`, tell the user that Pi bills Claude per token as extra usage and that each multi-model panel multiplies that cost. |
| `no-comments` | `pstack:comment-sicko` is the `comment-sicko` agent. Pass it the scope, not a request to run `/no-comments`, and run it without `worktree: true`, because a worktree starts from `HEAD` and lacks the scope's uncommitted changes. |
| `teach` | Running `how` and `why` in parallel maps to two `subagent` calls in one response. Pi has no image generation tool, so draw with Mermaid or plain text. |
| `create-verification-skill` | The generated skill lands under `.claude/skills/verify/` on Claude Code. Write it where Pi discovers project skills, `.pi/skills/verify/` or `.agents/skills/verify/`, instead. The app-driving harness is platform-neutral. |
| `maintain-verification-skill` | The parallel per-feature source readers map to `subagent` calls, read-only agents. The project-local skill lives under `.pi/skills/` or `.agents/skills/`, not `.claude/skills/`. |
| `create-formal-verification-skill` | Write the generated skill where Pi discovers project skills, `.pi/skills/<area>/` or `.agents/skills/<area>/`, named for the area it verifies. The verifier and its run command are platform-neutral. |
| `maintain-formal-verification-skill` | Project skills, the formal verification skill among them, live under `.pi/skills/` or `.agents/skills/`. |
| `babysit` | `loop` and `AskUserQuestion` resolve through the tables above. |
| `automate-me` | `plugin-dev:skill-development` resolves through the skills table above. The workspace's transcripts are its Pi sessions directory (see Tool actions). Pi does not discover `.claude/skills/`, so write the mode skill to `.pi/skills/<handle>-mode/SKILL.md` or `.agents/skills/<handle>-mode/SKILL.md` instead. |
| `architect` | The runner panel goes through the **arena** skill, so its `subagent` fan-out and model translation apply here too. |
| `arena` | The parallel candidates and the cross-judge are `subagent` calls, one runner agent with a different `model` per candidate, on the configured aliases translated as Model names says. |
| `how` | The parallel explorers and the explainer are `subagent` calls, and `readonly` becomes a read-only agent (see Subagent policy). |
| `reflect` | The three reviewers and the synthesizer are `subagent` calls. The transcript finder reads Pi sessions too. Pass it this workspace's Pi sessions directory (see Tool actions) in place of Claude Code's projects directory. It follows the session's active branch to its opening prompt. Skill files load from the Pi package directory that `pi list` shows, not `${CLAUDE_CODE_PLUGIN_CACHE_DIR:-${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins}/`, so treat reads under that directory as plugin skill reads. |
| `swarm` | Each worker is a `subagent` call on the configured model. Give each writing worker `worktree: true` or its own output directory (see Subagent policy above). |
| `why` | The parallel investigators and the synthesizer are `subagent` calls. List MCP servers from the tools Pi exposes to the session or `pi mcp list`, not from `.mcp.json` or `claude mcp list`. Spawn the synthesizer only after every investigator's completion notice has arrived, because those notices carry the findings the synthesizer gets. A `steer` or `resume` of an investigator returns only a receipt, not findings, so wait for its notice. |

## Vendored scripts

`skills/poteto-mode/scripts/` ships the `watch-pr` PR watcher and `worktree-audit.mjs`. These scripts use bun and Node.js and run the same on Pi; invoke them through `bash`. They need `bun` and `gh`, or `origin` when the forge is Origin. Run `worktree-audit.mjs` with `node`, as its shebang does. The audit follows each link in a worktree's ancestor directories with `stat` and matches it by the directory it lands on, under bun and node alike. A worktree whose own path bun cannot resolve loses its last chat and lands in `review`. A dangling or looping link is ignored. Any other link that `stat` cannot follow puts every worktree under the link's directory in `review`, with a warning that names the link. `worktree-audit.mjs` finds each worktree's last chat in every runtime's transcript directory that exists: Claude Code's projects under `$CLAUDE_CONFIG_DIR` (default `~/.claude`), Codex's sessions and archived sessions under `$CODEX_HOME` (default `~/.codex`), and Pi's `sessions/` and the pstack extension's subagent sessions under `$PI_CODING_AGENT_DIR` (default `~/.pi/agent`). Pass transcript directories after the repo path to scan others instead. It imports the transcript walker from `skills/reflect/scripts/find-transcript.mjs`, so keep the `reflect` skill installed beside `poteto-mode`.

## Instructions file

Where a pstack skill says "your instructions file", on Pi that is `AGENTS.md` or `CLAUDE.md`, which Pi loads from the working directory and each parent directory, plus `~/.pi/agent/AGENTS.md` for every directory. On Claude Code it is `CLAUDE.md`.
