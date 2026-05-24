# Claude Skills — 外接大脑系列 · External Brain for Claude

> 让 Claude 在自己不擅长的事情上，自动调用最合适的外部 AI / 平台。

Claude 是个通才。但有些事情，专才更靠谱：

| 场景 | 谁更擅长 |
|---|---|
| X (Twitter) 实时讨论、情绪、热点 | **Grok**（xAI 自家产品，独家 X firehose 访问） |
| 长期信赖的领域专家公众号的判断风格 | 通过 **wechat-persona** 把公众号蒸馏成 `PERSONA.md`，作为私人顾问随时调用 |
| ……更多场景待补 | …… |

这个仓库收集"让 Claude 在合适的时机外包给最合适的 AI"的 skill。每个 skill 走的都是同一套范式：
**Claude 检测意图 → 调用外脑 → 拿回结果 → 翻译给你看。**

---

## 已发布 Skills

### 📜 wechat-persona（公众号蒸馏）

**做什么**

把一个你长期关注的微信公众号的近期文章批量抓下来 → 清洗成干净 Markdown → 让 Claude 通读后蒸馏出一份 `PERSONA.md`：作者的核心议题、思维框架、语言风格、价值观、盲区都列清楚。之后你随时可以喊"**以 XX 公众号的视角看这条 K 线 / 这个症状 / 这个新闻**"，Claude 会套用这个 persona 给你辅助判断。

**为什么需要这个**

公众号里那些你信赖的"专家脑子"——财经号、医学号、行业观察号——平时只能等他们更新。这个 skill 把"一个我信赖的脑子"**持久化**下来变成你的私人顾问。

**触发场景**

- "把'挥手看天空'这个公众号近半年的文章批量下下来"
- "导出 XX 公众号最近三个月的内容"
- "把这个公众号蒸馏成一个角色，我以后用来辅助判断"
- "按 XX 作者的判断风格分析一下这个"
- 显式说："公众号蒸馏"、"公众号导出"、"用 wechat-article-exporter"

**工作原理**

借现成开源工具 [wechat-article-exporter](https://github.com/wechat-article/wechat-article-exporter)（在线站 `down.mptext.top`）抓文章，Claude in Chrome 全程驱动浏览器：添加目标号 → 设时间范围 → 同步 + 抓正文 → 导出 Markdown 到桌面 → 清洗 → 通读样本生成 PERSONA.md。

**前置条件**

- 已安装 [Claude for Chrome](https://claude.ai/chrome) 扩展
- **你本人有一个微信公众号**（订阅号即可，免费申请，1 分钟）—— 工具需要扫码登录你自己的号
- Windows 本机能跑 PowerShell（清洗脚本）；macOS/Linux 用 Bash 等价命令

详细流程、踩坑记录、tool 调用顺序：[`wechat-persona/SKILL.md`](./wechat-persona/SKILL.md)

---

### 🐦 ask-grok（系列第一个）

**做什么**

当你想知道 **X (Twitter) 上现在在发生什么**——热点、突发新闻反应、情绪走向、某条推 / 某个账号的讨论度——Claude 会自动驱动你本机已登录的 Chrome，去 grok.com 向 Grok 提问，再把答案总结给你。

**为什么必须是 Grok**

其他模型（包括 Claude 自己）拿到的是搜索引擎事后抓取的 X 内容，被 robots.txt + 登录墙 + JS 渲染挡掉一大半。Grok 是 xAI 自家产品，**直接接入 X 的 firehose**，能读源帖子。这点在主流 LLM 里目前是独家。

**触发场景**

- "X 上今天在聊什么 AI 新闻？"
- "GPT-5 发布后 Twitter 的反应？"
- "加密圈最近 X 上讨论什么"
- "What's trending on X about \<topic\>?"
- 显式说："问问 Grok"、"用 Grok 搜"、"在 Grok 里查"、"Grok 怎么说"

**工作原理**

通过 [Claude for Chrome](https://claude.ai/chrome) 扩展，Claude 用你本机已登录的浏览器：

1. 打开 grok.com
2. 把改写过、引导 Grok 搜 X 的问题输入到对话框
3. 等待回答（典型 8–15 秒，复杂查询 30s+）
4. 提取答案 + Grok 对话链接
5. 返回给你 **TL;DR + 关键要点 + 完整对话深挖入口**

**前置条件**

- 已安装 [Claude for Chrome](https://claude.ai/chrome) 扩展，并在 Chrome 里启用
- Chrome 里已登录 grok.com（免费账号即可）
- Claude Code 会话里能加载 `mcp__Claude_in_Chrome__*` 系列工具

详细流程、踩坑记录、tool 调用顺序：[`ask-grok/SKILL.md`](./ask-grok/SKILL.md)

---

## 安装

每个 skill 同时提供两种形态：

- **`<skill-name>.skill`** — 打包文件（zip 格式），适合分发
- **`<skill-name>/`** — 解压目录，便于在 GitHub 上直接浏览源码

### 方式 1：解压目录放进 Claude 配置目录

```
~/.claude/skills/<skill-name>/SKILL.md
```

Claude Code 启动时会扫描并加载。

### 方式 2：用 `.skill` 包

把 `<skill-name>.skill` 拖入 Claude Code 的 skill 安装入口（具体方式以你版本的 Claude Code 文档为准），或解压后按方式 1 放置。

### 验证

在 Claude Code 里随便说一句 skill 描述里的触发短语，例如 ask-grok 试：

> "X 上今天在聊什么？"

Claude 应当响应一个进入该 skill 的回复（通常会调用 `mcp__Claude_in_Chrome__list_connected_browsers` 这类工具）。

---

## Roadmap

| 状态 | Skill | 用途 |
|---|---|---|
| ✅ Released | **ask-grok** | 通过 Grok 查 X (Twitter) 实时信息 |
| ✅ Released | **wechat-persona** | 把微信公众号蒸馏成可调用的 persona 顾问 |
| 🔜 规划中 | … | 更多"外接大脑" skill 陆续添加 |

有想接入的外部 AI / 平台？欢迎在 issue 里提。

---

## 贡献 / Contribute

1. Fork 这个仓库
2. 新 skill 起一个目录，至少包含 `SKILL.md`（参考 [Claude Skill 规范](https://docs.claude.com/en/docs/build-with-claude/skills) 与 [`ask-grok/SKILL.md`](./ask-grok/SKILL.md) 风格）
3. 同时打一个 `<skill-name>.skill` zip 放仓库根目录方便分发
4. README 的 Roadmap 表格里加一行
5. 提 PR

每个 skill 的 description 字段最好包含**英文 + 中文**触发词，确保 Claude 在两种语言的对话里都能正确识别。

---

## License

TBD
