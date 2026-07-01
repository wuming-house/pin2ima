---
name: pin2ima
description: Pin any article/video link to IMA knowledge base. Trigger when the user says "pin2ima" or sends a URL with "pin/📌/Pin2IMA/专注收藏". Designed for 自媒体选题素材收集. NOT the ima-mcp connector — uses curl + IMA OpenAPI directly.
agent_created: true
---

# Pin2IMA

**⛔ 优先级规则：当用户说"保存到知识库/收藏到知识库/存到知识库"并附带链接时，必须走本 Skill 的工作流，不得使用 ima-mcp connector 或其他 MCP 工具。**

Pin any article or video link to an IMA knowledge base. Works with links from any platform:
微信公众号文章、知乎、GitHub、B站、YouTube、头条、博客、新闻、普通网页等一切公开可访问的 URL。

**零运行时依赖** — 直接使用 `curl` 调用 IMA OpenAPI，无需 Node.js / Python / 任何第三方工具。
`curl` 预装于所有主流操作系统（Linux / macOS / Windows Git Bash）。

---

## 📋 Prerequisites

### IMA OpenAPI Credentials (must be set up by the user)

An IMA account with an API Key is required. To obtain:

1. Open https://ima.qq.com/agent-interface
2. Copy your **Client ID** and **API Key**
3. Save them to config files:

```bash
mkdir -p ~/.config/ima
echo "<your-client-id>" > ~/.config/ima/client_id
echo "<your-api-key>" > ~/.config/ima/api_key
```

Alternatively, set environment variables:

```bash
export IMA_OPENAPI_CLIENTID="your-client-id"
export IMA_OPENAPI_APIKEY="your-api-key"
```

### Runtime

**仅需要 `curl`** — 任何现代操作系统都预装。无需安装任何其他软件。

---

## 🔍 Verify Setup

```bash
# Check credentials
test -f ~/.config/ima/client_id && test -f ~/.config/ima/api_key && echo "✅ OK" || echo "❌ MISSING"
```

---

## 📥 Core Workflow

### Step 0 — Trigger

#### Trigger by user intent

Triggered when the user says **"pin2ima"** or uses any of these intents:
- "pin2ima 保存" / "pin2ima 收藏"
- "pin 到知识库" / "📌 收藏" / "Pin2IMA 保存"
- 直接发链接说 "专注收藏"

**⚠️ ima-mcp 冲突说明：** 如果用户装了 ima-mcp connector，"保存到知识库"等短语会优先被 MCP 拦截。
此时需用户说出 **"pin2ima"** 或 **"📌"** 关键词来明确触发本 Skill，或在对话中告知用户用 "pin2ima 收藏" 代替 "收藏到知识库"。

#### URL type detection

Before proceeding, determine whether the URL points to an **article** (web page) or a **video**:

| Platform | Detection | Type |
|----------|-----------|------|
| `mp.weixin.qq.com` | Always article | article |
| `xiaohongshu.com` / `xhslink.com` | 小红书（需 WebFetch 区分图文/视频） | article-note |
| `zhuanlan.zhihu.com` / `zhihu.com` | 知乎文章 | article |
| General web page (no video platform host) | Assume article | article |
| `youtube.com/watch` / `youtu.be/` | YouTube video | video |
| `bilibili.com/video/` / `b23.tv/` | B站视频 | video |
| `douyin.com/video/` | 抖音 | video |
| `ixigua.com` | 西瓜视频 | video |
| `kuaishou.com` | 快手 | video |
| `channels.weixin.qq.com` / `weixin.qq.com/sph` | 视频号 | video |
| URL ends with `.mp4` / `.mov` / `.webm` etc. | Direct video file | video |

- **article** → follow the standard Core Workflow below
- **article-note** → follow the [Article-Note Workflow](#-文章笔记工作流支持自定义标题的平台) (like video, but with page title)
- **video** → jump to the [Video Workflow](#-进阶功能自媒体选题场景) section

### Step 1 — Build Auth Header

Before any API call, compute the auth header:

```bash
CLIENT_ID=$(cat ~/.config/ima/client_id)
API_KEY=$(cat ~/.config/ima/api_key)
```

All API requests go to `https://ima.qq.com/{apiPath}` with these HTTP headers:

```
ima-openapi-clientid: <client_id>
ima-openapi-apikey: <api_key>
Content-Type: application/json
ima-openapi-ctx: skill_version=1.0.0
```

### Step 2 — Find Target Knowledge Base

Determine which knowledge base to save to. Priority:

1. **User explicitly names one** → use it
2. **User says "知识库" without naming** → default to **"选题知识库"**
3. **No context** → ask the user: "你想保存到哪个知识库？"

Search by name:

**⚠️ 编码注意事项：** 在 Windows 环境中，`curl` 的 JSON body 中包含中文直接量可能因 GBK 编码导致乱码。
**解决方案：** 使用 `\uXXXX` Unicode 转义序列代替中文（例如 `公众号` → `\u516c\u4f17\u53f7`）。
Linux / macOS 无此问题。AI 代理在处理时，应自动将查询中的中文转换为 `\uXXXX` 格式再放入 JSON。

**Python 转换辅助（AI 代理可用此方法将中文 KB 名转为 \u 转义）：**
```python
# 将中文字符串转为 \uXXXX 格式
name = "选题知识库"
escaped = name.encode('unicode_escape').decode('ascii')
# 结果: \u516c\u4f17\u53f7\u6587\u7ae0\u77e5\u8bc6\u5e93
```

```bash
CLIENT_ID=$(cat ~/.config/ima/client_id)
API_KEY=$(cat ~/.config/ima/api_key)

# 用 \u 转义的中文构造查询（示例查询 "选题知识库"）
RESPONSE=$(curl -s "https://ima.qq.com/openapi/wiki/v1/search_knowledge_base" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d '{"query":"\u9009\u9898\u77e5\u8bc6\u5e93","cursor":"","limit":5}')
```

**Parse the response:**

Success response shape:
```json
{"code":0, "msg":"success", "data":{"info_list":[{"kb_id":"...","kb_name":"..."}]}}
```

- `code=0` → extract `kb_id` from `data.info_list[0].kb_id`
- `info_list` empty → knowledge base not found

**If no match — 引导用户创建知识库：**

当 `info_list` 为空时，知识库不存在。**不要猜测、不要自行搜索、不要默认存到其他库**。直接给出以下指引：

```
❌ 未找到「选题知识库」

请先在 IMA 中手动创建，操作步骤：

1. 打开 IMA 客户端（电脑版或微信小程序均可）
2. 进入「知识库」页面
3. 点击「新建知识库」
4. 输入名称：「选题知识库」
5. 点击「确定」创建

创建完成后，再发一次链接给我即可保存。
```

**说明：** IMA OpenAPI 没有提供创建知识库的接口，必须由用户在 IMA 客户端中手动创建。
一旦创建完成，Agent 即可通过名称搜索到并正常使用。

### Step 3 — Import the URL

```bash
CLIENT_ID=$(cat ~/.config/ima/client_id)
API_KEY=$(cat ~/.config/ima/api_key)
KB_ID="<kb-id-from-step-2>"
URL="<the-user-provided-url>"

RESPONSE=$(curl -s "https://ima.qq.com/openapi/wiki/v1/import_urls" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "{\"knowledge_base_id\":\"$KB_ID\",\"urls\":[\"$URL\"]}")
```

### Step 4 — Present Result

Parse `RESPONSE`:

| Condition | Action |
|-----------|--------|
| `code=0` | ✅ 已添加到知识库「{kb_name}」✓ |
| `code≠0` | 直接显示 `msg` 字段内容给用户 |
| curl error (empty or http error) | 显示网络错误，建议重试 |

**Progress reporting (concise):**
- During import: "正在保存文章到知识库…"
- On success: "✅ 已添加到知识库「{kb_name}」✓"
- On failure: "❌ 保存失败：{reason}"

---

## 📝 文章笔记工作流（支持自定义标题的平台）

部分平台（小红书等）的页面用 `import_urls` 保存后，标题会被 IMA 识别为站点名称（如"小红书"）而非实际笔记标题。
对于这些平台，用笔记方式保存，确保标题准确。同时根据内容类型（图文/视频）走不同的内容模板。

### 触发条件

URL 类型检测为 `article-note` 时走此流程。

### 第 1 步：WebFetch 判断内容类型

发送 WebFetch 请求抓取页面，同时判断内容类型：

| 判断依据 | 内容类型 | 后续流程 |
|---------|---------|---------|
| 页面包含视频播放器 / 视频标签 / 时长信息 | **视频** | 走 [视频处理 层级三](#层级三降级方案--页面信息快照)（页面快照 + 视频笔记模板） |
| 页面是图文笔记（无视频特征） | **图文** | 走下方图文笔记流程 |

### 第 2 步：图文笔记流程

对于小红书图文笔记，提取页面信息后创建简洁笔记：

```
📥 正在保存到「选题知识库」…

📝 小红书图文
标题：【手把手教你搭建个人知识库】全程干货
来源：小红书
摘要：从零开始搭建个人知识管理系统的完整教程…

✅ 笔记已保存 → 搜索标题即可在知识库中查看
```

### 第 3 步：视频走视频流程

对于小红书视频，跳转到 [视频处理层级三](#层级三降级方案--页面信息快照)：
1. 抓取页面信息（标题、简介、播放数据）
2. AI 生成视频摘要
3. 创建笔记（使用视频笔记模板，含来源链接 + 摘要）
4. 挂载到知识库

### 参考命令（图文笔记）

```bash
# Step A: WebFetch 抓取页面，提取标题和正文内容

# Step B: 用 Python + ensure_ascii=True 构造 JSON
NOTE_JSON=$(python3 -c "
import json
content = f'## {REAL_TITLE}\n\n**来源：** {URL}\n**平台：** 小红书\n\n### AI 摘要\n{SUMMARY}\n\n---\n*由 pin2ima 自动收录*'
print(json.dumps({'content_format': 1, 'content': content, 'folder_name': '\u9009\u9898\u7b14\u8bb0'}, ensure_ascii=True))
")

NOTE_RESP=$(curl -s "https://ima.qq.com/openapi/note/v1/import_doc" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "$NOTE_JSON")

# Step C: 挂到知识库（title 用 \uXXXX 转义避免编码问题）
ADD_JSON=$(python3 -c "
import json
print(json.dumps({'media_type': 11, 'note_info': {'content_id': '$NOTE_ID'}, 'title': '\u5c0f\u7ea2\u4e66\u56fe\u6587', 'knowledge_base_id': '$KB_ID'}, ensure_ascii=True))
")

ADD_RESP=$(curl -s "https://ima.qq.com/openapi/wiki/v1/add_knowledge" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "$ADD_JSON")
```

---

## 🚀 进阶功能（自媒体选题场景）

以下功能扩展了核心流程，专门服务于自媒体选题调研、热点追踪、内容策划的场景。

### 1. 批量保存

一次保存多个链接，适合做选题调研时批量收集素材。

```bash
CLIENT_ID=$(cat ~/.config/ima/client_id)
API_KEY=$(cat ~/.config/ima/api_key)

RESPONSE=$(curl -s "https://ima.qq.com/openapi/wiki/v1/import_urls" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "{\"knowledge_base_id\":\"$KB_ID\",\"urls\":[\"$URL1\",\"$URL2\",\"$URL3\"]}")
```

**输出：** 汇总报告每个 URL 的成功/失败状态，而非逐条提示。

### 2. 指定目标知识库（覆盖默认）

默认保存到「选题知识库」。如果用户想存到其他知识库，直接在指令中指定库名即可。

```bash
# 保存到其他知识库
把 https://example.com 保存到「某个知识库名称」
```

### 3. 保存时附带用户备注 + 自动摘要

适用场景：用户已经读过文章，觉得不错，想存下来作为选题参考。
**不需要帮用户判断文章好不好**——用户已经判断过了。要做的是帮他记录"为什么存了它"，方便以后翻知识库时快速回忆。

**流程：**

1. 用户发送 URL 时，**主动询问是否需要加备注**（如："这篇想用来做什么选题？"）
2. 如果用户有备注 → 将备注信息一并存档
3. 保存后，**AI 自动生成一段摘要**，作为将来回顾时的参考

**示例输出（用户加了备注）：**

```
📥 正在保存到「公众号文章知识库」…

📝 存档信息
标题：免费 Token 烧掉 5 万亿之后，他们出了个一站式创作平台
用户备注：这篇的免费策略分析可以作为 AI 工具号选题参考
AI 摘要：Agnes AI 通过免费 API 策略7天烧掉5万亿Token后推出Pavo一站式创作平台，
        整合图片/视频/短剧生成。值得关注的是其免费模型策略对行业的冲击。

✅ 已保存到「公众号文章知识库」
```

**示例输出（用户没加备注，简洁模式）：**

```
📥 正在保存到「公众号文章知识库」…

✅ 已保存
标题：免费 Token 烧掉 5 万亿之后，他们出了个一站式创作平台
AI 摘要：Agnes AI 通过免费API策略7天烧掉5万亿Token后推出Pavo一站式创作平台。

📌 下次翻到这篇时，AI摘要能帮你快速想起内容
```

**关键规则：**
- 备注由用户主动提供，不强制询问（用户赶时间就直接存）
- AI 摘要是**保存后的附加增值**，不影响保存主流程的响应速度
- 摘要控制在 2 句话以内，精炼为主
- 不要问"你确定要存吗"——用户发来链接本身就已经是确定的信号

### 4. 重复检测

保存前检查该 URL 是否已在知识库中存在，避免重复收集。

```bash
CLIENT_ID=$(cat ~/.config/ima/client_id)
API_KEY=$(cat ~/.config/ima/api_key)

# 在知识库中搜索该 URL
RESPONSE=$(curl -s "https://ima.qq.com/openapi/wiki/v1/search_knowledge" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "{\"query\":\"$ARTICLE_TITLE_OR_URL\",\"knowledge_base_id\":\"$KB_ID\",\"cursor\":\"\"}")
```

**逻辑：**
- 如发现已存在同名或同 URL 内容 → 提示用户"该文章已在知识库中"，询问是否仍要再次保存
- 如未找到 → 正常执行导入
- 此检查为非强制步骤，不要因为搜索耗时影响用户体验

### 5. 跨知识库搜索

用户想知道"之前有没有收集过某个话题"时，跨所有知识库一键搜索：

```bash
# 搜索知识库列表
KB_LIST=$(curl -s "https://ima.qq.com/openapi/wiki/v1/search_knowledge_base" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d '{"query":"","cursor":"","limit":20}')

# 遍历每个知识库搜索关键词
for KB in $(echo "$KB_LIST" | jq -c '.data.info_list[]'); do
  KB_ID=$(echo "$KB" | jq -r '.kb_id')
  # 在每个知识库中搜索
done
```

**使用场景：** "帮我搜一下知识库里有没有关于 'AI 视频' 的文章"

### 6. 今日选题速览（自动化 + 可选的定时任务）

每天自动从知识库中提取最新的收藏文章，生成一份选题速览。这项功能需要结合 WorkBuddy 的定时任务（automation）使用。

**automation prompt 示例：**
```
从「公众号文章知识库」中列出最近 7 天收藏的文章，
按热度/话题分组，输出一份选题速览报告。
重点标注哪些话题出现频率高、哪些角度值得跟进。
```

---

## 🎬 视频链接处理（独立工作流）

当触发检测识别为视频链接时，走此独立流程。

### 核心原则

- **视频内容无法直接存入 IMA 知识库**，`import_urls` 仅保存一个 URL 书签，价值有限
- 本工作流的目的是：将视频的核心信息提取成摘要，**通过笔记永久保存到知识库中**
- 笔记内容已包含来源链接，**不再重复保存 URL 条目** — 只保留笔记一条记录

### 前置：笔记 API 用法

```bash
# API 1: import_doc — 创建笔记 (POST https://ima.qq.com/openapi/note/v1/import_doc)
# Body: {"content_format": 1, "content": "markdown正文", "folder_name": "选题笔记"}
# 返回: {"code": 0, "data": {"note_id": "xxx"}}

# API 2: add_knowledge — 挂载笔记到知识库 (POST https://ima.qq.com/openapi/wiki/v1/add_knowledge)
# Body: {"media_type": 11, "note_info": {"content_id": "<note_id>"}, "title": "标题", "knowledge_base_id": "<kb_id>"}
```

**笔记内容格式：** `content_format` 固定为 `1`（Markdown）。

**⛔ 编码铁律：**

所有通过 curl 传递中文的 API 调用，**必须使用 Python `json.dumps(body, ensure_ascii=True)` 构造 JSON body**。这会自动将中文转为 `\uXXXX` 转义序列，避免 Windows bash/curl 的 GBK 编码破坏。

```python
# ✅ 正确
body = {'content_format': 1, 'content': '中文字符', 'folder_name': '\u9009\u9898\u7b14\u8bb0'}
json_str = json.dumps(body, ensure_ascii=True)
# → {"content_format": 1, "content": "\u4e2d\u6587\u5b57\u7b26", "folder_name": "\u9009\u9898\u7b14\u8bb0"}
```

```bash
# 配合 curl 使用
NOTE_JSON=$(python3 -c "... json.dumps(body, ensure_ascii=True) ...")
curl -s "https://ima.qq.com/path" -H "..." -d "$NOTE_JSON"
```

**解决方案：写入笔记前，用 Python 清洗所有字符串字段：**

```python
# 清洗非法 UTF-8 字节
content = raw_content.encode('utf-8', 'ignore').decode('utf-8')
title = raw_title.encode('utf-8', 'ignore').decode('utf-8')
```

**⛔ JSON 构造规则（curl 传递中文时必须遵循）：**

在 Windows 环境下，通过 `curl -d` 传递 JSON body 时，中文直接量会被 GBK 编码破坏。
**必须使用 Python `json.dumps(body, ensure_ascii=True)` 构造 JSON**，
将所有中文转为 `\uXXXX` 转义序列，服务端能正确解码。

```python
# ✅ 正确：ensure_ascii=True，中文不会因 bash/curl 产生编码问题
import json
body = {'content_format': 1, 'content': content, 'folder_name': '\u9009\u9898\u7b14\u8bb0'}
print(json.dumps(body, ensure_ascii=True))

# ❌ 错误：ensure_ascii=False，中文在 Windows 的 bash/curl 中会乱码
body = {'content_format': 1, 'content': content, 'folder_name': '选题笔记'}
print(json.dumps(body, ensure_ascii=False))
```

或在 bash 中处理：

```bash
# 用 python3 清洗变量中的非法 UTF-8
CLEAN_CONTENT=$(printf '%s' "$RAW_CONTENT" | python3 -c "import sys; sys.stdout.write(sys.stdin.buffer.read().decode('utf-8','ignore'))")
CLEAN_TITLE=$(printf '%s' "$RAW_TITLE" | python3 -c "import sys; sys.stdout.write(sys.stdin.buffer.read().decode('utf-8','ignore'))")
```

**强制执行：** 在调用 `import_doc` 之前，**必须**对 `content` 和 `title` 执行上述清洗。无论内容来源是 WebFetch、API 返回还是变量拼接，都不能假设已经是合法 UTF-8。

### 通用保存步骤（所有层级共用）

无论哪个层级获得摘要后，统一执行：

```bash
# Step A: 获取知识库 ID（同文章工作流 Step 2）
# ... search_knowledge_base ...

# Step B: 用 Python 构造 JSON（ensure_ascii=True 防止 Window 下中文乱码）
NOTE_JSON=$(python3 -c "
import json
content = f'## $VIDEO_TITLE\n\n**来源：** $VIDEO_URL\n**平台：** $PLATFORM\n**发布时间：** $PUB_DATE\n\n### AI 摘要\n$SUMMARY\n\n---\n*由 pin2ima 于 \$(date +%Y-%m-%d) 自动收录*'
print(json.dumps({'content_format': 1, 'content': content, 'folder_name': '\u9009\u9898\u7b14\u8bb0'}, ensure_ascii=True))
")

NOTE_RESP=$(curl -s "https://ima.qq.com/openapi/note/v1/import_doc" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "$NOTE_JSON")
# 解析 NOTE_RESP 获取 note_id

# Step C: 挂载到知识库（同样用 ensure_ascii=True）
ADD_JSON=$(python3 -c "
import json
print(json.dumps({'media_type': 11, 'note_info': {'content_id': '$NOTE_ID'}, 'title': '\u5feb\u770b\u5b66\u957f - \u9009\u9898\u7b14\u8bb0', 'knowledge_base_id': '$KB_ID'}, ensure_ascii=True))
")

ADD_RESP=$(curl -s "https://ima.qq.com/openapi/wiki/v1/add_knowledge" \
  -H "ima-openapi-clientid: $CLIENT_ID" \
  -H "ima-openapi-apikey: $API_KEY" \
  -H "ima-openapi-ctx: skill_version=1.0.0" \
  -H "Content-Type: application/json" \
  -d "$ADD_JSON")

### 层级一：有公开字幕（B站 / YouTube）

B站和 YouTube 有公开的字幕/CC 字幕接口，直接拉取字幕做 AI 摘要，轻量高效。

**字幕拉取方案：**

| 平台 | 方案 | 命令 |
|------|------|------|
| YouTube | `yt-dlp` | `yt-dlp --skip-download --write-auto-sub --sub-lang "zh-Hans,en" --output "%(id)s" "URL"` |
| YouTube | `youtube-transcript-api` | `pip install youtube-transcript-api && python -c "from youtube_transcript_api import YouTubeTranscriptApi; print(YouTubeTranscriptApi.get_transcript('VIDEO_ID'))"` |
| B站 | `curl` API | `curl "https://api.bilibili.com/x/web-interface/view?bvid=BVxxx"` 获取 cid，再拉字幕 |

**流程：**

1. 从 URL 中提取视频 ID（B站的 `BVxxx` 或 YouTube 的 `v=xxx`）
2. 拉取字幕文本（优先中文，降级英文）
3. AI 摘要：2-3 句话概括核心内容
4. 执行通用保存步骤（创建笔记 → 挂知识库）
5. 返回视频快照

**输出示例：**

```
📥 正在保存到「公众号文章知识库」…

🎬 视频快照
标题：为什么 AI 视频生成在 2026 年爆发？
来源：B站 · up主: xxx
字幕摘要：从算力成本下降、开源模型成熟、商业场景验证三个角度
          分析了 AI 视频生成爆发的底层逻辑。
          关键数据：2026 Q1 视频生成 API 调用量增长 20 倍。

✅ 笔记已保存 → 在知识库中搜索「为什么 AI 视频生成在 2026 年爆发 - 选题笔记」即可查看
```

**如果拉取失败**（视频无人上传字幕）：降级到层级二。

### 层级二：无公开字幕但有音频（通用）

视频平台没有提供字幕接口时，下载音频 → Whisper 语音转文字 → AI 摘要。

**流程：**

1. 用 `yt-dlp` 提取音频：`yt-dlp -x --audio-format mp3 -o "temp_audio.%(ext)s" "URL"`
2. 用 Whisper 本地转录为文字
3. AI 摘要 2-3 句话
4. 执行通用保存步骤（创建笔记 → 挂知识库）
5. 清理临时音频文件：`rm -f temp_audio.mp3 temp_audio.*`
6. 返回视频快照

**⚠️ 注意事项：**
- 此过程较慢（音频下载 + 转录可能需要 1-5 分钟）
- 部分平台（抖音、视频号等）`yt-dlp` 可能无法下载
- 下载失败时告知用户，降级到层级三
- 完成后务必清理临时文件

### 层级三：降级方案 — 页面信息快照

当字幕获取和音频下载都不可行时，退回到视频页面本身可见的信息做快照。

**信息来源：** 页面标题、视频简介、播放量、发布时间、评论区高赞内容

**流程：**

1. 用 WebFetch 或 curl 抓取视频页面内容
2. 提取标题、简介、播放数据、评论区亮点
3. AI 基于这些信息生成摘要
4. 执行通用保存步骤（创建笔记 → 挂知识库）
5. 返回视频快照

**输出示例：**

```
📥 正在保存到「公众号文章知识库」…

🎬 视频快照（页面信息）
标题：2026 最新 AI 工具测评
来源：抖音
数据：播放 32万 · 28分钟前发布
摘要：从视频简介和评论区来看，这是一个多款 AI 视频工具的横向对比测评，
      评论区讨论集中在 xx 工具和 xx 工具的效果对比。

✅ 笔记已保存 → 在知识库中搜索「2026 最新 AI 工具测评 - 选题笔记」即可查看

💡 当前为页面信息摘要，完整内容建议点开观看
```

### 层级决策总结

```
收到视频链接
     │
     ├─ B站/YouTube → 层级一（字幕拉取）
     │     └─ 失败 → 层级二（音频转录）
     │
     ├─ 其他平台 → 层级二（音频转录）
     │     └─ 失败 → 层级三（页面快照）
     │
     └─ 纯.mp4文件 → 层级三（页面快照，同步告知用户
                     建议用 IMA 桌面客户端直接导入文件）
     │
     └─ 摘要生成后 → 通用保存步骤
                      ├─ import_doc → 创建笔记（摘要永久保存）
                      └─ add_knowledge → 笔记挂到知识库
```

**用户体验规则：**
- 不给用户讲技术细节，不要说"正在用 yt-dlp 下载""正在调用 Whisper"
- 层级一：直接输出快照
- 层级二：先提示"视频较长，正在处理音频…"，完成后输出
- 层级三：直接输出快照
- 输出末尾提示笔记已保存，给出搜索关键词方便定位
- **笔记内容示例：**

```
## 为什么 AI 视频生成在 2026 年爆发？

**来源：** https://www.bilibili.com/video/BVxxx
**平台：** B站
**发布时间：** 2026-06-28

### AI 摘要
从算力成本下降、开源模型成熟、商业场景验证三个角度分析了 AI 视频生成爆发的底层逻辑。
关键数据：2026 Q1 视频生成 API 调用量增长 20 倍。

---
*由 pin2ima 于 2026-07-01 自动收录*
```

---

## ⚙️ Configuration

| Setting | Default | Override |
|---------|---------|----------|
| Target knowledge base name | "选题知识库" | User specifies in conversation |
| Credentials file location | `~/.config/ima/` | Via environment variables |

---

## 🐛 Error Handling

| Scenario | Detected by | Action |
|----------|-------------|--------|
| Credentials not configured | `test -f` check fails | Guide user to https://ima.qq.com/agent-interface, STOP |
| Knowledge base not found | `info_list` is empty | Inform user, list available KBs, ask which to use |
| API returns error | `code≠0` | Display `msg` to user |
| Network / curl error | Empty response or non-200 | Show error, suggest retry |

---

## 📁 Skill Structure

```
pin2ima/
├── SKILL.md              ← This file
├── meta.json             ← Version metadata
```

**零外部依赖** — 没有任何脚本文件，所有操作通过标准的 `curl` HTTP 请求完成。
`curl` 在所有主流操作系统上默认预装。唯一需要用户手动配置的是 IMA 凭证。
