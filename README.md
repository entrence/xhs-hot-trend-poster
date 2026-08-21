# xhs-hot-trend-poster

A Hermes Agent skill that searches XHS (Xiaohongshu / 小红书) keywords, analyzes top 30 trending notes, auto-generates copy + card images, and publishes to XHS.

Hermes Agent 的一个 skill，用于搜索小红书关键词、分析 TOP 30 热门笔记趋势、自动生成文案和卡片图、并发布到小红书。

---

## Features / 功能

- Search XHS keywords (top 30 notes across 3 pages) / 搜索小红书关键词（3页共30条笔记）
- Trend analysis: hot topics, engagement patterns, content angles / 趋势分析：热门话题、互动数据、内容角度
- Auto-generate XHS-style title + description + hashtags / 自动生成小红书风格标题 + 正文 + 话题标签
- Generate 1080x1440 card images (single or multi-card carousel) via HTML + Chrome headless / 通过 HTML + Chrome 无头模式生成 1080x1440 卡片图（单图或多图轮播）
- One-click publish to XHS (public, with image attachments) / 一键发布到小红书（公开，带配图）

## Install / 安装

```bash
hermes skills install https://raw.githubusercontent.com/entrence/xhs-hot-trend-poster/main/SKILL.md
```

## Prerequisites / 前置依赖

This skill depends on the [All-IN-ONE](https://github.com/cv-cat/All-IN-ONE) toolchain (`aione` CLI) which provides XHS API access via [Spider_XHS](https://github.com/cv-cat/Spider_XHS).

本 skill 依赖 [All-IN-ONE](https://github.com/cv-cat/All-IN-ONE) 工具链（`aione` CLI），通过 [Spider_XHS](https://github.com/cv-cat/Spider_XHS) 提供小红书 API 访问能力。

### 1. aione CLI / aione 命令行工具

Install and set up the aione toolchain:

安装并初始化 aione 工具链：

```bash
# Install aione (see https://github.com/cv-cat/All-IN-ONE for details)
# 安装 aione（详见 https://github.com/cv-cat/All-IN-ONE）
aione setup
```

### 2. XHS Cookies / 小红书 Cookie

You need two XHS cookies configured:

需要配置两个小红书 Cookie：

```bash
# PC cookie (for search)
# PC cookie（用于搜索）
aione auth xhs set-cookie --cookie "YOUR_XHS_PC_COOKIE"

# Creator cookie (for publishing)
# Save as JSON: {"cookie": "YOUR_XHS_CREATOR_COOKIE"}
# Default path: ~/.config/aione/xhs/creator.json
# Creator cookie（用于发布）
# 保存为 JSON 格式：{"cookie": "YOUR_XHS_CREATOR_COOKIE"}
# 默认路径：~/.config/aione/xhs/creator.json
```

### 3. Python Dependencies / Python 依赖

Install in the aione venv:

在 aione 虚拟环境中安装：

```bash
# aione venv path (default: ~/.hermes/venvs/aione/)
# aione 虚拟环境路径（默认：~/.hermes/venvs/aione/）
pip install opencv-python numpy qrcode Pillow
```

### 4. Node.js Dependencies / Node.js 依赖

```bash
cd ~/.local/share/aione/upstreams/Spider_XHS
npm install crypto-js
```

### 5. System Packages / 系统包

```bash
# Chrome (for headless card image rendering)
# Chrome（用于无头模式渲染卡片图）
sudo apt install -y google-chrome-stable

# Fonts (Chinese + emoji)
# 字体（中文 + emoji 表情）
sudo apt install -y fonts-noto-cjk fonts-noto-color-emoji
```

### 6. Environment Variables / 环境变量

At runtime, set `NODE_PATH` so the XHS creator API can find `crypto-js`:

运行时需设置 `NODE_PATH`，让小红书 Creator API 能找到 `crypto-js`：

```bash
export NODE_PATH=~/.local/share/aione/upstreams/Spider_XHS/node_modules
```

## Usage / 使用方法

In your Hermes session, just say:

在 Hermes 会话中直接说：

```
搜索小红书 减脂餐 并分析
小红书搜索 日本旅游攻略 然后写笔记
小红书热点 独立女性
```

The skill will:

Skill 会自动完成：

1. Search XHS for the keyword (top 30 notes) / 搜索小红书关键词（TOP 30 笔记）
2. Analyze trends and engagement patterns / 分析趋势和互动数据
3. Draft title + description + hashtags / 拟定标题 + 正文 + 话题标签
4. Generate card image(s) (1080x1440) / 生成卡片图（1080x1440）
5. Show you the draft and ask for confirmation / 展示草稿并等待确认
6. Publish to XHS on your confirmation / 确认后发布到小红书

## Repository Structure / 仓库结构

```
xhs-hot-trend-poster/
├── SKILL.md                              # Skill main file (workflow + pitfalls) / Skill 主文件（工作流 + 踩坑记录）
├── README.md                             # This file / 本文件
├── references/
│   ├── card-layouts.md                  # 3 reusable card layout patterns (A/B/C) / 3种卡片布局模板
│   └── xhs-publish-pitfalls.md          # Session-verified publishing pitfalls / 实测验证的发布踩坑记录
└── templates/
    ├── card_template.html                # Basic card template / 基础卡片模板
    └── unified_card_template.html        # Unified carousel template (multi-card) / 统一轮播模板（多图）
```

## Notes / 注意事项

- XHS blocks datacenter IPs in browser (error 300012 "IP at risk"). Search must go through the Python API, not browser navigation. / 小红书会封锁数据中心 IP 的浏览器访问（报错 300012 "IP at risk"）。搜索必须走 Python API，不能直接浏览器访问。
- Published notes may be shadow-limited on sensitive topics (gender issues, 彩礼, etc.). Use neutral framing for such topics. / 敏感话题（性别议题、彩礼等）的笔记可能被限流（仅自己可见）。此类话题需使用温和措辞。
- Card images are 1080x1440px (XHS standard). Keep content within height budget to avoid clipping. / 卡片图为 1080x1440px（小红书标准尺寸）。内容需控制在高度预算内，避免底部截断。
- Font sizes should be large (title 54px+, body 27px+) since cards are viewed on phones. / 字体要大（标题 54px+，正文 27px+），因为卡片在手机上浏览。

## License

MIT
