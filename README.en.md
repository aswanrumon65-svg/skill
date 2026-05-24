# Claude Skills — Handy Daily Skills · 日常好用 Skill 合集

🌐 [中文](./README.md) · **English**

> Let Claude automatically route the things it's not good at to the most suitable external AI / platform.

Claude is a generalist. But for some tasks, specialists do better:

| Scenario | Who does it better |
|---|---|
| Real-time X (Twitter) discussion, sentiment, trends | **Grok** (xAI's own product, exclusive X firehose access) |
| Channeling a domain-expert WeChat account's judgment style | **wechat-persona** — distill the account into a `PERSONA.md` you can summon as a private advisor anytime |
| ……more scenarios to come | …… |

This repo collects skills that "let Claude outsource to the most suitable AI at the right moment." Every skill follows the same paradigm:
**Claude detects intent → calls external brain → fetches results → translates for you.**

---

## Released Skills

### 🐦 ask-grok (first in the series)

**What it does**

When you want to know **what's happening on X (Twitter) right now** — hot topics, breaking-news reactions, sentiment shifts, discussion around a specific tweet or account — Claude drives your already-logged-in Chrome to grok.com, asks Grok, and summarizes the answer back to you.

**Why it has to be Grok**

Other models (Claude included) only see X content after search engines have scraped it, with robots.txt + login walls + JS rendering blocking most of it. Grok is xAI's own product, **directly plugged into X's firehose**, reading the source posts. Among mainstream LLMs this is currently exclusive.

**Trigger scenarios**

- "What's trending on X about AI today?"
- "How is Twitter reacting to the GPT-5 launch?"
- "What is the crypto community on X discussing lately?"
- "X 上今天在聊什么 AI 新闻？"
- Explicit phrases: "ask Grok", "问问 Grok", "用 Grok 搜", "在 Grok 里查", "Grok 怎么说"

**How it works**

Through the [Claude for Chrome](https://claude.ai/chrome) extension, Claude uses your already-logged-in browser to:

1. Open grok.com
2. Type a rewritten question that nudges Grok to search X
3. Wait for the answer (typically 8–15 s, complex queries 30 s+)
4. Extract answer + Grok conversation link
5. Return **TL;DR + key bullets + a link to dive into the full conversation**

**Prerequisites**

- [Claude for Chrome](https://claude.ai/chrome) extension installed and enabled
- Logged into grok.com in Chrome (free account is fine)
- `mcp__Claude_in_Chrome__*` tool family available in the Claude Code session

Full workflow, gotchas, tool-call order: [`ask-grok/SKILL.md`](./ask-grok/SKILL.md) (in Chinese)

---

### 📜 wechat-persona (WeChat account distillation)

**What it does**

Batch-download recent articles from a WeChat Official Account you follow → clean them into Markdown → let Claude read them and distill a `PERSONA.md`: the author's core topics, reasoning framework, language style, values, and blind spots, all laid out. Afterwards you can say "**view this K-line / symptom / news through the lens of <author>**" and Claude will reason in that persona.

**Why this exists**

Those "trusted expert brains" living inside WeChat accounts — finance analysts, doctors, industry observers — normally you can only wait for them to post. This skill **persists** "a brain I trust" into a reusable private advisor.

**Trigger scenarios**

- "Batch download the last six months of 'XX' WeChat account"
- "Export the last three months of <account> for me"
- "Distill this WeChat account into a persona I can consult later"
- "Analyze this through <author>'s judgment style"
- 显式说："公众号蒸馏"、"公众号导出"、"用 wechat-article-exporter"

**How it works**

Uses the open-source [wechat-article-exporter](https://github.com/wechat-article/wechat-article-exporter) (hosted at `down.mptext.top`) to fetch articles, with Claude in Chrome driving the browser end-to-end: add target account → set time range → sync + fetch bodies → export Markdown to desktop → clean → read a sample and generate `PERSONA.md`.

**Prerequisites**

- [Claude for Chrome](https://claude.ai/chrome) extension installed
- **You personally own a WeChat Official Account** (a free personal subscription account is fine — sign up at mp.weixin.qq.com, 1 minute) — the tool requires scanning a QR code to log into your own account
- PowerShell on Windows for the cleanup scripts (Bash equivalents on macOS/Linux)

Full workflow, gotchas, tool-call order: [`wechat-persona/SKILL.md`](./wechat-persona/SKILL.md) (in Chinese)

---

## Installation

Each skill ships in two shapes:

- **`<skill-name>.skill`** — packaged file (zip format), good for distribution
- **`<skill-name>/`** — unpacked directory, browseable on GitHub

### Option 1: drop the unpacked directory into Claude's config

```
~/.claude/skills/<skill-name>/SKILL.md
```

Claude Code scans and loads on startup.

### Option 2: use the `.skill` package

Drag `<skill-name>.skill` into Claude Code's skill install entry (exact UX depends on your Claude Code version), or unpack it and follow Option 1.

### Verify

In Claude Code, say one of the trigger phrases from the skill description. For ask-grok try:

> "What's trending on X today?"

Claude should respond by entering the skill (typically calling something like `mcp__Claude_in_Chrome__list_connected_browsers`).

---

## Roadmap

| Status | Skill | Purpose |
|---|---|---|
| ✅ Released | **ask-grok** | Query X (Twitter) live info via Grok |
| ✅ Released | **wechat-persona** | Distill a WeChat Official Account into a callable persona advisor |
| 🔜 Planned | … | More "external brain" skills coming gradually |

Got an external AI / platform you want to wire up? Open an issue.

---

## Contribute

1. Fork this repo
2. Create a new skill directory, containing at least `SKILL.md` (follow the [Claude Skill spec](https://docs.claude.com/en/docs/build-with-claude/skills) and the style of [`ask-grok/SKILL.md`](./ask-grok/SKILL.md))
3. Also ship a `<skill-name>.skill` zip at the repo root for easy distribution
4. Add a row to the Roadmap table in the README
5. Open a PR

Each skill's `description` field should ideally include **both English and Chinese trigger phrases**, so Claude recognizes intent in either language.

---

## License

TBD
