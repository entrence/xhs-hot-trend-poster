# XHS Hot Trend Poster

Search XHS (小红书/Xiaohongshu) keywords, analyze top 30 results, extract hot topic trends, auto-generate copy + card image, confirm with user, then publish to XHS.

## Features

- Search top 30 XHS notes by keyword (3 pages, ~60 results filtered)
- Engagement-based trend analysis (likes + collects sorting, "万" count parsing)
- Auto-generate note copy: title, description, hashtags
- Card image generation (1080x1440) via HTML + Chrome headless
- Multi-card carousel mode with unified design
- Poster mode for aesthetic/editorial topics (梦核, dreamcore, etc.)
- Direct publishing via XHS Creator API
- PC cookie-first strategy (no separate Creator login needed)

## Install

```bash
hermes skills install https://raw.githubusercontent.com/entrence/xhs-hot-trend-poster/main/SKILL.md
```

## Prerequisites

### Python packages (install in aione venv)
```bash
~/.hermes/venvs/aione/bin/pip install opencv-python numpy qrcode Pillow
```

### Node.js packages
```bash
cd ~/.local/share/aione/upstreams/Spider_XHS && npm install crypto-js
```

### System packages
```bash
sudo apt install fonts-noto-cjk fonts-noto-color-emoji google-chrome-stable
```

### Credentials
- XHS PC cookie: `~/.config/aione/xhs/default.json` (required — contains `web_session`)
- XHS Creator cookie: `~/.config/aione/xhs/creator.json` (optional fallback)
- Obtain via QR login scripts: `xhs_qr_login.py` (PC) / `xhs_creator_qr_login.py` (Creator)

### Environment variables
```bash
export NODE_PATH=~/.local/share/aione/upstreams/Spider_XHS/node_modules
```

### Runtime dependencies
- aione CLI + Spider_XHS upstream (`~/.local/share/aione/upstreams/Spider_XHS/`)
- aione Python venv: `~/.hermes/venvs/aione/bin/python3`
- google-chrome (headless) for card image rendering

## Usage

Say any of these to trigger the skill:
- "搜索小红书 减脂餐 并分析"
- "小红书热点 梦核"
- "小红书搜索 日本旅游 然后写笔记"
- "搜索小红书 XXX 然后分析趋势发个笔记"

The skill will: search → analyze → draft copy → generate card image → ask for confirmation → publish.

## Repository Structure

```
xhs-hot-trend-poster/
├── SKILL.md                           # Main skill definition
├── README.md                          # This file
├── references/
│   ├── card-layouts.md                # 3 reusable card layouts (A/B/C) with HTML/CSS
│   ├── xhs-image-fields.md            # XHS search result image URL structure
│   └── xhs-publish-pitfalls.md        # Full session-verified publish pitfalls
└── templates/
    ├── card_template.html             # Standard card HTML template
    └── unified_card_template.html     # Unified multi-card template
```

## Notes

- XHS blocks datacenter IPs for browser access — all search/publish goes through Python API
- PC cookie with `web_session` can call Creator APIs directly (no separate Creator login needed)
- Creator QR login endpoint is more prone to HTTP 406 IP blocks than PC endpoint
- Card images: 1080x1440px, max 5-6 items per card, large fonts to fill canvas
- Sensitive topics (彩礼, gender issues) may get shadow-limited — use neutral framing
- No delete-note API exists — manual deletion via XHS app only
- Publishing rate limit: 3+ notes/day triggers account-level cooldown

## License

MIT
