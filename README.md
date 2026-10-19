# xhs-feishu：小红书 → 飞书笔记

一个 Claude Code skill：把小红书帖子整理成飞书云文档笔记。

- 通过你已登录小红书的 Chrome 读取帖子（需要 Claude in Chrome 扩展）
- 图文帖子：识别图片里的文字，并自动挑出有用的配图（图表、对比表、步骤图等）插入文档
- 视频帖子：有字幕直接用字幕；没字幕自动上传飞书妙记转成文字
- 用 `lark-cli` 写入飞书，同时在 `文稿/小红书笔记` 留一份本地备份（自动创建）
- 写不进飞书时，可手动导入本地的 Markdown 文件

## 安装

把 `xhs-feishu` 文件夹放到：

- Mac：`~/.claude/skills/xhs-feishu/`
- Windows：`C:\Users\你的用户名\.claude\skills\xhs-feishu\`

重新启动 `claude` 即可。

## 使用

```
/xhs-feishu <小红书链接>
/xhs-feishu <链接1> <链接2> <链接3>
```

使用前请确认：

- Chrome 已安装 Claude in Chrome 扩展，并且在 Chrome 里登录了小红书
- 已安装飞书命令行工具 `lark-cli` 并登录（`lark-cli auth login`）
- 视频转写会消耗飞书妙记的转写额度

想固定存到某个飞书文件夹，把文件夹链接填到 `SKILL.md` 里的 `FEISHU_FOLDER`。

## 致谢

改写自 [chenxiachan/xhs-claude-skills](https://github.com/chenxiachan/xhs-claude-skills)（MIT License）。
