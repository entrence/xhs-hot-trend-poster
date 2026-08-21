---
name: xhs-hot-trend-poster
description: "Use when the user asks to search XHS keywords, analyze trends, and publish a note. Searches top 30 notes, summarizes hot topics, generates copy + card image, confirms with user, then publishes."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [xhs, xiaohongshu, trend-analysis, content-creation, auto-publish]
    related_skills: [all-in-one]
---

# XHS Hot Trend Poster

## Overview

Search XHS keywords, analyze top 30 results, extract hot topic trends, auto-generate copy + card image, confirm with user, then publish to XHS.

## When to Use

- User says "搜索小红书 XXX 并分析" or "小红书热点 XXX"
- User says "小红书搜索 XXX 然后写笔记/写一篇关于XXX的笔记"
- User wants XHS trend analysis + auto content creation + publishing
- User wants XHS content analysis as research for writing their own note (even if not publishing)
- Don't use for: simple XHS search only (use all-in-one skill directly)

## CRITICAL: Load This Skill BEFORE Browser Navigation

Do NOT attempt `browser_navigate` to `xiaohongshu.com` first — XHS blocks datacenter IPs with error code `300012 "IP at risk"`. The search must go through the aione venv Python API (see Step 1). If the user asks anything involving XHS search/analysis, load this skill immediately and go straight to the Python API approach. Trying the browser first wastes a round-trip and frustrates the user.

## Prerequisites

1. Read ~/.hermes/notes/xhs.md for account info, pitfalls, and script paths
2. Ensure aione pc + creator cookies are configured (check with `aione auth xhs status`)
3. Dependencies: opencv-python, numpy, qrcode, Pillow, crypto-js (npm), fonts-noto-cjk, fonts-noto-color-emoji

## Workflow

### Step 1: Search (fetch 30 notes)

Use Python to call XHS_Apis directly (aione CLI search has a bug):

```python
import sys, os, json
sys.path.insert(0, os.path.expanduser("~/.local/share/aione/upstreams/Spider_XHS/apis"))
sys.path.insert(0, os.path.expanduser("~/.local/share/aione/upstreams/Spider_XHS"))
from xhs_pc_apis import XHS_Apis
from xhs_utils.xhs_pc.auth import XHSPcAuth

# Read cookie
with open(os.path.expanduser("~/.config/aione/xhs/default.json")) as f:
    cookie = json.load(f)["cookie"]

auth = XHSPcAuth.from_cookie(cookie)
apis = XHS_Apis(auth)
apis.bootstrap()

# Search 3 pages x ~20 = ~60 results, take top 30
# IMPORTANT: search_note returns a tuple (success: bool, msg: str, result: dict)
# Do NOT use len(result) or result[2] — result is already the dict
all_items = []
for page in [1, 2, 3]:
    success, msg, result = apis.search_note(query=KEYWORD, page=page)
    if success and isinstance(result, dict):
        items = result.get("data", {}).get("items", [])
        all_items.extend([i for i in items if i.get("model_type") == "note"])
    if len(all_items) >= 30:
        break

# Extract: title, type(image/video), likes, collects, comments, shares, author, publish_time
notes = []
for item in all_items[:30]:
    card = item.get("note_card", {})
    interact = card.get("interact_info", {})
    notes.append({
        "title": card.get("display_title", ""),
        "type": card.get("type", ""),
        "author": card.get("user", {}).get("nickname", ""),
        "likes": interact.get("liked_count", "0"),
        "collects": interact.get("collected_count", "0"),
        "comments": interact.get("comment_count", "0"),
        "shares": interact.get("shared_count", "0"),
    })
```

Completion criteria: 30 notes extracted with title + engagement metrics.

### Step 2: Analyze trends

Analyze the 30 notes and produce:

1. **Hot topic summary** (2-3 sentences): What's trending? Common themes?
2. **Top 5 engagement patterns**: Which content types get most likes/saves?
3. **Content angle**: Pick one actionable angle for the note (e.g. "XX合集", "XX避坑指南", "XX懒人攻略")
4. **Suggested title**: ≤20 chars, catchy, emoji-friendly
5. **Suggested desc**: 200-400 chars, includes hook + key points + CTA + hashtags
6. **Suggested tags**: 3-5 relevant hashtags in #xxx[话题]# format

Sort notes by engagement (likes + collects) to find true hot content.
Use numeric comparison (parse string counts like "1.2万" → 12000).

Completion criteria: title, desc, tags all drafted. Present to user.

### Step 3: Generate card image(s)

**Single-card mode** (default, for trend-analysis notes): Generate 1 card image (1080x1440) using HTML + Chrome headless.

**Multi-card carousel mode** (when user says "每一个议题一个图" or wants a series): Generate N cards, one per topic/论点. Each card should have a distinct color palette to create visual variety across the carousel. Number them as "X / N" in the footer.

Steps for each card:
1. Write HTML file to /tmp/xhs_card_{N}.html using template at `templates/card_template.html` or custom HTML
2. Pick color palette based on topic mood (see PALETTES below). For multi-card, assign different palette per card.
3. Fill template: title, subtitle, key points with emoji, tags, page number ("X / N")
4. Render all images in parallel: `google-chrome --headless --disable-gpu --no-sandbox --screenshot=/tmp/xhs_card_{N}.png --window-size=1080,1440 --default-background-color=00000000 file:///tmp/xhs_card_{N}.html`
5. Verify each output exists and size > 50KB

**Multi-card design tips:**
- **UNIFIED design across all cards** — same color palette, same font sizes, same card structure, same footer style. The user corrected this once ("四张都重新调整一下，美观统一一些") — they want a cohesive carousel, not a patchwork of different palettes. Only use varied palettes if the user explicitly asks for it.
- **Rotate layouts for visual variety** — the user corrected this ("配图不要每次都这一种模板"). When generating 4+ cards, alternate between 2-3 layout patterns (see `references/card-layouts.md` for 3 reusable layouts: A=hero-image+list, B=big-number+text, C=timeline). Don't use the same single layout for all cards.
- **Oversized elements for visual memory** — "通过放大某种字体来加强视觉记忆". Use extra-large numbers (42px+), oversized section titles (54px+), or hero images to create focal points. Not every element needs to be big — pick ONE thing per card to enlarge.
- **Rich descriptions, not one-liners** — "描述的太浅了，每一条就寥寥几句，丰富一些". Each card item should have 2-3 sentences with specifics: prices, times, locations, tips, comparisons. One-liners feel lazy and the user will send it back.
- **Real photos when possible** — "必要情况可以在网上找一些配图". For topics with visual appeal (travel, food, fashion), embed Unsplash images as base64 in the HTML hero section. Check downloaded file size > 1KB (failed downloads return 29 bytes). Not every card needs a photo — 3-4 out of 9 is enough.
- Each card should be self-contained (a reader can understand one card alone)
- Card 1 = hook/定义, middle cards = 分论点, last card = 情感升华/CTA
- See `references/card-layouts.md` for the 3-layout system with full HTML/CSS templates

Color palettes (pick ONE and use across ALL cards in the carousel):
- warm: 暖色系 bg #fff8f0→#fff0e8, accent #ff6c5c→#ff82ab — definitions, warmth, encouragement
- warm-red: 珊瑚红系 bg #fdf2f8→#fef3f2, accent #d51717→#ff6c5c — travel guides, energetic topics
- fresh: 清新系 bg #f0fff8→#e8fff0, accent #4ec388→#5dd39e — comparisons, growth
- cute: 可爱系 bg #fff0f5→#ffe8f0, accent #ff82ab→#ffa3bc — lifestyle, soft topics
- cool: 冷色系 bg #f0f5ff→#e8f0ff, accent #4285f4→#5c9eff — professional, analytical
- yellow: 明亮系 bg #fffce8→#fff8d8, accent #ffc107→#ff8a65 — how-to, actionable steps
- dark: 深色系 bg #1a1528→#2d1f3d with #ff82ab/#fbbf24 text — emotional, 情感升华, call to action

Completion criteria: PNG file(s) generated, each verified > 50KB.

### Step 4: Present to user

**For single-card:** Show analysis summary, drafted title, desc, tags, card image path. Ask "是否发布?"

**For multi-card carousel:** Show analysis summary, drafted title, desc, tags, list all card image paths. Ask "是否发布?"

**Important: WeChat multi-image delivery**
When delivering multiple images to the user via WeChat, send images in SEPARATE messages or small batches (1-2 per message). Sending 4+ images in a single message will cause some images to be dropped — the user will only receive a subset. If the user reports missing images, re-send the missing ones individually.

Do NOT publish until user confirms.

### Step 5: Publish (on user confirmation)

Use XHS_Creator_Apis to publish. Critical pitfalls (see ~/.hermes/notes/xhs.md):

1. Read cookie: ~/.config/aione/xhs/creator.json
2. Image must be bytes: `with open(path, "rb") as f: img = f.read()`
3. Set NODE_PATH: `export NODE_PATH=~/.local/share/aione/upstreams/Spider_XHS/node_modules`
4. note_info["type"] = 0 (PUBLIC! default 1 is private!)
5. note_info["topics"] = [] (get_topic has bug, use #tag[话题]# in desc instead)
6. rap_fingerprint path fix must be in place (check file exists at xhs_utils/xhs_creator/xhs_utils/xhs_creator/js/rap_fingerprint_creator.json)

```python
import sys, os, json
sys.path.insert(0, os.path.expanduser("~/.local/share/aione/upstreams/Spider_XHS/apis"))
sys.path.insert(0, os.path.expanduser("~/.local/share/aione/upstreams/Spider_XHS"))
from xhs_creator_apis import XHS_Creator_Apis
from xhs_utils.xhs_creator import XHSCreatorAuth

with open(os.path.expanduser("~/.config/aione/xhs/creator.json")) as f:
    cookie = json.load(f)["cookie"]
auth = XHSCreatorAuth.from_cookie(cookie)
apis = XHS_Creator_Apis(auth)

with open("/tmp/xhs_card_output.png", "rb") as f:
    image_bytes = f.read()

note_info = {
    "title": TITLE,
    "desc": DESC,
    "images": [image_bytes],
    "media_type": "image",
    "topics": [],
    "type": 0,  # 0=public, 1=private(!)
}
success, msg, res = apis.post_note(note_info)
```

Completion criteria: success=True, share_link returned, show link to user.

## Common Pitfalls

See also: `references/xhs-publish-pitfalls.md` for full session-verified details.

0. **Browser-first reflex — DON'T** — XHS blocks datacenter IPs (error 300012 "IP at risk"). Always go straight to the aione Python API for search. Do NOT `browser_navigate` to xiaohongshu.com. The user corrected this once ("你用skill啊") — it's a round-trip waste.

1. **aione CLI search_note bug** — `TypeError: missing 'self'`. Fix: call XHS_Apis directly via Python, not aione CLI.

2. **search_note return type** — Returns tuple `(success: bool, msg: str, result: dict)`, NOT a list. Use `isinstance(result, dict)` check, then `result.get("data", {}).get("items", [])`. Do NOT use `len(result) >= 3` or `result[2]`.

3. **execute_code vs terminal venv** — execute_code runs in Hermes's own venv which lacks `curl_cffi` and other aione deps. Always run XHS Python scripts via terminal with the aione venv: `/home/ubuntu/.hermes/venvs/aione/bin/python3 script.py`.

4. **Type=1 is private** — Upstream default `noteInfo.get('type', 1)` returns 1 (private). Always set `"type": 0` for public notes.

5. **Image must be bytes** — `get_file_info()` calls `np.frombuffer(file)`, expects bytes not path string. Read file as `open(path, "rb").read()`.

6. **get_topic KeyError 'msg'** — API response lacks 'msg' field. Skip topics, write `#tag[话题]#` directly in desc.

7. **rap_fingerprint path bug** — Path joins `_STATIC_DIR/../xhs_utils/xhs_creator/js/` which double-nests. Pre-check and cp file if missing.

8. **Missing dependencies** — opencv-python + numpy (pip), crypto-js (npm), NODE_PATH env var. All go in aione venv: `/home/ubuntu/.hermes/venvs/aione/`.

9. **Engagement count parsing** — XHS returns string counts like "1.2万". Parse: `float(x.replace("万","")) * 10000 if "万" in x else int(x)`.

10. **WeChat multi-image delivery drop** — Sending 4+ images in a single message via WeChat causes some images to be silently dropped. The user only receives a subset and has to ask "剩下两张呢". Fix: send images in separate messages or batches of 1-2. If the user reports missing images, re-send individually.

11. **Multi-card carousel: Chrome rendering loop** — When generating N card images, write all HTML files first, then render them with a `for` loop: `for i in 1 2 3 4; do google-chrome --headless --disable-gpu --no-sandbox --screenshot=/tmp/card_${i}.png --window-size=1080,1440 --default-background-color=00000000 file:///tmp/card_${i}.html 2>/dev/null; done`. Do NOT use shell `&` backgrounding — the terminal tool rejects it with "Foreground command uses '&' backgrounding". Batch separate terminal calls for true parallelism.

12. **Card content overflow ("显示不全")** — 1080x1440px canvas fits ~5-6 items max. If a card has 7+ items, the bottom content (including footer) gets clipped. Prevention: cap at 6 items per card; if topic needs more, split into 2 cards or reduce each item to 1 line (title + short desc, no `<br>`). Test by checking if the footer page number is visible. If user reports "显示不全", compress: reduce font sizes by 15-20%, reduce padding/margin, shorten descriptions.

  **Height budget formula (learned the hard way, twice in one session)** — Before rendering, estimate total content height:
  - Hero image: 240-340px (layout A with photo)
  - Tag + title + sub + bar: ~180px
  - Per item: padding(2x ~20-32px) + title(~35px) + desc(N lines × 1.5 × font-size). E.g. 3-line desc at 25px ≈ 112px + 32px padding = ~144px per item.
  - Footer: 60px (incl bottom margin)
  - **Total must be < 1440px.** If estimate > 1440, either cut items, shrink fonts/padding, or reduce hero height.
  - **6 items with 3-line descriptions + hero image = overflow.** Either use 5 items, drop the hero, or compress to 2-line descriptions.
  - Cards with 7+ short items (1-2 lines each) can fit without a hero image. Cards with a hero image should cap at 5 items.
  - When patching overflow post-render: reduce hero height first (biggest single win), then item padding/margin, then font sizes.

13. **Use large fonts + fill the canvas ("字体大一些" / "尽量占满了")** — XHS cards are viewed on phones; the user wants big, readable text that FILLS the 1440px canvas. Recommended sizes: title 56-64px, item titles 29-34px, item descriptions 25-29px, quotes 26-30px, footer 24px. The user corrected this TWICE in one session — first asking for larger fonts, then asking to fill empty space at the bottom. When content doesn't fill 1440px, INCREASE font sizes and padding to fill the canvas, not add more items. Aim for the last card's content to reach near the footer.

14. **Batch work, don't trickle ("还没做完吗？")** — When the user asks for N cards, generate ALL of them in one pass — write all HTML files first, then render all PNGs. Don't pause partway to show intermediate results. Use `execute_code` to generate all N HTML files programmatically from a data array, then render all in one terminal call with a `for` loop. The user gets frustrated when progress seems stalled.

15. **Sensitive content gets shadow-limited ("别人看不到")** — XHS silently limits notes on sensitive topics (彩礼, 性别对立, etc.): the note publishes successfully (API returns success=True) and is visible to the author, but other users CANNOT see it in search or feed. The "被边缘拒绝(code=-1)" warning in the publish log is a signal this may happen. If the topic is socially sensitive (gender issues, 彩礼, politics), pre-emptively soften the angle: avoid aggressive/攻击性 language, use neutral framing ("结婚前该不该谈钱" instead of "彩礼不是卖女儿"), include both perspectives, and avoid inflammatory hashtags. If a note is already shadow-limited, tell the user to delete it manually — there is NO API to delete notes.

16. **Content tone: casual and life-like ("生活化一点别那么正式")** — The user explicitly rejects academic/formal language in XHS notes. Write like chatting with a friend: "人性就是这样的" not "心理学中的沉没成本效应", "这笔账该不该算" not "结构性不平等的补偿", "唠唠嗑" not "理性探讨", "说白了" not "核心结论". Keep arguments sharp but language accessible. Quotes from XHS commenters should sound like real comments, not edited quotes.

17. **No delete-note API exists** — XHS_Creator_Apis has no delete_note method. The creator web dashboard (creator.xiaohongshu.com) requires separate browser login (cookie injection doesn't work well). If a note needs deletion, tell the user to delete it manually in the XHS app: Profile → Notes → select → delete.

## Verification Checklist

- [ ] ~/.hermes/notes/xhs.md read at session start
- [ ] 30 notes fetched (or as many as available if fewer)
- [ ] Notes sorted by engagement (likes + collects)
- [ ] Hot topic summary drafted
- [ ] Title ≤ 20 chars
- [ ] Desc includes hook + key points + hashtags
- [ ] 3-5 hashtags in #xxx[话题]# format
- [ ] Card image generated (1080x1440, > 50KB)
- [ ] User shown full draft + asked to confirm
- [ ] type=0 (public) in note_info
- [ ] Image passed as bytes
- [ ] NODE_PATH exported
- [ ] rap_fingerprint path exists
- [ ] Publish success confirmed, share_link given to user

## Title Style: "炸裂" and Click-Driven

The user wants titles that are **explosive and click-worthy**, not safe or generic. They explicitly asked for "标题起的炸裂一点，爆一点" and rejected mild options. Characteristics of winning titles:

- **Overcome-loss framing**: "花了6万去日本踩的坑 这9张图帮你全避开" — implied stakes + actionable promise
- **FOMO + authority**: "听劝！第一次去日本的姐妹一定要刷到这篇" — urgency + target audience
- **Contrarian/真相**: "去了香港才知道之前看的攻略有多离谱" — challenges assumptions
- **Persona-based**: "回来的人劝你" / "过来人血泪" — experienced voice, not generic blogger

Losing patterns (user rejected these):
- Safe/descriptive: "日本旅游攻略｜9张图搞定吃喝玩乐买" — too generic
- Academic: "理性探讨彩礼的必要性" — too formal
- Mild questions: "结婚前该不该谈钱" — okay but not "炸裂"

When generating title options, provide 4-6 choices across different angles (FOMO, authority, contrarian, loss-aversion, persona). Let the user pick. Don't default to safe options.

## Multi-Keyword Search Strategy

When the user's topic spans multiple sub-topics (e.g. "日本旅游攻略" → food, transport, attractions, shopping, accommodation), search **5 related keywords** in one script run (e.g. "日本旅游攻略", "日本自由行", "东京美食", "大阪攻略", "日本购物"). Each keyword returns 8 top notes. Cross-reference across all 5 to build a comprehensive content map before designing the 9-card structure. This gives richer, more diverse content than searching one keyword alone.

## One-Shot Recipe

User: "搜索小红书 减脂餐 并分析"

1. read_file ~/.hermes/notes/xhs.md
2. Search "减脂餐" 3 pages, extract top 30 notes
3. Analyze: sort by engagement, find patterns (e.g. "食谱合集" gets most saves, "懒人" angle trending)
4. Draft: title="懒人减脂餐｜一周不重样", desc=hook+7天食谱+CTA+tags
5. Generate card: warm palette, 7 items (周一~周日), food emoji
6. Present analysis + draft + image path to user
7. Wait for "发布" confirmation
8. Publish with type=0, return share_link
