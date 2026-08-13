---
title: The Perfect Stack Is Two
description: Stop chasing one framework to do it all. The perfect stack is two.
tags: thoughts
date: 2025-10-19
---

Someone asked me why I use different stacks for my landing page and for my app. He was confident a proper meta-framework must exist that can do both.

I get where he's coming from. It's a tempting idea; one framework that handles marketing pages, docs, and highly interactive apps/dashboards. But I think that's exactly where web development went off track. We're folding totally different problems into the same framework, and then wondering why everything feels so heavy.

It's like those famous all-in-one kitchen machines. They promise to blend, cook, chop, and bake. In practice, they do none of those things well. Better to have a few small tools, each perfect at what it does.

## Two problems, two stacks

A landing page and an app don't share the same constraints.

A landing page is static. Its job is to load instantly, rank well, and change rarely. The simplest thing that could possibly work is a pre-rendered HTML file, served from a CDN. Astro embraces that model beautifully. You write components, build once, and deploy pure files. It even lets you sprinkle interactivity when needed, without shipping React to the entire page.

An app, on the other hand, is dynamic. It's stateful, personalized, and often long-lived in memory. You don't want every click to trigger a server render. You want a responsive client that manages data locally, syncs via APIs, and updates instantly.

For that, a simple SPA with React, TanStack Router, and React Query is close to perfect. It gives you client-side routing, caching, and data fetching without hiding what's going on. No magical loader functions, no [blurred boundaries](/articles/isomorphic-development/) between server and browser. Just tools that do one thing well.

## The meta-framework trap

Frameworks like Next.js and Remix started as good ideas bridging server and client. But they've grown into all-purpose platforms that try to be both a static site generator and a full app framework. Although, I believe Remix still doesn't support static generation. You get the idea.

The result? Complexity disguised as convenience.

You now have code that runs half on the server, half in the browser, and [sometimes everywhere](/articles/isomorphic-development/). You're not sure where your function executes, or why hydration fails, or why you suddenly need to understand React Server Components just to render a button. We've created new abstractions to hide the old ones, and the only thing that's truly universal is the confusion.

Yes, it's nice to share a language across environments. But sharing code across them is where the trouble begins. There has to be a boundary. That's not a limitation, that's architecture.

## The perfect stack

So no, there's no such thing as _the_ perfect stack. But there are two that are each perfect in their own domain.

- **Astro** for anything mostly static.
- **Vite + React + TanStack** for anything truly dynamic.

Two stacks. Two mindsets. Clear boundaries, no magic, and way less stress.

Sometimes simplicity isn't about finding one framework that does everything. It's about letting go of that idea entirely.