---
title: "memlite part 1: what is memlite?"
description: "Building an MCP-compatible long-term memory engine for AI agents"
date: 2026-06-01T11:30:00+01:00
draft: false
tags:
- ai
- claude
- dmons
- open-source
categories:
- Coding
---

Problem: every time I started a new Claude session, it didn't know me. Not in the "we haven't met" sense — in the "I told you about my coffee three days ago and you've forgotten" sense. There's a model gap there that no amount of context-window engineering closes. The fix isn't a bigger prompt. It's a place to *put* the things you'd want any agent to remember.

I tried a couple, like SQLite-Memory, but none of them worked the way I wanted. And the whole point of this AI-coding era is custom software, just the way you want it. So I built one. **memlite** is a long-term memory engine for AI agents — a shared, per-user fact store that any MCP-capable host (Claude, Pi, whatever else) can read from and write to. It's not a docs index. It's not RAG over your codebase. It's the boring relational stuff that makes an assistant feel like it actually knows you: facts, preferences, events, small bits of context.

---

## What it looks like

One static binary. This was important, and it's why memlite is written in Zig 0.16, with SQLite, [sqlite-vec](https://github.com/asg017/sqlite-vec), [llama.cpp](https://github.com/ggerganov/llama.cpp), and [md4c](https://github.com/mity/md4c) compiled directly in. On macOS, `otool -L` shows exactly one dependency — libSystem. There's nothing to install around it.

```sh
memlite init     # creates ~/.memlite/, pulls a 99MB embedding model
memlite serve    # speaks MCP over stdio
```

That second command is what your agent host connects to. The protocol is newline-delimited JSON-RPC — proper MCP, framing-correct, no LSP-style Content-Length headers. Logs go to stderr; the protocol channel on stdout stays clean.

---

## The data model is relationship-shaped

The thing I had to work out early was *what* memlite stores. The temptation is to make it a generic note-taker. I think that's a mistake — generic stores end up as junk drawers. memlite has opinions.

A memory has:

- A **slug**: an agent-supplied human-readable identity (`user-tea-preference`, `daughters-birthday`). This is how you update something later without guessing IDs.
- **Content**: short text or markdown. There's a single `format` column rather than two columns, two paths, two everything.
- **Tags**: an EAV side table — `kind: preference`, `source: claude`, `topic: coffee`. Multi-agent attribution is the headline use of `source`; you can ask "what did Pi remember about me?" and get an answer.
- **History**: soft-delete keeps a snapshot of every prior version, so the store has a memory of its own forgetting.

Markdown content gets chunked on ingest, so the search side can address paragraphs, not just whole memories.

---

## The bit I'm most happy about: a handshake that tells the agent what to do

MCP has an `initialize` step. Most servers fill in the protocol fields and stop. memlite's `initialize` response includes a multi-paragraph **instructions** string that ships compiled into the binary. It tells the connecting agent what memlite is for, when to add a memory (with the search-before-add pattern), the slug convention, the tag conventions, and the full read/list vocabulary.

This matters more than I expected. The behaviour of a memory tool isn't just its API — it's the shape of *how the agent decides to use it*. The handshake is the only place to put that.

---

## Where to go next

Part 2 is the search story — how a query becomes results when you've got an FTS5 index, a vector index, and no good reason to pick between them. Spoiler: you don't pick.

— *Code: [`github.com/daemonicai/memlite`](https://github.com/daemonicai/memlite)*
