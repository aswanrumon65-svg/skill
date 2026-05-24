---
name: wechat-persona
description: Distill a WeChat Official Account (微信公众号) into a reusable persona profile by batch-downloading its recent articles via wechat-article-exporter (down.mptext.top), cleaning the markdown, and synthesizing the author's voice/framework/blind-spots into a PERSONA.md you can later @-reference for "what would this author say about X?" reasoning. Use whenever the user wants to: archive a specific WeChat account's articles ("把某公众号的文章批量下载下来"、"导出微信公众号近X个月的文章"、"挖某个公众号的内容"), turn a domain-expert account into an advisory persona ("把这个公众号蒸馏成一个角色"、"按这个公众号作者的视角分析"、"模仿曲曲大女人的判断风格"), or mentions wechat-article-exporter / down.mptext.top by name. Trigger on Chinese phrases like "公众号蒸馏"、"公众号导出"、"公众号批量下载"、"把XX公众号变成顾问". Skip for: searching one-off articles (use sogou/weixin search instead), or accounts the user merely wants to read once.
---

# 公众号蒸馏 · wechat-persona

把一个微信公众号的近期文章批量抓下来 → 清洗成干净 Markdown → 蒸馏出一份 PERSONA.md，让 Claude 之后能"以这位作者的视角"帮用户做辅助判断。

## 存在原因

很多用户长期关注某一两个公众号（财经分析号、医学顾问号、行业观察号……），靠的是作者**稳定的思维框架和判断口味**。Claude 自己不读公众号，也无法实时抓微信内容。这个 skill 走的范式：

**借现成工具 wechat-article-exporter 把文章离线下来 → 让 Claude 通读 → 提炼成可复用的人格档案。**

之后用户可以随时调用："以曲曲大女人的判断风格看这条 K 线"、"按 XX 医生的逻辑评估这个症状要不要去医院"——本质是把"一个我信赖的脑子"持久化下来。

## 前置条件

- **Claude in Chrome 扩展**：会话需提供 `mcp__Claude_in_Chrome__*` 工具集（`list_connected_browsers`、`tabs_context_mcp`、`computer`、`find`、`read_console_messages`、`browser_batch` 等）。缺失则告知用户安装。
- **用户本人有一个公众号**：wechat-article-exporter 依赖"公众号后台编辑文章时插入历史文章"API，登录需要扫码选择**自己的**公众号（订阅号即可，个人免费申请）。**这是硬门槛**——用户没有自己的公众号就走不通这条路。
- **本地 Shell**：要能跑 PowerShell（Windows）或 Bash 文件操作（清洗 + 蒸馏阶段）。
- **可选**：用户已在浏览器登录 https://down.mptext.top（cookie 会跨标签复用）。

## 工作流

### 第 0 步——前置确认

先问用户三件事：
1. **公众号名字**（精确名称，不是模糊关键词）
2. **时间范围**（"近一个月 / 三个月 / 半年 / 一年"——工具内置选项，自定义会更慢）
3. **是否已经有自己的公众号能用来扫码登录** —— 没有就让用户先去 mp.weixin.qq.com 申请个人订阅号（实名认证，1 分钟，免费、不需要审核）

### 第 1 步——连接浏览器 + 打开工具站

```
list_connected_browsers → select_browser →
tabs_context_mcp (createIfEmpty: true) →
navigate https://down.mptext.top
```

读页面，确认右下角显示"登录信息过期时间还剩 X 天"。如果跳到登录页：让用户**亲自**扫二维码并选公众号（这步无法代为操作）。

### 第 2 步——添加目标公众号

进入"公众号管理"页（默认页）。点"添加"按钮（左上工具栏第一个），弹出搜索框 → 输入公众号名字 → 回车 → 候选列表里点击目标条目。

**踩坑**：用 `ref` 点击有时无效，回退到 `computer` `left_click` + 坐标。候选列表可能有同名小号 / 关联号，注意核对**微信号**（subtitle 处显示）确认是主号。

成功后页面顶部 toast "公众号添加成功"，表格里新增一行，记下"消息总数"（历史发布总数）。

### 第 3 步——配置时间范围

侧边栏点"设置"，往下滚到"其他 → 同步时间范围"。这是个原生 select，**listbox 必须用 `read_page` + `ref` 才能可靠选中选项**，坐标点击会卡渲染。

可选范围：最近24小时 / 一天 / 三天 / 七天 / 一个月 / 三个月 / **半年** / 一年 / 自定义时间。选完即时生效（无保存按钮），同行右侧"实际同步范围"会立刻刷新显示具体起止日期。

### 第 4 步——同步 + 抓正文

回"公众号管理"页，点目标行末尾的蓝色圆形操作按钮（单行同步）。按钮变绿、显示加载动画。

完成判断：按钮恢复蓝色 + 该行"已同步消息数"≈ 时间范围内文章数。

去"文章下载"页（侧边栏），从顶部下拉选目标公众号（item 文本类似"曲曲大女人 (120篇)"），表格出现文章列表。

**全选**：点表头复选框（坐标约 `(338, 168)`），底部状态栏显示"已选 N/N"。点右上"抓取"下拉 → 选"文章内容"（"阅读量"/"留言"需要 credential，跳过）。

抓正文是**长任务**：默认 3 秒/请求 × N 篇，预计 N × 10 秒。期间 `screenshot` 会因 `document_idle` 等不到而失败，**改用 `find` 工具查"抓取中 X/N"按钮文本**确认进度。可用 `Bash run_in_background sleep` 拉长间隔，避免轮询浪费。

### 第 5 步——导出 Markdown（关键坑）

点右上"导出"下拉 → 选 "Markdown"。

⚠️ **核心坑**：导出 Markdown 走 **File System Access API**（`window.showDirectoryPicker()`），浏览器会弹一个 **Windows/macOS 原生**"选择文件夹"对话框，**不是网页里的弹窗**——Claude in Chrome **看不到也操作不了**。

必须：
1. 在 Skill 触发后**用 Bash/PowerShell 先在用户桌面建好目标文件夹**（如 `C:\Users\<user>\Desktop\<公众号名>-md`）
2. 明确告诉用户："**现在留意你电脑的任务栏，会弹一个原生选择文件夹对话框**，导航到桌面那个文件夹，点'选择文件夹'，再点'允许'。"
3. 等用户口头确认"选好了"再继续监控进度

如果对话框被自动 abort（120 秒超时），console 会输出 `AbortError: showDirectoryPicker`。需要再点一次"导出 → Markdown"重新触发。

写入完成后导出按钮恢复"导出"原文本，目标文件夹里有 N 个 `.md` 文件，每个 ~10KB。**图片用远程 URL 保留（`mmbiz.qpic.cn`），不下载本地** —— 这是工具的默认行为，需要联网才能看图。

### 第 6 步——清洗

每个 .md 文件有两处垃圾：

**A. 文件开头的 CSS 样式（第 1 行）** —— 微信公众号 `<style>` 内容被当文本输出。
**B. 文件末尾的 SVG 按钮图标行** —— "阅读 / 赞 / 分享 / 推荐 / 留言"按钮的内联 `data:image/svg+xml`，一长串干扰阅读。

PowerShell 一把梭（Windows，用户桌面 `曲曲大女人-md` 为例）：

```powershell
$target = "C:\Users\<USER>\Desktop\曲曲大女人-md"

# 清 A：删第 1 行 CSS（仅在确认是 CSS 时删）
Get-ChildItem -Path $target -Filter "*.md" | ForEach-Object {
    $lines = Get-Content $_.FullName -Encoding UTF8
    if ($lines.Length -ge 3 -and $lines[0] -match 'font-family|margin|padding|body\s*\{') {
        Set-Content $_.FullName -Value $lines[2..($lines.Length-1)] -Encoding UTF8
    }
}

# 清 B：截掉第一个 svg+xml 按钮之前的内容
Get-ChildItem -Path $target -Filter "*.md" | ForEach-Object {
    $content = Get-Content $_.FullName -Encoding UTF8 -Raw
    $idx = $content.IndexOf('![](data:image/svg+xml')
    if ($idx -ge 0) {
        Set-Content $_.FullName -Value $content.Substring(0, $idx).TrimEnd() -Encoding UTF8
    }
}
```

Bash 版（macOS/Linux）：

```bash
cd "$HOME/Desktop/曲曲大女人-md"
for f in *.md; do
  # 清 A
  head -1 "$f" | grep -qE 'font-family|margin|padding|body *\{' && sed -i '' '1,2d' "$f"
  # 清 B：截到 svg+xml 标记之前
  awk '/!\[\]\(data:image\/svg\+xml/{exit} {print}' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
done
```

### 第 7 步——蒸馏成 PERSONA.md（本 skill 的精髓）

让 Claude 通读清洗后的 .md 文件，输出一份**结构化的人格档案**到同目录 `PERSONA.md`。

如果文章总量超过单次 context（一般 100+ 篇 × 8KB ≈ 800KB 接近上限），按时间倒序**抽样 30~50 篇**，覆盖最新 + 中段 + 较老三段，比全量更稳。

`PERSONA.md` 推荐结构（这是 skill 提供的模板，可直接生成到目录里供用户复制）：

```markdown
# <公众号名> · Persona Profile

## 1. 身份与定位
- 作者自称 / 笔名
- 所在行业、专业背景（从文章里推断）
- 写作语言、目标读者画像

## 2. 核心议题（Top 5）
按出现频次和着墨长度排序。每个议题给一句话定义 + 2-3 个该作者的典型观点。

## 3. 思维框架
- 该作者分析问题的固定步骤（如"先看 K 线 → 再看仓位 → 最后给结论"）
- 用得最多的概念/术语（10 个以内，按频次）
- 偏好的论证方式（数据 / 历史类比 / 直觉 / 个人经验）

## 4. 价值观与情绪基调
- 对 X 类话题的立场
- 情绪曲线：什么情境下兴奋、什么情境下保守、什么情境下愤怒
- 一句话总结这个人的"信条"

## 5. 语言风格
- 标志性开场 / 收尾（该作者反复出现的固定句式，需从文章里实际找到具体例子，不要凭印象编造）
- 高频词、口头禅
- 句式特征（长句 vs 短句、是否爱用反问、是否爱用具体数字）

## 6. 已知盲区与立场偏见
- 该作者明显不擅长或回避的话题
- 立场倾向（看多/看空、左/右、保守/激进……）
- 读者要警惕的"过度自信领域"

## 7. 调用方式
当用户说"以 <公众号名> 的视角看 X"时，Claude 应：
- 套用第 3 节的分析步骤
- 用第 5 节的语言风格
- 在第 6 节列出的盲区给免责声明
- 不要伪造该作者从未表达过的具体数字 / 个股 / 病例细节
```

蒸馏 prompt 模板（Claude 内部使用，写进 SKILL.md 让运行时照抄）：

> 我将给你 <公众号名> 的 N 篇文章。请通读后，按上面 PERSONA.md 的 7 节结构输出 markdown。
> 要求：1) 每个论断必须能在文章里找到出处，不要编造；2) 第 5 节"高频词"要给具体例子（带原句片段）；3) 第 6 节"盲区"要从作者明显不谈的话题反推，不要客套；4) 输出后告诉用户保存到 `<目标目录>/PERSONA.md`。

## 常见故障

- **公众号添加：搜索结果空 / 加载不出**：检查是否已登录（右下角"登录信息过期时间还剩"）；过期重新扫码。
- **抓正文阶段：`screenshot` 一直 `document_idle` 超时**：正常现象，页面在持续抓取就不会 idle。改用 `find "抓取中 X/N"` 查进度，或读 `read_console_messages pattern: "进度|完成"`。
- **导出 Markdown 没文件出现**：检查 console 是否有 `AbortError: showDirectoryPicker` —— 原生对话框被取消了。让用户重点一次"导出 → Markdown"并**手动**操作弹出来的目录选择器。
- **触发风控（console: "文章下载失败（风控所致）"）**：工具会自动 1 秒后重试。如果整体失败率 > 5%，去"设置 → 代理节点"加私有代理（README 推荐 deno deploy）。
- **想要图片本地化**：换 HTML 格式导出（100% 还原排版，含图片二进制），不过同样要走目录选择器。Markdown 导出永远是远程 URL。
- **用户不愿意申请自己的公众号**：换路径——用 `KANIKIG/wechat-search-weread`（关键词搜索式，无登录，但召回不全且不能按时间过滤），明确告诉用户**召回会有遗漏**。

## 快速参考

```
[第 0 步] 问公众号名 / 时间范围 / 是否有自己的公众号
[第 1 步] list_connected_browsers → select_browser → 
          tabs_context_mcp(createIfEmpty) → navigate https://down.mptext.top
[第 2 步] 公众号管理 → 添加 → 搜索 → 选中目标
[第 3 步] 设置 → 同步时间范围 → ref 点选（read_page filter:"all" 拿到 listbox refs）
[第 4 步] 公众号管理 → 行末同步按钮；轮询 find "X/N" 直到完成
          → 文章下载 → 选公众号 → 全选 → 抓取→文章内容（Bash 后台 sleep 监控）
[第 5 步] PowerShell 建桌面文件夹 → 导出→Markdown → 
          ⚠️ 让用户手动操作弹出的原生目录选择器 → 等写入完成
[第 6 步] PowerShell/Bash 跑清洗脚本（删 CSS + 删 SVG 尾部）
[第 7 步] Claude 通读样本 .md → 按 7 节模板生成 PERSONA.md
```

**绝不省略的两件事**：① 步骤 5 前在文件系统里**先**建目标文件夹（否则用户在原生对话框里手忙脚乱）；② 步骤 7 蒸馏 prompt 里**强调出处验证**（不允许编造作者从未说过的具体数字 / 个案 / 立场）。
