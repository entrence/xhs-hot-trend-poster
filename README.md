# xhs-hot-trend-poster

A Hermes Agent skill that searches XHS (Xiaohongshu / 小红书) keywords, analyzes top 30 trending notes, auto-generates copy + card images, and publishes to XHS.

## Features

- Search XHS keywords (top 30 notes across 3 pages)
- Trend analysis: hot topics, engagement patterns, content angles
- Auto-generate XHS-style title + description + hashtags
- Generate 1080x1440 card images (single or multi-card carousel) via HTML + Chrome headless
- One-click publish to XHS (public, with image attachments)

## Install

```bash
hermes skills install https://raw.githubusercontent.com/entrence/xhs-hot-trend-poster/main/SKILL.md
```

## Prerequisites

This skill depends on the [All-IN-ONE](https://github.com/cv-cat/All-IN-ONE) toolchain (`aione` CLI) which provides XHS API access via [Spider_XHS](https://github.com/cv-cat/Spider_XHS).

### 1. aione CLI

Install and set up the aione toolchain:

```bash
# Install aione (see https://github.com/cv-cat/All-IN-ONE for details)
aione setup
```

### 2. XHS Cookies

You need two XHS cookies configured:

```bash
# PC cookie (for search)
aione auth xhs set-cookie --cookie "YOUR_XHS_PC_COOKIE"

# Creator cookie (for publishing)
# Save as JSON: {"cookie": "YOUR_XHS_CREATOR_COOKIE"}
# Default path: ~/.config/aione/xhs/creator.json
```

### 3. Python Dependencies

Install in the aione venv:

```bash
# aione venv path (default: ~/.hermes/venvs/aione/)
pip install opencv-python numpy qrcode Pillow
```

### 4. Node.js Dependencies

```bash
cd ~/.local/share/aione/upstreams/Spider_XHS
npm install crypto-js
```

### 5. System Packages

```bash
# Chrome (for headless card image rendering)
sudo apt install -y google-chrome-stable

# Fonts (Chinese + emoji)
sudo apt install -y fonts-noto-cjk fonts-noto-color-emoji
```

### 6. Environment Variables

At runtime, set `NODE_PATH` so the XHS creator API can find `crypto-js`:

```bash
export NODE_PATH=~/.local/share/aione/upstreams/Spider_XHS/node_modules
```

## Usage

In your Hermes session, just say:

```
搜索小红书 减脂餐 并分析
小红书搜索 日本旅游攻略 然后写笔记
小红书热点 独立女性
```

The skill will:

1. Search XHS for the keyword (top 30 notes)
2. Analyze trends and engagement patterns
3. Draft title + description + hashtags
4. Generate card image(s) (1080x1440)
5. Show you the draft and ask for confirmation
6. Publish to XHS on your confirmation

## Repository Structure

```
xhs-hot-trend-poster/
├── SKILL.md                              # Skill main file (workflow + pitfalls)
├── README.md                             # This file
├── references/
│   ├── card-layouts.md                  # 3 reusable card layout patterns (A/B/C)
│   └── xhs-publish-pitfalls.md          # Session-verified publishing pitfalls
└── templates/
    ├── card_template.html                # Basic card template (Jinja-style placeholders)
    └── unified_card_template.html        # Unified carousel template (multi-card)
```

## Notes

- XHS blocks datacenter IPs in browser (error 300012 "IP at risk"). Search must go through the Python API, not browser navigation.
- Published notes may be shadow-limited on sensitive topics (gender issues, 彩礼, etc.). Use neutral framing for such topics.
- Card images are 1080x1440px (XHS standard). Keep content within height budget to avoid clipping.
- Font sizes should be large (title 54px+, body 27px+) since cards are viewed on phones.

## License

MIT
