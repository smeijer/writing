---
description: I stopped switching models mid-conversation and gave Pi named roles, scoped tasks, and a way to keep working.
date: 2026-09-16
tags: tooling
draft: true
---

# Give Pi a Team

I kept using a single model in [Pi](https://pi.dev), even tho I knew it wasn't the right choice for everything. Not because switching models is hard. Because deciding when to switch is another thing to do while I'm trying to get something built.

Which model was good at planning again? Which one should handle the UI? Has that changed since the last release? And am I really going to interrupt this conversation to change models now?

Usually, no. I'd keep going with whatever was already selected.

The thing I wanted wasn't a better model picker. I wanted to ask an engineer to implement something, or a designer to work on the interface, without remembering which model I'd assigned to each job.

## 1. Roles: Give the Models a Job

That's where [pi-subagents](https://github.com/tintinweb/pi-subagents) comes in. It adds agents with their own instructions, tools, and model selection. Assuming you already have Pi and an authenticated model, install it with:

```shell
pi install npm:@tintinweb/pi-subagents
mkdir -p ~/.pi/agent/agents
```

Use Pi 0.84.0 or newer for the version described here. Restart Pi after installing.

An agent is a Markdown file. Here's a small `~/.pi/agent/agents/engineer.md`:

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

Replace the model with a `provider/modelId` you have authenticated in Pi. It's a choice you make for the role, not a recommendation here. Check `/agents` → Agent types to confirm which model it resolves to.

Now I can type:

```text
@engineer Add tests for the CSV parser's empty-input behavior.
```

For a designer, create `designer.md` using the same format. Change `name` to `designer`, describe its presentation work, and replace the body with instructions to own layout, interaction, and component interfaces. Choose its model separately.

I talk to `@engineer` or `@designer`. The assignments live in files, where I can change them without changing how I ask for work.

I still choose the model. I just choose it per role, instead of per interruption.

## 2. Tasks: Stop Being the Continue Button

Roles solved the model-switching problem. They didn't solve the next problem: I still had to tell the agents to continue.

One task finished. There was more work. Back to the prompt.

[pi-tasks](https://github.com/tintinweb/pi-tasks) adds the missing task list and dependencies:

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

Restart Pi after saving the config.

Both settings matter. Tasks default to session storage, with cascade disabled. Project storage gives the CTO and main agent the same board at `.pi/tasks/tasks.json`. Otherwise, the planner can create tasks the main agent cannot see.

These are global defaults. `/tasks` → Settings can override them for a project, saving to `.pi/tasks-config.json`.

The wiring is straightforward: `TaskCreate` assigns an `agentType`, `TaskUpdate` connects prerequisites through `addBlockedBy`, and `TaskExecute` starts ready tasks. Auto-cascade starts dependent tasks when their prerequisites have completed.

It's not a background company that keeps inventing work. It's a dependency graph that keeps moving through work you've already described.

## 3. Planning: Give the Engineer Less to Read

I don't want to write every task myself either. That's the CTO's job. It inspects the repo, breaks the goal into tasks, and gives each one exact files and the context needed to do the work. The engineer shouldn't need our whole conversation to implement one change.

Create `~/.pi/agent/agents/cto.md`:

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
its AGENTS.md/CLAUDE.md, and relevant build and source files first.
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

The CTO plans; it cannot call `TaskExecute`. The main agent starts the ready tasks through that tool, and auto-cascade handles the dependents when tracked execution completes. Creating tasks, manually marking one completed, or spawning a plain `Agent` doesn't start the cascade.

The task descriptions matter more than the title “CTO”. “Fix CSV importing” isn't a handoff. Something like this is:

```text
Allowed files: src/import/parse.ts, test/import/parse.test.ts.
Return an empty array for empty or whitespace-only input.
Preserve the existing result for valid CSV.
Acceptance: add regression tests; existing parser tests still pass.
```

Those filenames are an example; the planner must use the actual repository. It should split a larger change into several tasks, not send an engineer a whole project with a reassuring title.

Sure, I could prepare those handoffs myself. But that's the part I don't want to keep doing by hand. The engineer needs the decisions we made, not every option we decided against.

With `prompt_mode: append`, the engineer keeps system instructions and project conventions alongside its role instructions. With `inherit_context: false`, it doesn't inherit the entire parent chat. Cascaded tasks also receive their prerequisites' stored results. _Focused context, not no context._

## 4. Routing: Keep the Main Conversation Out of Implementation

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

Using `ext:` makes extension access an explicit allowlist, so keep any other extension tools your engineer needs on that line.

Append this instruction to the engineer's body: “File unrelated findings with TaskCreate as pending tasks without agentType; do not fix them.” It can record the finding, but the main agent decides what gets scheduled next.

These instructions are workflow rules, not a security sandbox. File scopes still need review, and an agent with shell access can change files through the shell.

Restart Pi after the setup, open your repository, and give the CTO a goal:

```text
@cto Make CSV imports reject duplicate headers with a clear error.
Read the existing implementation and plan the smallest change.
```

The CTO returns a scoped plan. The main agent starts it. Dependent work follows without me typing “continue” after every task. For an explicit first test, tell the main agent: “Use cto to plan this change, then execute the ready tasks.”

Normal completion of an agent run through `TaskExecute` marks its task completed. That doesn't mean the tests passed. The extension doesn't interpret “I'm blocked” in the response or verify acceptance criteria, so I still need to inspect the results.

## A Team: Not Another Platform

What I wanted from tools like [Paperclip](https://paperclip.ing) and [Squad](https://squad.so) was the team and the cascade. Not everything those products offer. Just named roles, scoped work, and tasks that start when their prerequisites are done.

I can have that inside Pi, without another orchestration subscription. Model usage still costs money. I still review the changes.

But the question I bring to the conversation has changed. Not which model to use, or whether to tell it to continue. Just what I want the team to build.
