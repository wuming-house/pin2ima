# Pin2IMA

> Pin any link to your IMA knowledge base. AI-powered summaries, forever searchable.

把你的自媒体选题素材、调研资料、灵感来源——**无论来自哪个平台**——一键存进 IMA 知识库。文章链接全文可检索，视频链接自动提取字幕/音频生成摘要并保存为笔记。

## 🌟 Features

- **全平台覆盖** — 微信公众号、知乎、GitHub、B站、YouTube、抖音、头条、普通网页……一切公开 URL
- **双模式处理**：
  - 📄 **文章** — 直接导入知识库，全文可检索可问答
  - 🎬 **视频** — 自动提取字幕/音频，AI 生成摘要，以笔记形式永久存入知识库
- **智能图文笔记** — 视频摘要通过 IMA 笔记持久化，内容含来源链接 + AI 摘要，分类到「选题笔记」文件夹
- **三层视频处理引擎**（自动降级）：
  1. 有字幕 → 拉字幕做摘要（秒级）
  2. 无字幕 → 音频转文字做摘要（分钟级）
  3. 都不行 → 页面信息快照（兜底）
- **零运行时依赖** — 全程只使用 `curl`，无需 Node.js / Python / 任何第三方工具
- **AI Agent 原生适配** — 任何能执行 bash 命令的 AI Agent（WorkBuddy、Claude Code、Cursor 等）都能直接使用

## 🎯 Use Cases

| 场景 | 说明 |
|------|------|
| **自媒体选题收集** | 刷到好文章/好视频，一键存档，AI 帮你记下为什么值得收藏 |
| **竞品调研** | 批量保存竞品内容，集中分析选题方向和内容策略 |
| **热点追踪** | 定时收集热点文章，自动生成选题速览报告 |
| **个人知识管理** | 碎片化阅读 → 知识库归档，不丢不乱 |

## 🚀 Quick Start

### 1. 获取 IMA 凭证

前往 [IMA 开放平台](https://ima.qq.com/agent-interface) 获取你的 **Client ID** 和 **API Key**，并保存到本地：

```bash
mkdir -p ~/.config/ima
echo "你的ClientID" > ~/.config/ima/client_id
echo "你的APIKey" > ~/.config/ima/api_key
```

### 2. 安装 Skill

```bash
# 克隆项目
git clone https://github.com/你的用户名/pin2ima.git

# 如果你用的是 WorkBuddy / Claude Code 等 AI Agent
cp -r pin2ima/skill ~/.workbuddy/skills/pin2ima
```

### 3. 开始使用

对 AI Agent 说：

> "帮我把 https://example.com/article 保存到知识库"

或：

> "收藏这几个链接到知识库"
> https://link1.com
> https://link2.com

Agent 会自动处理凭证验证、知识库识别、URL 导入，并返回保存结果。

## 📖 Usage Guide

### 保存文章

```
你：https://mp.weixin.qq.com/s/xxx 保存到知识库
Agent：📥 正在保存到「公众号文章知识库」…
       ✅ 已添加到知识库「公众号文章知识库」✓
```

### 保存视频

```
你：https://www.bilibili.com/video/BVxxx 收藏到知识库
Agent：📥 正在处理视频…
       🎬 视频快照
       标题：为什么 AI 视频生成在 2026 年爆发？
       字幕摘要：...
       ✅ 笔记已保存 → 在知识库中搜索标题即可查看
```

### 带备注保存

```
你：https://example.com 存到知识库，这篇适合做 AI 工具号的选题参考
Agent：📥 正在保存…
       ✅ 已保存 + 笔记已记录你的备注
```

### 指定知识库

```
你：把 https://github.com/xxx 保存到「GitHub 项目知识库」
Agent：📥 正在保存到「GitHub 项目知识库」…
```

### 批量保存

```
你：收藏这几个链接
   https://link1.com
   https://link2.com
   https://link3.com
Agent：📥 正在保存 3 个链接…
       ✅ 3 个已保存到「公众号文章知识库」
```

## 📁 Project Structure

```
pin2ima/
├── README.md                 ← 项目说明（就是这个文件）
├── LICENSE                   ← 开源许可证
├── .gitignore
├── skill/
│   ├── SKILL.md              ← AI Agent 技能指令（核心）
│   └── meta.json             ← 技能元数据
└── examples/
    └── usage.md              ← 使用示例和场景演示
```

## 🧠 How It Works

```
你发送链接
    │
    ├─ 文章链接 → import_urls → URL 存入知识库 ✅
    │
    └─ 视频链接 → 字幕/音频/页面 → AI 摘要
                   │
                   ├─ import_doc → 创建笔记（摘要永久保存）
                   ├─ add_knowledge → 笔记挂到知识库
                   └─ import_urls → 保存 URL 入口
```

技术原理：通过调用 [IMA OpenAPI](https://ima.qq.com/agent-interface) 的三个接口完成全流程：
- `openapi/wiki/v1/import_urls` — 导入网页/文章 URL
- `openapi/note/v1/import_doc` — 创建笔记（视频摘要持久化）
- `openapi/wiki/v1/add_knowledge` — 笔记挂载到知识库

## 🔧 Compatibility

| AI Agent | 能否使用 | 备注 |
|----------|---------|------|
| WorkBuddy | ✅ | 原生支持，适用 skill/ 目录 |
| Claude Code | ✅ | 支持 bash + curl |
| Cursor | ✅ | 支持终端命令 |
| Windsurf | ✅ | 支持 bash |
| GitHub Copilot CLI | ✅ | 支持 shell 命令 |
| 任何支持 bash 的 Agent | ✅ | 唯一依赖：curl |

## 📝 License

MIT

---

**Created by [吾鳴](https://mp.weixin.qq.com/s/Gslh5H5D9hqHnHxk8GzLFQ)**
- 微信公众号：**蕪鳴**
- 微信：**wu_ming_2025**
- Built for 自媒体创作者 · Made with ❤️
