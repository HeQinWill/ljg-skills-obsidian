---
name: ljg-x-download
description: "Download images and videos from X (Twitter) posts to ~/Downloads. Use when user shares an X/Twitter link and wants to save media, or says '下载', 'download', '保存图片', '保存视频', or provides a x.com/twitter.com URL with intent to download media."
version: "1.1.0"
user_invocable: true
---

# ljg-x-download

从 X (Twitter) 链接下载图片或视频到 ~/Downloads。

## 依赖

- `yt-dlp`（已安装）

## 执行流程

### 1. 解析输入

从用户输入中提取 X/Twitter URL。支持的格式：
- `https://x.com/user/status/123456`
- `https://twitter.com/user/status/123456`
- `https://mobile.twitter.com/user/status/123456`
- 带查询参数的 URL（yt-dlp 自动处理 `?s=20` 等追踪参数）

缩短链接（t.co）先解析：`curl -Ls -o /dev/null -w '%{url_effective}' "SHORT_URL"`

如果用户没有提供 URL，用 AskUserQuestion 要求提供。

### 2. 尝试直接下载（视频优先）

直接用 yt-dlp 下载，无需先探测：

```bash
yt-dlp -o "~/Downloads/%(uploader)s_%(id)s.%(ext)s" "URL"
```

如果成功（视频推文），完成。跳到步骤 4 汇报结果。

### 3. 视频下载失败时，提取图片

yt-dlp 对纯图片推文可能报错。此时用 `--dump-json` 提取图片 URL：

```bash
yt-dlp --dump-json "URL" 2>&1
```

**判断结果：**
- JSON 中有 `thumbnails` 数组 → 提取图片 URL
- JSON 为空或报错 `no video` → 推文无媒体，告知用户"该推文不包含可下载的图片或视频"
- 报错含 `login` / `authentication` → 需要登录（见故障排除）
- 其他错误 → 报告具体错误信息

**图片下载：**

从 JSON 的 `thumbnails` 数组提取所有图片 URL，替换 `name=small` 或 `name=medium` 为 `name=orig` 获取原图，然后逐一下载：

```bash
curl -L -o ~/Downloads/tweet_ID_1.jpg "https://pbs.twimg.com/media/xxx?format=jpg&name=orig"
curl -L -o ~/Downloads/tweet_ID_2.jpg "https://pbs.twimg.com/media/yyy?format=jpg&name=orig"
```

文件扩展名跟随 URL 中的 `format` 参数（jpg/png/webp）。

### 4. 汇报结果与 Obsidian 归档

下载完成后：
1. 用 `ls -lh` 列出已下载的文件。
2. **Obsidian 归档 (可选)**: 若环境包含 Obsidian 库，建议在 `~/Obsidian/aitalk/` 生成一个归档笔记：
   - 文件名：`{时间戳}--X下载-{uploader}__xdown.md`
   - YAML Frontmatter:
     ```yaml
     ---
     title: "X 下载：{uploader}"
     date: {{YYYY-MM-DD HH:mm}}
     tags:
       - x-media
       - download
     skill: ljg-x-download
     source: {URL}
     ---
     ```
   - 内容：包含原贴链接、下载路径、以及（如果是图片）使用 `![[图片文件名]]` 的预览。

## 故障排除
...

### 需要登录

yt-dlp 报错含 `login` / `Sign in` / `age-restricted` 时，加 `--cookies-from-browser chrome`：

```bash
yt-dlp --cookies-from-browser chrome -o "~/Downloads/%(uploader)s_%(id)s.%(ext)s" "URL"
```

### 推文无媒体

纯文字推文没有可下载的媒体。告知用户即可。
