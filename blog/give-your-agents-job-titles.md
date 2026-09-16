---
description: Skip the model research. Give each agent a role with its own model, tools, and rules — then add a CTO to plan the work.
tags: tooling
date: 2026-09-16
draft: true
---

# Give Your Agents Job Titles

I used one model for everything.

Not because I thought that was the best way. I knew it wasn't. Some models are genuinely better at visual work. Some are quick and shallow, which is exactly what you want for searching. But keeping track of which model is good at what is a chore. The knowledge goes stale every few months, and I would rather not spend my attention on it.

So I spent everything on the same model instead, and quietly accepted that some of it would be mediocre.

What I wanted was the specialization without the research: to type `@engineer do this`, `@designer make that look right`, `@writer write about this`, and have the model choice already made somewhere I never have to look at it.

That's the whole reason I went down this road. It started with one role. It ended with a small company.

## A role is a file

In [pi](https://pi.dev), subagents aren't built in; you install a package for them. I used [`@tintinweb/pi-subagents`](https://github.com/tintinweb/pi-subagents), which discovers agent definitions from Markdown files with frontmatter:

```markdown
---
name: engineer
model: deepseek/deepseek-v4-flash
tools: read, write, edit, bash, grep, find, ls
thinking: medium
---
You implement exactly one assigned task, well, and stop.
```

That file _is_ the role. Three of those lines carry the whole trade-off, written down once:

- **`model`** — researched once, written down, never thought about again.
- **`tools`** — what this role is allowed to touch. This turns out to matter more than the model.
- **`thinking`** — how hard it should think. Medium for most implementation, high for planning and review, low for search-and-summarize.

Now `@engineer change the billing calculation` costs me nothing to think about. The model decision happens when I write the file, not when I have a problem to solve.

That's the whole trick, and it's worth sitting with for a second: **the role is where you put the knowledge you refuse to keep in your head.**

## The first version was just an engineer

I didn't start with a company. I started with `engineer`, because that's the work I have the most of.

It was immediately better. I stopped writing long prompts that began with context and ended with "be careful about X". The care was already in the role. And when I noticed a habit I didn't like — reaching for `bash` instead of the proper tools, or fixing things it wasn't asked to fix — I edited one file and every future agent inherited the correction.

But something was still wrong, and it took me a while to name it.

**I was still the planner.**

Every piece of work started with me describing it. I'd break a feature into parts in my head, write them out as prose, and hand them over one at a time. The engineer was doing the work, but I was still doing the thinking, and I was doing it badly — in the middle of a conversation, at whatever level of detail I happened to feel like.

I'd also become a message bus. The engineer would finish, I'd read the report, and I'd hand the next thing to the next agent. That's not delegation. That's me with a clipboard.

## So I added a CTO on top

The missing role wasn't another worker. It was someone to do the decomposing.

```markdown
---
name: cto
tools: read, grep, find, bash, ext:pi-tasks
disallowed_tools: TaskExecute, TaskStop
thinking: high
---
You turn goals into executable plans. You never implement.
```

Now I say `@cto add an adult PIN gate to the parent progress view`, and a few minutes later there's a task list: build the PIN prompt, wire it to the route, add the coin attribution, review the whole thing. With dependency edges, in the order they have to happen.

Two things make this work rather than being ceremony.

**It reads the code before it plans.** A plan written without looking at the repository is a guess. The CTO's instructions start with "inspect the repository first", and every task it writes has to name the files it will touch and state how the engineer will know it's done — because the engineer that receives it has never seen our conversation and can't ask follow-up questions.

**It cannot execute.** `TaskExecute` is deliberately removed from its tools. The CTO writes the plan; starting it is my call. That's the one human decision per plan, and it's the only gate I want.

## The task board becomes the conversation

The piece I didn't see coming: once work exists as tasks with dependencies, my agents can talk to each other without me.

A task board — I use [`@tintinweb/pi-tasks`](https://github.com/tintinweb/pi-tasks) — gives you a shared, file-backed list with `blocks`/`blockedBy` edges. Wiring it to the subagent package means:

- An engineer that trips over an unrelated bug **files a task** instead of burying it in a paragraph I'll skim past.
- When a task completes, **everything it was blocking becomes eligible** and starts on its own. Nobody says "continue".
- The CTO runs `TaskList` before planning, so it sees what its workers filed and sequences or deletes those rather than planning around them.

My role shrinks to two things: approve plans, and triage findings. Both are judgment calls I actually want to make. Everything I was doing between them was clerical.

## The value is in what a role cannot do

This is the part I'd tell anyone first, because it's the least obvious and it's where the quality came from.

Every role I added earned its place by being **denied** things:

- The **reviewer** has no write or edit tools. A reviewer that can edit will edit instead of reporting, and then nobody has reviewed anything.
- The **CTO** cannot execute. A planner that can start work isn't a gate, it's just a faster way to do the wrong thing.
- The **designer** is told explicitly not to touch application logic. The moment it can, it will, and you'll be debugging a layout change that altered a data shape.
- The **engineer** can create tasks but cannot start, stop, or reassign them. It can report what it found; it can't act on its own priorities.

We treat tool access as a capability question. It's also a _discipline_ question. A role is defined as much by its prohibitions as by its powers.

## Every problem turned out to be a prompt problem

I spent a week reaching for knobs, and the knobs were almost never the answer.

An engineer ground through a task for 86 turns and hit its turn ceiling. My instinct was to raise the ceiling. The actual problem was that the task said "and all three modules" — that's three tasks wearing one coat. The fix went in the CTO's instructions: if you can't state a task's file scope in one line, it's more than one task.

An agent kept asking for a reviewer to tell it whether its work was done. That's not a missing capability; that's a missing definition of done. It already had the acceptance criteria and a test suite. Now it has a written rule that it's done when the criteria are met and a test exists that fails without the change.

The turn counter in the widget is the most useful thing on screen, and it's a _symptom_. An agent at 86 of 100 turns doesn't mean the limit is too low. It means the plan was wrong, and the plan is a prompt.

## Start with one role

If any of this is useful, the order matters: **one role, then the plan.**

Write `engineer`. Give it the model you'd have picked anyway, the tools it obviously needs, and the three habits you keep correcting. Use it until it's boring.

Then notice how much of your time goes into describing what to do next. That's your signal that you need a planner, not another worker.

I built the whole company because I wanted `@engineer` to work. Turns out the roles were the easy part. The hard part was admitting that most of what I was doing wasn't work — it was routing.

---

_It's the same lesson as [the perfect stack is two](/articles/the-perfect-stack-is-two/): a few small tools, each genuinely good at one thing, beat one thing that claims to do everything. It just turns out that's true of the things doing the work, too._
