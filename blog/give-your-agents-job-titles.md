---
description: Skip the model research. Give each agent a role with its own model, tools, and rules — then add a CTO to plan the work.
tags: tooling
date: 2026-09-16
draft: true
---

# Give Your Agents Job Titles

I used one model for everything.

Not because I thought that was the best way. I knew it wasn't. Some models are genuinely better at visual work. Some are quick and shallow, which is exactly what you want when you're searching a codebase. But keeping track of which model is good at what is a chore, the knowledge goes stale every few months, and I'd rather not spend my attention on it. So I ran everything on the same model and quietly accepted that some of it would come out mediocre.

What I wanted was the specialization without the research. To type `@engineer do this`, or `@designer make that look right`, and have the model decision already made somewhere I never have to look at it.

It took me longer to name the second thing I wanted, and it turned out to be the more important one. I wanted to stop typing "continue".

## It's the same text you already write

If you use a coding agent, you already customize it. You write a system prompt, or a project instruction file, or a skill, and you watch the agent behave differently. That's the entire interaction model, and it works remarkably well.

Look at what you're doing though. "You are a careful reviewer, never propose a change you can't test" is a _job description_. You've been writing roles this whole time, you just don't call them that, and they don't carry a model or a set of permissions with them.

A role is that same text, with a name and a few extra lines attached. It's not a new concept so much as a promotion of one you're already using. After all, it's all just text.

There are products that sell this pre-assembled. [paperclip.ing](https://paperclip.ing) and [squad.so](https://squad.so) will hand you a team of agents with a CEO and a CTO for around $99 a month. I'm not going to tell you they're overpriced, because they bundle a UI, hosting and integrations I had to solve myself. But the part I actually cared about was the configuration, and the configuration is text. So I wrote my own.

## A role is a file: model, tools, thinking

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

Now `@engineer change the billing calculation` costs me nothing to think about. The model decision happens when I write the file, not when I have a problem to solve. The role is where you put the knowledge you refuse to keep in your head.

## What I actually wanted was to stop saying "continue"

I started with one role. An engineer, because that's the work I have the most of. It was immediately better. I stopped writing long prompts that began with context and ended with "be careful about X", because the care was already in the role. And when I noticed a habit I didn't like, I edited one file and every future agent inherited the correction.

It didn't solve anything else, though. I was still the planner and, worse, I was the scheduler.

Every piece of work started with me describing it. Every finished piece ended with me reading a report and deciding what came next. The engineer was doing the work, but the queue only moved when I pushed it. Prompt, wait, read. Prompt, wait, read. The agent never blocked in the middle of a task; it blocked at the boundary of every task, waiting for me to push.

That's the part that wears you down. Not the model choice, not the prompt writing — the _advancing_. I was the thing that needed a "continue", and I was saying it all day.

## So I added a CTO: decompose, then write it down

The missing role wasn't another worker. It was the two jobs I was doing by hand: breaking a goal into pieces, and writing those pieces down somewhere that outlives the conversation.

```markdown
---
name: cto
tools: read, grep, find, bash, ext:pi-tasks
disallowed_tools: TaskExecute, TaskStop
thinking: high
---
You turn goals into executable plans. You never implement.
```

Now I say `@cto add an adult PIN gate to the parent progress view`, and a few minutes later there's a task list: build the PIN prompt, wire it to the route, add the attribution, review the whole thing. With dependency edges, in the order they have to happen.

Two things make that work rather than being ceremony.

**It reads the code before it plans.** A plan written without looking at the repository is a guess. The CTO's instructions start with "inspect the repository first", and every task it writes has to name the files it will touch and state how the engineer will know it's done — because the engineer receiving it has never seen our conversation and cannot ask follow-up questions.

**It cannot execute.** `TaskExecute` is deliberately missing from its tools. The CTO writes the plan; starting it is my call. That's the one human decision per plan, and it's the only gate I want to keep.

## The board becomes the conversation: nothing waits for me

The piece I didn't see coming: once work exists as tasks with dependencies, the agents can talk to each other without me in the middle.

A task board — I use [`@tintinweb/pi-tasks`](https://github.com/tintinweb/pi-tasks) — gives you a shared, file-backed list with `blocks`/`blockedBy` edges. Wiring it to the subagent package means three things:

- An engineer that trips over an unrelated bug **files a task** instead of burying it in a paragraph I'll skim past.
- When a task completes, **everything it was blocking becomes eligible** and starts on its own. Nobody says "continue".
- The CTO runs `TaskList` before planning, so it sees what its workers filed and sequences or deletes those instead of planning around them.

My role shrinks to two things: approve plans, and triage findings. Both are judgment calls I actually want to make. Everything I was doing between them was clerical work I'd mistaken for involvement.

## The value is in what a role cannot do

This is the part I'd tell anyone first, because it's the least obvious and it's where the quality came from.

Every role I added earned its place by being _denied_ things:

- The **reviewer** has no write or edit tools. A reviewer that can edit will edit instead of reporting, and then nobody has reviewed anything.
- The **CTO** cannot execute. A planner that can start work isn't a gate, it's just a faster way to do the wrong thing.
- The **designer** is told explicitly not to touch application logic. The moment it can, it will, and you'll be debugging a layout change that altered a data shape.
- The **engineer** can create tasks but cannot start, stop or reassign them. It can report what it found; it cannot act on its own priorities.

We treat tool access as a capability question. It's also a _discipline_ question. A role is defined as much by its prohibitions as by its powers.

## Every problem turned out to be a prompt problem

I spent a week reaching for knobs, and the knobs were almost never the answer.

An engineer ground through a task for 86 turns and hit its turn ceiling. My instinct was to raise the ceiling. The actual problem was that the task said "and all three modules", which is three tasks wearing one coat. The fix went into the CTO's instructions: if you can't state a task's file scope in one line, it's more than one task.

The same engineer kept asking for a reviewer to tell it whether its work was done. That's not a missing capability, that's a missing definition of done. It already had the acceptance criteria and a test suite. Now it has a written rule: it's done when the criteria are met and a test exists that fails without the change.

The turn counter on screen is the most useful number in the whole setup, and it's a _symptom_. An agent at 86 of 100 turns doesn't mean the limit is too low. It means the plan was wrong, and the plan is a prompt.

## Final words

If any of this is useful, the order matters: one role, then the plan.

Write `engineer`. Give it the model you'd have picked anyway, the tools it obviously needs, and the three habits you keep correcting. Use it until it's boring. Then pay attention to how much of your time goes into describing what to do next, because that's the signal that you need a planner, not another worker.

I didn't set out to build a team. I set out to stop thinking about models. Turns out the model problem was the easy one, and the roles were the easy part after that. The hard part was admitting that what I was doing between tasks — the reading, the deciding, the pushing — wasn't work. It was scheduling. And I was the bottleneck.

_It's the same lesson as [the perfect stack is two](/articles/the-perfect-stack-is-two/): a few small tools, each genuinely good at one thing, beat one thing that claims to do everything. It just turns out that's true of the things doing the work, too._
