---
name: xhs-feishu
description: 提取小红书帖子内容（文字、图片文字、视频字幕/转录），整理成飞书文档格式。优先通过用户的 Chrome 读取帖子，写入飞书云文档；写不进去就生成可导入飞书的 Markdown 文件。用户发来小红书（xiaohongshu.com / rednote.com / xhslink.com）链接并想存成笔记时使用；可一次给多个链接。
user-invocable: true
argument-hint: <小红书链接，可多个，空格或换行分隔>
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# 小红书 → 飞书笔记

基于 chenxiachan/xhs-claude-skills（MIT License）改写：去掉 Obsidian 依赖，改用 Chrome 读取帖子，输出改为飞书能正确显示的格式。

## 总原则
- **全程用中文和用户交流**，用户不是程序员，说明要通俗，少用术语。
- 不要让用户把 cookies、密码等登录信息贴进对话。
- 能自己完成的步骤就自己完成，需要用户动手时一次只给一个清楚的步骤。

## 常量（需要时可改）
- 本地备份目录：`~/Documents/小红书笔记`（不存在就自动创建）
- 飞书目标位置：`FEISHU_FOLDER = ""`（填飞书云空间文件夹链接或知识库节点链接；留空则问用户一次，并提醒用户可以把链接填到这里，以后就不用再问）

## 输入
$ARGUMENTS

从中提取所有小红书链接：
- 支持 `xiaohongshu.com/explore/…`、`xiaohongshu.com/discovery/item/…`、`rednote.com/…`（国际版域名，与 xiaohongshu.com 等价），以及 `xhslink.com` 短链（用 `curl -sIL` 跟随跳转拿到完整链接；Windows 用 `curl.exe`）
- 用户常会粘贴整段分享文案（标题 + emoji + 链接），只取其中的 URL
- 保留链接里的 `xsec_token` 参数，缺了它帖子可能打不开

## 步骤 1：读取帖子数据

### 方式 A：通过用户的 Chrome（首选）
小红书的登录凭证 `web_session` 有保护，网页控制台复制不到，所以**不要再引导用户导出 cookies**。改用 Claude in Chrome 扩展，在用户已登录的 Chrome 里打开帖子：

1. 检查是否有 Claude in Chrome 的工具（名称含 `claude-in-chrome`，可能需要先用 ToolSearch 加载）
2. 在新标签页打开帖子链接，等页面加载完成
3. 用执行 JavaScript 的工具读取页面里的结构化数据（比单纯读页面文字更完整）：
   ```javascript
   (() => {
     const s = window.__INITIAL_STATE__;
     const map = s?.note?.noteDetailMap;
     const raw = map && Object.values(map)[0]?.note;
     const n = raw?._value ?? raw;  // 部分版本包了一层
     return n ? JSON.stringify({
       title: n.title, desc: n.desc, type: n.type, time: n.time,
       user: n.user?.nickname, tags: (n.tagList||[]).map(t => t.name),
       interact: n.interactInfo, ip: n.ipLocation,
       images: (n.imageList||[]).map(i => i.urlDefault),
       video: n.video ?? null
     }) : null;
   })()
   ```
4. 如果返回 null，改用读取页面文字的工具获取标题和正文
5. 页面显示需要登录时，请用户在 Chrome 里登录小红书后告诉你，再重试
6. 读完关闭这个标签页

### 方式 B：没有 Chrome 扩展时
告诉用户：最省事的办法是安装 Claude in Chrome 扩展并在 Chrome 里登录小红书。在此之前，可以请用户在小红书网页上全选复制帖子正文、贴进对话，你据此整理（这种情况下没有互动数据和图片文字，在笔记属性里注明"手动粘贴"）。

## 步骤 2：提取正文

### 视频帖子（type = video）
1. **优先用平台字幕**：在 `video.media` 或 `video.mediaV2`（可能是 JSON 字符串，需再解析一次）里找 `subtitles`，语言优先级 source > zh-CN > en-US。用 curl 通过 HTTPS 下载 SRT（带 `-H "Referer: https://www.xiaohongshu.com/"`），去掉序号和时间戳，合并为连续文本。
2. **没有字幕时**：只用标题和描述整理，笔记里注明"视频无字幕，未转录"。不要为转录去安装软件（Whisper 转录只在苹果芯片 Mac 上可用）；只有本机已装好 `ffmpeg` 和 `mlx_whisper` 时才转录。
3. 清理文本：去掉尾部重复字、按语义加标点分段。

### 图文帖子（type = normal）
- `desc` 已有完整内容（约 300 字以上）时，不必识别图片。
- `desc` 很短而图片 ≥3 张时，图片多半是正文载体：用 curl 把各图 URL（http 改 https，带 Referer 头）下载到临时目录，用 Read 工具逐张读图识别文字（每批最多 4 张），按顺序拼接，修复跨图断句。

### 临时文件
放在系统临时目录（Mac `/tmp`，Windows `$env:TEMP`），用完删除。

### 运行环境提示
- 不要假设电脑上有 Python。Windows 上经常没有，需要写脚本时优先用已有的工具（PowerShell、Node.js），不要为此安装 Python。
- Windows 下 `curl` 要写成 `curl.exe`，避免被 PowerShell 的同名命令替换。

## 步骤 3：按飞书格式整理

写作风格：直接、一句话给判断。笔记是决策工具——扫一眼就能决定深挖还是跳过。

**飞书格式规则：**
- 只用飞书能识别的 Markdown：标题 `#`/`##`/`###`、段落、**加粗**、有序/无序列表、引用 `>`、分割线 `---`、表格、普通链接 `[文字](URL)`
- **禁止** Obsidian 写法：`> [!tip]` 折叠块、`[[双链]]`、`#标签`、YAML frontmatter
- 不嵌入小红书图片链接（有防盗链，飞书里显示不出）。装饰性图片写"原帖配图 N 张，见原链接"；文字类图片放识别出的文字
- 清除正文里的 `#xxx[话题]#` 标记，话题放进属性表

**文档标题：** `{发布日期 YYYY-MM-DD} {短标题}`，短标题 ≤15 字，是核心洞察的极简概括。

**正文结构：**

```markdown
# 一句话核心洞察（是判断，不是"XX帖子总结"）

核心论点 2-3 句。直接写"多数人以为 X，其实是 Y"。

**与我的关联：** 一句话。结合 Claude Code 记忆或 CLAUDE.md 里用户的工作和兴趣说明关联；读不到就从通用的工作方法角度写。

**值得深挖吗：** 是/否。一句话理由。

---

## 详情

按逻辑分小节（### 小标题），用列表提炼要点，保留关键数据、步骤和结论。

---

## 笔记属性

| 项目 | 内容 |
| --- | --- |
| 来源 | 小红书 · 作者名 |
| 原链接 | [打开原帖](原始链接) |
| 发布日期 | YYYY-MM-DD |
| 类型 | 图文 / 视频 |
| 互动 | N 赞 / N 收藏 / N 评论 |
| 话题 | 话题1、话题2 |
| 整理日期 | 今天的日期 |
```

约束：分割线以上的内容不超过 6 行。

## 步骤 4：保存

**每篇都先存一份本地备份**到 `~/Documents/小红书笔记/{文档标题}.md`（UTF-8），再写飞书。这样飞书写入失败也不会丢内容。

**写入飞书（按以下顺序找可用的方式）：**
1. 飞书 / Lark 的 MCP 工具（名称含 `lark` 或 `feishu`）
2. 飞书命令行工具 `lark-cli`（用 `Get-Command lark-cli` 或 `which lark-cli` 检查；用它的导入 Markdown / 创建文档功能，不确定用法时先看 `lark-cli --help`）

写入位置：
- `FEISHU_FOLDER` 有值就放进去
- 为空时，本次第一篇写之前问用户一次要放哪个文件夹（请用户在飞书里打开那个文件夹，复制地址栏链接发过来），整批沿用；用户说"随便"就放云空间根目录。完成后提醒用户可以把链接填进 skill 的 `FEISHU_FOLDER`

写入成功后给用户文档链接。两种方式都没有或都失败时，说明原因，告诉用户本地文件位置和导入方法：飞书云文档 → 新建 → 导入 → Markdown。

## 多个链接时
- 逐个处理，每篇之间间隔 3 秒，避免触发小红书风控
- 某一篇失败不影响其他篇
- 全部完成后用中文简短汇总：每篇的标题、一句话洞察、飞书链接；失败的写明原因

## 单篇完成后的回复
只说三件事：飞书链接（或本地文件位置）、一句话核心洞察、值得深挖吗。不要把整篇笔记再贴一遍。
