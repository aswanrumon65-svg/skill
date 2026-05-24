---
name: ask-grok
description: Ask Grok (xAI's chatbot at grok.com) a question by driving the user's Chrome browser through the Claude in Chrome extension, then summarize the answer with a link to the Grok conversation. Use whenever the user wants real-time information from X (Twitter) — trending discussions, breaking news, sentiment on a topic, reactions to a launch, "what's happening on X right now" — because Grok has live X firehose access other models lack. Also use when the user explicitly says "ask Grok", "用 Grok 搜", "问问 Grok", "在 Grok 里查", "Grok 怎么说". Trigger even without naming Grok if the query is clearly about live X activity (e.g. "今天 X 上在聊什么", "what's trending on Twitter about Y", "搜下 X 上对 Z 的反应"). Do NOT use for static facts, code generation, or anything the model already knows — those don't need a browser.
---

# ask-grok

This skill lets you ask Grok a question through the user's own logged-in browser session and return a summary of the answer.

## Why this exists

Grok has a unique capability among major LLMs: it can search X (Twitter) in real time. When the user wants to know what's happening on X right now — trends, reactions, sentiment, breaking news threads — no other tool reaches that data as cleanly. Web search hits article aggregations after the fact; Grok hits the source posts as they appear.

The user has the Claude for Chrome extension installed and is logged into grok.com. You drive their browser, ask Grok the question, wait for the response, and report back with a summary plus the conversation link so they can dig deeper if they want.

## Prerequisites

Before doing anything, make sure the Claude in Chrome browser tools are available in this session. Specifically you need: `mcp__Claude_in_Chrome__list_connected_browsers`, `select_browser`, `tabs_context_mcp`, `navigate`, `find`, `computer`, `get_page_text`, and `browser_batch`. Load them via `ToolSearch` with a query like `select:mcp__Claude_in_Chrome__list_connected_browsers,mcp__Claude_in_Chrome__navigate,mcp__Claude_in_Chrome__computer,mcp__Claude_in_Chrome__find,mcp__Claude_in_Chrome__get_page_text,mcp__Claude_in_Chrome__browser_batch,mcp__Claude_in_Chrome__tabs_context_mcp,mcp__Claude_in_Chrome__select_browser` if they aren't loaded already.

If those tools don't exist at all in the session (different agent surface), stop and tell the user this skill requires the Claude for Chrome extension; suggest they fall back to plain `WebSearch` instead.

## Workflow

### Step 1 — Make sure the browser is connected

Call `list_connected_browsers`. If no browser is returned, tell the user:

> Claude for Chrome doesn't seem connected right now. Please open Chrome, click the Claude extension icon, and make sure it's signed in. Then say "try again".

When at least one browser is returned, call `select_browser` with the `deviceId` of the first local browser (the one with `isLocal: true`). If there are multiple, ask the user which to use.

### Step 2 — Get a fresh tab and open Grok

Call `tabs_context_mcp` with `createIfEmpty: true` to get a working tab group. Take the `tabId` of the new (or only) tab from the result. Then `navigate` that tab to `https://grok.com`.

It's fine to put the `tabs_context_mcp` call by itself first (the tab ID isn't known until it returns), and then batch the rest. Once you have the tab ID, prefer `browser_batch` for any sequence of two or more browser actions in a row — round-trip latency adds up otherwise.

### Step 3 — Type the question and submit

Grok's input box is a `contenteditable` element, not a standard `<input>`. **Do not use `form_input` on it** — it will fail with "Element type UL is not a supported form input" or similar.

**Recommended approach: take a screenshot first, then click on coordinates.** Empirically, `find` returns a `ref` for the textbox, but clicking that `ref` often lands on the surrounding container rather than focusing the editable area — `type` afterward then enters nothing visible. Going by coordinates from a screenshot is more reliable.

Concretely:

1. Take a `screenshot` of the page. The input bar is the rounded pill near the vertical center of the viewport on the Grok homepage (or pinned to the bottom in an active conversation). Identify the visible cursor or the placeholder text ("向 Grok 提任何问题" / "Ask Grok anything") and pick a coordinate inside the text area — somewhere around the middle horizontally, on the line of the placeholder.
2. `computer` `left_click` at that coordinate to focus the input.
3. `computer` `type` the user's question. Phrase it in the user's language. For X realtime queries, ask in a way that nudges Grok toward searching X — e.g. "X 上今天关于 [topic] 的热门讨论是什么？请列出主要推文" or "What's trending on X about [topic] right now? Summarize the top posts."
4. Take another screenshot to confirm the text actually landed in the input bar before submitting. If it didn't (input still shows placeholder, or text went somewhere else like the URL bar), reposition and try again — don't submit blind.
5. Submit by clicking the dark submit button (an upward arrow) at the right end of the input bar. From a typical 1568-wide viewport, this sits around x≈1270, y≈309 on the homepage; in an active conversation the input bar moves to the bottom (y≈695) but the arrow remains at the right end. Read coordinates off your screenshot rather than hardcoding them. Pressing the Return key sometimes also works but is less reliable across layouts.

(`find` is still worth calling if you want extra signal — but treat its `ref` as a hint, not a guaranteed clickable target. Coordinates from a fresh screenshot are the source of truth.)

**Phrasing tip:** if the user wants real-time X data, **always include the word "X" or the phrase "搜 X" / "search X for"** in the query you type, even if they didn't. Grok decides whether to invoke its live X search based on the wording; nudging it explicitly makes that more likely.

### Step 4 — Wait for the response

After submitting, Grok shows "思考了 Ns" / "Thought for Ns" while it works, then streams the response. Real-time X queries typically take 8–15 seconds end to end; pure-LLM answers can be faster.

Strategy: `wait` 8 seconds, then `get_page_text` and check whether a "20 sources" / "N sources" footer or a complete-looking answer is present. If the response still looks mid-stream (ends abruptly mid-sentence, no sources footer), wait another 5–8 seconds and re-read. Give up after about 45 seconds total and report what you have.

### Step 5 — Extract and summarize

`get_page_text` returns the conversation text along with the title and current URL. The URL after Grok responds looks like `https://grok.com/c/<conversation-id>?rid=<reply-id>` — this is a stable link to the conversation that the user can open later. Save it.

Now write your reply to the user with this shape:

> **TL;DR:** [2–4 sentence summary in the user's language of what Grok found.]
>
> **Key points:**
> - [bullet 1]
> - [bullet 2]
> - [bullet 3]
>
> **Sources:** Grok cited N posts. [Open the full conversation in Grok](<conversation URL>)

Keep the summary tight — the user can click through if they want the full thing. Don't reproduce Grok's response verbatim or quote large chunks; paraphrase. (If Grok's response includes named X users or specific quoted tweets, you can mention up to a few by handle in the bullets — that's the kind of detail that pays for itself.)

If Grok's response is itself a refusal or hedge ("I don't have access to real-time data right now", etc.), say so honestly rather than dressing it up.

## Common failure modes

**The input box won't accept typing.** This is the #1 failure mode and has two causes: (a) you tried `form_input` — it doesn't work on Grok's contenteditable UL; (b) you clicked a `ref` returned by `find` and the click landed on a container rather than the editable area. Solution in both cases: take a fresh screenshot, locate the visible cursor or the placeholder text "向 Grok 提任何问题", click at coordinates inside that text area, then type. Always screenshot once more *before* submitting to confirm the text actually landed — submitting blind wastes time when the input was empty.

**The page loaded but the user isn't logged in.** You'll see a "Sign in" prompt instead of the input bar. Stop and tell the user:
> Looks like you're not signed into Grok in this Chrome profile. Please sign in at grok.com and say "try again".

**Tab group disappeared between calls.** The MCP tab group can be closed by the user mid-task. If `tabs_context_mcp` (without `createIfEmpty`) returns no tabs, call it again with `createIfEmpty: true` to start fresh, and re-do the navigation.

**Response takes too long.** Some complex X queries take 30+ seconds. Don't abandon early — check the page text every 8 seconds for up to ~45s. If it's still streaming when you give up, tell the user that and offer to re-check by opening the saved conversation URL.

**User asked something that doesn't need Grok.** If on reflection the question is a static factual one that this skill was triggered on by accident, you can just answer it directly and skip the browser dance. The browser overhead is real — don't pay it when it doesn't buy anything.

## Quick reference: tool sequence

A clean run looks like this:

```
list_connected_browsers          → pick deviceId
select_browser                    → connect
tabs_context_mcp (createIfEmpty)  → get tabId
navigate (url=grok.com, tabId)
computer screenshot               → locate input bar coordinates
browser_batch:
  computer left_click (input-bar coords)
  computer type (query)
  computer screenshot              ← verify text landed
  computer left_click (submit-button coords, right end of input bar)
  computer wait 8s
  computer screenshot              ← see if response is in
get_page_text                     → extract response + URL
[optional: wait 5–8s and get_page_text again if mid-stream]
```

Two screenshots inside the batch are non-negotiable: one after `type` to confirm input landed, one after the wait to see response state. They're the cheapest defense against silently submitting an empty query or returning a half-streamed answer.

Then write the summary.
                                                                                                                                                                                                                                       