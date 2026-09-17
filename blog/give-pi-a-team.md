---
description: Give Pi named roles, scoped tasks, and automatic handoffs instead of managing models and prompting agents to continue.
date: 2026-09-17
tags: tooling
---

# Give Pi a Team

I kept using a single model, even tho I knew it wasn't the right thing to do. Not because switching models is hard. Because deciding to switch is another thing to do, while I'm trying to get something built.

Which model was good at planning again? Which one should handle the UI? Has that changed since the last release? And am I really going to interrupt this conversation to change models now?

Usually, no. I'd keep going with whatever was already selected. With whatever was already in my context.

The thing I wanted wasn't a better model picker. I wanted to ask an engineer to implement something, or a designer to work on the interface, without having to pick a model.

Turns out, this is quite doable. Not that hard even. And you already know the fundamentals: skills & (system) prompts.

## Give the Models a Role

Subagent sounds fancy. But it's just another run of the same kind of agent you're already working with. The main agent hands it a task, it does the work, and returns a result. With [pi-subagents](https://github.com/tintinweb/pi-subagents), you can also ask for one yourself through an `@mention`.

What makes the `@mention` extra useful is the custom profile behind it. Without that, I'd mostly be asking another agent to do the same thing as the main one. With it, `@engineer` gets instructions for implementation, while `@designer` gets instructions for the interface. Each can have its own model, thinking level, and available tools.

The profile is basically a custom system prompt, plus settings. Skills add task-specific instructions when loaded. Neither adds new intelligence. We're writing down what an engineer or designer should do, then giving those instructions a name we can `@call`.

Let's start with installing pi-subagents. Assuming you already have Pi and an authenticated model, install it with:

```shell
pi install npm:@tintinweb/pi-subagents
mkdir -p ~/.pi/agent/agents
```

Restart or `/reload` Pi after installing.

Now let's create our first (sub)agent. Like almost anything with LLMs, an agent profile is a simple markdown file. This profile uses `deepseek/deepseek-v4-flash` because that's the model I chose for this role. Save it as `~/.pi/agent/agents/engineer.md`.

```markdown
---
name: engineer
description: Implements one scoped task, with tests.
model: deepseek/deepseek-v4-flash
tools: read, write, edit, bash, grep, find, ls
prompt_mode: append
inherit_context: false
---

Implement the assigned task and stop. Stay inside its allowed files.
Run the relevant tests. Report what changed, what passed, and blockers.
Do not fix adjacent issues just because you noticed them.
```

Let's go through the frontmatter:

- `name` is the thing you can mention. In this case, `@engineer`. I recommend keeping it in sync with the filename.
- `model` picks the provider and model for this role, using `provider/model`. List available models with `pi --list-models`
- `tools` limits the built-in tools the engineer can use. See [pi.dev](https://pi.dev/docs/latest/sdk#tools) for available tools.
- `prompt_mode: append` adds these instructions to Pi's normal agent prompt instead of replacing it.
- `inherit_context: false` starts the agent without the entire parent conversation. It still gets its own instructions and the project context that applies to it.

Trigger `/reload` after saving this file, then open `/agents` → Agent types. You should see `engineer` and the model Pi resolved for it. Or alternatively, type `@eng` and completion should suggest `@engineer`.

After this, I can type:

```text
@engineer Add tests for the CSV parser's empty-input behavior.
```

For a designer, create `designer.md` using the same format. Change `name` to `designer`, pick a model that's better suited for UI work (for example `openai-codex/gpt-6-astra`), describe its presentation work, and replace the body with instructions to own layout, interaction, and component interfaces. 

Instead of switching models, I talk to `@engineer` or `@designer`. The chat feels more natural, and the handover contains only the context that role needs. I don't need to clear my session through `/new` all the time.

## Keep the Work Moving

Roles solved the model-switching problem. They didn't solve the next problem: I still had to tell the agents, "continue".

Sounds familiar? Some work gets done, but the agent ends with "just say continue and I'll get that done too". Exhausting. You come back from your coffee break, and that's the message you're running into.

My solution is to introduce a CTO and give it a task system. The CTO inspects the repository and breaks my goal into scoped tasks. The task board stores their dependencies and keeps the handoffs moving.

These two need each other. A CTO without a task system returns a plan I still have to advance. A task list without a CTO means I have to write every task by hand.

[pi-tasks](https://github.com/tintinweb/pi-tasks) provides that shared task board and its dependencies. Install it with:

```shell
pi install npm:@tintinweb/pi-tasks
```

Create or merge this into `~/.pi/agent/tasks-config.json`:

```json
{
  "taskScope": "project",
  "autoCascade": true
}
```

Restart or `/reload` Pi after saving the config.

Both settings matter. Tasks default to session storage, with cascade disabled. Project storage puts the board at `.pi/tasks/tasks.json`, where agents with access to the task tools can share it. With session storage, a subagent can create tasks the main agent cannot see.

These are global defaults. `/tasks` → Settings can override them for a project, saving to `.pi/tasks-config.json`.

That gives the CTO somewhere to record the plan. It doesn't create the plan or start the work by itself. Create `~/.pi/agent/agents/cto.md`:

```markdown
---
name: cto
description: Reads the repo and creates small, executable tasks.
model: deepseek/deepseek-v4-flash
tools: read, bash, grep, find, ls, ext:pi-tasks
disallowed_tools: TaskExecute, TaskStop
prompt_mode: replace
inherit_context: false
---

Plan; never implement or start execution. Read the repository,
its AGENTS.md, and relevant build and source files first.
Check TaskList before creating work.

Use TaskCreate for small tasks that can finish in one run.
Every task needs an agentType, exact allowed files, acceptance
criteria, and all context the worker needs beyond the repository.
Use only configured agent types.

Wire dependencies with TaskUpdate/addBlockedBy. Tasks touching
one file must not run concurrently. Report which IDs are ready.
Assume the worker has not seen this planning conversation.
```

Again, choose your own authenticated model. The `ext:pi-tasks` selector exposes the installed extension's tools; `disallowed_tools` removes execution and stopping from this role.

The split is deliberate. The CTO inspects the repository, creates tasks with an `agentType` through `TaskCreate`, and connects prerequisites through `TaskUpdate` with `addBlockedBy`. It cannot call `TaskExecute`. The main agent starts the initial ready tasks through that tool. When tracked `TaskExecute` runs complete, auto-cascade launches their dependents.

Creating a task doesn't bootstrap the cascade. Neither does manually marking one completed or spawning a plain `Agent`. The first ready tasks must go through `TaskExecute`.

The task descriptions matter more than the title "CTO". "Fix CSV importing" isn't a handoff. Something like this is:

```text
Allowed files: src/import/parse.ts, src/import/parse.test.ts.
Return an empty array for empty or whitespace-only input.
Preserve the existing result for valid CSV.
Acceptance: add regression tests; existing parser tests still pass.
```

Those filenames are an example; the planner must use the actual repository. It should split a larger change into several tasks, not send an engineer a whole project with a reassuring title.

Sure, I could prepare those handoffs myself. But that's the part I don't want to keep doing by hand. The engineer needs the decisions we made, not every option we decided against. Smaller tasks also mean smaller contexts for each engineer.

With `prompt_mode: append`, the engineer keeps system instructions and project conventions alongside its role instructions. With `inherit_context: false`, it doesn't inherit the entire parent chat. Cascaded tasks also receive their prerequisites' stored results. _Focused context, not no context._

This doesn't guarantee correct work, and it isn't an autonomous team. But it does keep agents progressing through work the CTO already described. I can come back from a coffee break to results or a blocker, instead of an invitation to say “continue”.

## Steer the Team

The remaining piece is routing. Add this to `~/.pi/agent/AGENTS.md`, preserving your existing instructions:

```markdown
These routing rules apply only to the main agent, not subagents.

Answer simple questions directly. Delegate implementation.
For non-trivial goals, ask cto to inspect the repo and plan tasks.
Route behavior and tests to engineer; presentation to designer.

When the plan returns, use TaskList and TaskGet to inspect it.
Use TaskExecute for ready pending tasks with agentType set.
Leave them pending: TaskExecute marks them in_progress.
Do not also spawn those tasks through Agent.
Let auto-cascade run dependents. Review results and test outcomes.
Triage failures and unassigned findings.
```

For follow-up findings, I also give the engineer limited access to the task board. This part is optional. Once tasks are installed, replace its `tools` line with:

```yaml
tools: read, write, edit, bash, grep, find, ls, ext:pi-tasks/TaskCreate, ext:pi-tasks/TaskList
```

And append this instruction to the engineer's body: "_File unrelated findings with TaskCreate as pending tasks without agentType; do not fix them._" It can record the finding, but the main agent decides what gets scheduled next.

Note that these instructions are workflow rules, not a security sandbox. File scopes still need review, and an agent with shell access can change files through the shell, even when they lack the `write` tool.

Restart Pi after the setup, open your repository, and give the CTO a goal:

```text
@cto Make CSV imports reject duplicate headers with a clear error.
Read the existing implementation and plan the smallest change.
```

The CTO returns a scoped plan. The main agent starts it. Dependent work follows without me typing “continue” after every task. For an explicit first test, tell the main agent: “Use cto to plan this change, then execute the ready tasks.”

Normal completion of an agent run through `TaskExecute` marks its task completed. That doesn't mean the tests passed. The extension doesn't interpret “I'm blocked” in the response or verify acceptance criteria, so I still need to inspect the results.

## A Team, Not Another Platform

What I wanted from tools like [Paperclip](https://paperclip.ing) and [Squad](https://squad.so) was the team and the cascade. Not everything those products offer. Just named roles, scoped work, and tasks that start when their prerequisites are done.

I can have that inside Pi, without another orchestration subscription. Model usage still costs money. I still review the changes.

But the question I bring to the conversation has changed. Not which model to use, or whether to tell it to continue. Just what I want the team to build.
