---
name: xhs-feishu
description: 提取小红书帖子内容（文字、图片文字、关键配图、视频字幕；无字幕视频用飞书妙记转写），整理成飞书云文档。通过用户的 Chrome 读取帖子，用 lark-cli 写入飞书；写不进去就生成可导入飞书的 Markdown 文件。用户发来小红书（xiaohongshu.com / rednote.com / xhslink.com）链接并想存成笔记时使用；可一次给多个链接。
user-invocable: true
argument-hint: <小红书链接，可多个，空格或换行分隔>
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# 小红书 → 飞书笔记

基于 chenxiachan/xhs-claude-skills（MIT License）改写：去掉 Obsidian 依赖，改用 Chrome 读取帖子，无字幕视频用飞书妙记转写，关键配图插入飞书文档。

## 总原则
- **全程用中文和用户交流**，用户不是程序员，说明要通俗，少用术语。
- 不要让用户把 cookies、密码等登录信息贴进对话。
- 能自己完成的步骤就自己完成，需要用户动手时一次只给一个清楚的步骤。

## 常量（需要时可改）
- 工作目录（也是本地备份目录）：`~/Documents/小红书笔记`（不存在就自动创建）。下载、写文件、运行 `lark-cli` 都在这个目录下进行
- 飞书目标位置：`FEISHU_FOLDER = ""`（填飞书云空间文件夹链接或知识库节点链接；留空则问用户一次，并提醒用户可以把链接填到这里，以后就不用再问）
- 视频转写时，视频上传到的飞书文件夹：`FEISHU_VIDEO_FOLDER = ""`（留空则上传到云空间根目录）

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

**2a. 优先用平台字幕**
在 `video.media` 或 `video.mediaV2`（可能是 JSON 字符串，需再解析一次）里找 `subtitles`，语言优先级 source > zh-CN > en-US。用 curl 通过 HTTPS 下载 SRT（带 `-H "Referer: https://www.xiaohongshu.com/"`），去掉序号和时间戳，合并为连续文本。

**2b. 没有字幕时：自动用飞书妙记转文字**
前提是 `lark-cli` 可用（检查方法见步骤 4）。不要改用 Whisper、ffmpeg 等本地转录工具。按顺序执行，每一步失败就停在那一步并报告是哪一步失败：

1. **下载视频**：从 `video.media.stream` 里按 h264 > h265 > av1 取第一个可用的 `masterUrl`（http 改 https），立即下载（链接会过期）到工作目录的 `_tmp/` 下，命名 `{帖子ID}.mp4`：
   `curl.exe -L -o "_tmp/{帖子ID}.mp4" -H "Referer: https://www.xiaohongshu.com/" -H "User-Agent: Mozilla/5.0" "<视频URL>"`
   下载后确认文件大于 100KB，否则视为失败。
2. **上传到飞书云空间**：
   `lark-cli drive +upload --file "_tmp/{帖子ID}.mp4" --name "{帖子ID} 小红书视频.mp4" --as user`
   `FEISHU_VIDEO_FOLDER` 有值时加 `--folder-token <token>`。从结果取 `file_token`。
3. **生成妙记**：
   `lark-cli minutes +upload --file-token <file_token> --as user`
   从返回的 `minute_url` 路径最后一段取 `minute_token`（去掉 `?` 后的参数）。
4. **等待并取逐字稿**：
   `lark-cli minutes +detail --minute-tokens <minute_token> --wait-ready --transcript --as user`
   读取结果里 `artifacts.transcript_file` 指向的文本文件。仍在处理中时按提示轮询，**不要重复上传或重复创建妙记**。
5. 把逐字稿当作视频正文，进入 2d 清理。笔记属性里加一行"转写：飞书妙记"并附妙记链接。
6. 删除本地 `_tmp/` 里的视频。云空间里的视频文件和妙记保留，不要删除；完成时告诉用户它们在哪里，需要的话可以自己删。

**失败时的退路**（任何一步失败都按这个处理，不中断整篇笔记）：
- 没有 `lark-cli`、未登录或缺少妙记/云空间权限（`missing_scopes`）：按 lark-cli 的提示引导用户授权，授权后可重试；本篇先用标题和描述整理
- 返回 `quota_exceeded`（妙记转写额度用完）：告诉用户额度已用尽、可在妙记页面查看；本篇先用标题和描述整理，不要重试
- 上传成功但妙记创建失败：保留并报告 `file_token`，不要重复上传
- 以上情况在笔记里注明"视频未能转写，原因：…"

**2c. 视频画面**
不能截取视频画面。如果视频里有重要的画面信息（比如演示操作），在笔记里注明"关键内容在画面中，建议看原视频"。

**2d. 清理文本**
去掉尾部重复字、语气词和明显的识别错误，按语义加标点、分段；有步骤或要点结构时整理成列表。

### 图文帖子（type = normal）

**看图**：把 `images` 里的图片（http 改 https）用 curl 下载到工作目录的 `img/{帖子ID}/` 下，按顺序命名 `01`、`02`…。根据文件头判断格式并用对应扩展名（JPEG → `.jpg`，PNG → `.png`，WebP → `.webp`）。然后用 Read 工具逐张查看（每批最多 4 张）。

**每张图看完后：**
1. **图里的文字**：图片是正文载体（长文截图、PPT、清单）时，识别文字，按顺序拼接，修复跨图断句，写进"详情"。
2. **图片本身是否值得放进飞书文档**（最多 6 张）：
   - **放**：数据图表、对比表、流程图/步骤截图、地图/路线、价格单/菜单、产品或空间的关键实拍（正文在讲它）、文字识别后会丢失排版信息的图（如表格）
   - **不放**：纯装饰图、自拍/氛围图、封面标题图、文字已完整识别且排版不重要的纯文字图、重复角度的图
   - 拿不准时不放
3. 不放进文档的图片，删除其本地文件；要放的保留，供步骤 4 使用。

### 临时文件
视频等临时文件放在工作目录的 `_tmp/` 下，用完删除。要放进文档的图片保留在 `img/{帖子ID}/`，作为本地备份的一部分。

### 运行环境提示
- 不要假设电脑上有 Python。Windows 上经常没有，需要写脚本时优先用已有的工具（PowerShell、Node.js），不要为此安装 Python。
- Windows 下 `curl` 要写成 `curl.exe`，避免被 PowerShell 的同名命令替换。

## 步骤 3：按飞书格式整理

写作风格：直接、一句话给判断。笔记是决策工具——扫一眼就能决定深挖还是跳过。

**飞书格式规则：**
- 只用飞书能识别的 Markdown：标题 `#`/`##`/`###`、段落、**加粗**、有序/无序列表、引用 `>`、分割线 `---`、表格、普通链接 `[文字](URL)`
- **禁止** Obsidian 写法：`> [!tip]` 折叠块、`[[双链]]`、`#标签`、YAML frontmatter
- 不引用小红书图片网址（有防盗链，飞书里显示不出）。只插入步骤 2 选出并下载到本地的关键图片
- **关键图片放在"详情"里对应内容的旁边**，写成 `![图片说明](img/{帖子ID}/03.jpg)`。图片说明一句话讲清这张图展示了什么，比如"三家酒店早餐价格对比表"
- 没放进文档的图片，在笔记属性里写"原帖配图 N 张，已选 M 张放入文档"
- 正文里出现字面意义的 `<`（如"价格 < 500"）时写成 `\<`，否则飞书会把它当成标签
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
视频帖子：这里放整理后的字幕或妙记逐字稿要点。
关键图片插在它所说明的那一段后面。

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
| 配图 | 原帖 N 张，已选 M 张放入文档 |
| 转写 | 平台字幕 / 飞书妙记（附妙记链接）/ 未转写（原因）——仅视频帖子 |
| 整理日期 | 今天的日期 |
```

约束：分割线以上的内容不超过 6 行。

## 步骤 4：保存

**先存本地备份**：在工作目录 `~/Documents/小红书笔记/` 下保存 `{文档标题}.md`（UTF-8），图片引用写 `img/{帖子ID}/03.jpg` 这种相对路径。这样飞书写入失败也不会丢内容。

**写入飞书（首选 lark-cli）：**
1. 检查：`Get-Command lark-cli`（Mac 用 `which lark-cli`）。找不到时看有没有飞书 / Lark 的 MCP 工具（名称含 `lark` 或 `feishu`），有就用它创建文档（这种情况下图片可能无法插入，在回复里说明）
2. 第一次使用前，先运行 `lark-cli auth status` 确认已登录；未登录就请用户运行 `lark-cli auth login` 并按提示在浏览器里授权
3. 生成一份飞书专用副本 `_tmp/{帖子ID}.feishu.md`：内容同本地备份，但把每个图片路径前加 `@./`，即 `![说明](@./img/{帖子ID}/03.jpg)`（lark-cli 只认这种写法上传本地图片，且路径必须在当前目录内）。文档第一行的 `# 标题` 去掉，标题改用 `--title` 传
4. **在工作目录下**执行：
   `lark-cli docs +create --doc-format markdown --title "{文档标题}" --content "@./_tmp/{帖子ID}.feishu.md" --as user`
   有目标文件夹时加 `--parent-token <文件夹或知识库节点 token>`（从用户给的链接里取最后一段）
   这里有意使用 Markdown 格式（是对已写好笔记的保真导入），即使 lark-cli 自带的指引默认推荐 XML 也照此执行
5. 检查返回结果：`ok` 为 true 且有 `document.url` 才算成功；`warnings` 不为空时（比如某张图上传失败）要告诉用户。返回 `missing_scopes`（缺少权限）时，按 lark-cli 的提示引导用户补充授权后重试
6. 删除 `_tmp/` 下的副本

写入位置：
- `FEISHU_FOLDER` 有值就放进去
- 为空时，本次第一篇写之前问用户一次要放哪个文件夹（请用户在飞书里打开那个文件夹，复制地址栏链接发过来），整批沿用；用户说"随便"就放云空间根目录。完成后提醒用户可以把链接填进 skill 的 `FEISHU_FOLDER`

写入成功后给用户文档链接。写入失败时，说明原因，告诉用户本地文件位置和导入方法：飞书云文档 → 新建 → 导入 → Markdown（这种方式图片不会一起导入，需要手动拖进去）。

## 多个链接时
- 逐个处理，每篇之间间隔 3 秒，避免触发小红书风控
- 某一篇失败不影响其他篇
- 全部完成后用中文简短汇总：每篇的标题、一句话洞察、飞书链接；失败的写明原因

## 单篇完成后的回复
只说这几件事：飞书链接（或本地文件位置）、一句话核心洞察、值得深挖吗；放了几张图；视频是怎么转写的（字幕 / 妙记 / 未转写及原因）。不要把整篇笔记再贴一遍。
