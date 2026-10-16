---
name: xhs-hot-trend-poster
description: "Use when the user asks to search XHS keywords, analyze trends, and publish a note. Searches top 30 notes, summarizes hot topics, generates copy + card image, confirms with user, then publishes."
version: 1.1.0
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
2. **PC cookie required** (`~/.config/aione/xhs/default.json`) — used for both search AND publish. A fresh PC cookie with `web_session` can call Creator APIs directly (see Pitfall #20).
3. **Creator cookie optional** (`~/.config/aione/xhs/creator.json`) — fallback if PC cookie can't publish. Creator QR login is more prone to HTTP 406 IP blocks.
4. Cookie validity check (must do at session start):
   ```python
   import sys, os, json
   sys.path.insert(0, os.path.expanduser("~/.local/share/aione/upstreams/Spider_XHS/apis"))
   sys.path.insert(0, os.path.expanduser("~/.local/share/aione/upstreams/Spider_XHS"))
   from xhs_pc_apis import XHS_Apis
   from xhs_utils.xhs_pc.auth import XHSPcAuth
   with open(os.path.expanduser("~/.config/aione/xhs/default.json")) as f:
       cookie = json.load(f)["cookie"]
   if "web_session" not in cookie:
       print("PC cookie EXPIRED — needs QR re-login (see Cookie Management section)")
   else:
       auth = XHSPcAuth.from_cookie(cookie)
       apis = XHS_Apis(auth)
       apis.bootstrap()
       success, msg, result = apis.search_note(query="测试", page=1)
       print(f"PC cookie valid: {success}")
   ```
5. Dependencies: opencv-python, numpy, qrcode, Pillow, crypto-js (npm), fonts-noto-cjk, fonts-noto-color-emoji

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

1. **Cookie priority: PC cookie first, Creator cookie fallback** — A fresh PC cookie with `web_session` can call Creator APIs directly (see Pitfall #20). Only fall back to `~/.config/aione/xhs/creator.json` if PC cookie publish fails.
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

# Try PC cookie first, fall back to Creator cookie
cookie_path = os.path.expanduser("~/.config/aione/xhs/default.json")
if not os.path.exists(cookie_path):
    cookie_path = os.path.expanduser("~/.config/aione/xhs/creator.json")
with open(cookie_path) as f:
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
# If PC cookie fails (code=-1), retry with Creator cookie if available
if not success:
    creator_path = os.path.expanduser("~/.config/aione/xhs/creator.json")
    if os.path.exists(creator_path) and cookie_path != creator_path:
        with open(creator_path) as f:
            cookie = json.load(f)["cookie"]
        auth = XHSCreatorAuth.from_cookie(cookie)
        apis = XHS_Creator_Apis(auth)
        success, msg, res = apis.post_note(note_info)
```

Completion criteria: success=True, share_link returned, show link to user.

## Cookie Management & Re-login

### Cookie priority: PC-first (IMPORTANT)

PC cookie (`~/.config/aione/xhs/default.json`) is the **primary cookie for everything**: search, analysis, AND publishing. A fresh PC cookie with `web_session` can call Creator APIs (`get_all_posted_notes`, `post_note`) directly — see Pitfall #20. The Creator cookie (`~/.config/aione/xhs/creator.json`) is a **fallback only**.

This means: in most sessions, you only need to scan the PC QR code. Skip the Creator QR login entirely unless PC cookie publish fails.

### Quick validity check
Check if `web_session` is present in the cookie string. If missing, the cookie is expired/stale and needs refresh:
```python
import json, os
for name, path in [("PC", "~/.config/aione/xhs/default.json"), ("Creator", "~/.config/aione/xhs/creator.json")]:
    p = os.path.expanduser(path)
    if os.path.exists(p):
        cookie = json.load(open(p))["cookie"]
        expired = "web_session" not in cookie
        print(f"{name} cookie: {'EXPIRED' if expired else 'valid'} ({len(cookie)} chars)")
    else:
        print(f"{name} cookie: NOT FOUND")
```
For a definitive check, run a search with PC cookie (see Prerequisites step 4).

### QR login scripts

**PC login (run this FIRST — most reliable):**
`/home/ubuntu/.hermes/venvs/aione/bin/python3 /home/ubuntu/xhs_qr_login.py`
- Saves QR to `/home/ubuntu/xhs_qrcode.png`
- Polls 180s for scan
- On success, outputs JSON with cookie fields to stdout

**Creator login (only if PC cookie can't publish):**
`/home/ubuntu/.hermes/venvs/aione/bin/python3 /home/ubuntu/xhs_creator_qr_login.py`
- Saves QR to `/home/ubuntu/xhs_creator_qrcode.png`
- More prone to HTTP 406 IP blocks (see below)

Run them as background processes. Both can run simultaneously to save time, but usually PC-only is enough.

### Saving cookie after QR login

The login scripts output `{"cookies": {...}}` to stdout. You must save this to the aione cookie file:

```python
import json, os
# Parse the script's stdout JSON, then build a semicolon-joined cookie string
# For PC login:
raw = {"abRequestId": "...", "web_session": "...", ...}  # from script output
cookie_str = "; ".join(f"{k}={v}" for k, v in raw.items())
path = os.path.expanduser("~/.config/aione/xhs/default.json")
os.makedirs(os.path.dirname(path), exist_ok=True)
json.dump({"cookie": cookie_str}, open(path, "w"), ensure_ascii=False)
print(f"Saved to {path}")
```

### Creator QR login HTTP 406
The Creator QR endpoint (`creator.xiaohongshu.com`) is more aggressively IP-controlled than the PC endpoint. **PC QR login can succeed while Creator QR login fails with HTTP 406 simultaneously.** This is server-side IP risk control, not a code bug. Retrying immediately usually fails again. Options:
- **Use PC cookie for publishing instead** — most of the time this works (see Pitfall #20).
- Wait several hours and retry (IP cooldown).
- Publish manually from the app if the note is urgent (hand off title + desc + image paths).
- If only PC cookie is fresh, search/analysis steps can proceed; publishing can also use PC cookie.

## Common Pitfalls

See also: `references/xhs-publish-pitfalls.md` for full session-verified details, and `references/xhs-image-fields.md` for XHS search result image URL structure (for analysis/reference only — do NOT re-upload XHS CDN images).

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

18. **Publish rate limiting (code=-1 on ALL retries)** — If you've published 3+ notes in the same session (same day), subsequent `post_note` calls will return `code=-1` on all 3 retries even with valid cookies and small images. This is NOT a cookie expiry issue (you can still call `get_all_posted_notes` successfully). It's account-level rate limiting on the creator publishing endpoint. The fix is to WAIT several hours before retrying. Do not keep hammering the API — each retry attempt may extend the cooldown. If the user needs the note published immediately, give them the title + desc + image paths and let them publish manually from the app.

19. **XHS-internal images cause publish rejection** — Downloading images from XHS search results (via `cover.url_default` or `image_list[].info_list[].url`) and re-uploading them as card backgrounds triggers `code=-1` rejection. XHS likely fingerprints its own CDN images and blocks re-upload. This applies even if the image is embedded as base64 in an HTML card and rendered to PNG. Workaround: use CSS-only gradients for aesthetic cards, or use Unsplash/external stock photos (not from XHS CDN). The `cover.url_default` field structure is documented in `references/xhs-image-fields.md`.

18. **Topic-appropriate stock imagery** — When downloading stock photos for card backgrounds, the imagery MUST match the topic's aesthetic. Generic landscape photos for a dreamcore/梦核 topic is a hard reject ("素材图和梦核没啥关系"). For aesthetic topics (梦核, dreamcore, liminal space), search for: empty rooms, staircases, corridors, old balconies, windows, pools — the imagery IS the aesthetic, not decoration. Then apply CSS filters (desaturate, darken, blur, sepia) to unify the mood. For travel topics, search for actual city landmarks (Tokyo street, Mt Fuji, food). Always verify downloaded image > 1KB; failed Unsplash downloads return 29 bytes.

20. **PC cookie can call Creator APIs directly (IMPORTANT)** — A fresh PC cookie (from `~/.config/aione/xhs/default.json`) with `web_session` can initialize `XHS_Creator_Apis` and call `get_all_posted_notes()` / `post_note()` directly. The API may return HTTP 406 in the status field, but the response body still contains valid data (notes list, publish results). This means:
    - You usually DON'T need a separate Creator QR login — PC QR login is enough for both search and publish.
    - Creator QR endpoint is MORE prone to 406 IP blocks than PC QR endpoint.
    - In Step 5 (Publish), use PC cookie first; only fall back to Creator cookie if PC cookie publish fails.
    - Verified: `get_all_posted_notes()` with PC cookie returned HTTP 406 status but body included full notes list with titles, images, dates.
    - This was discovered in a session where Creator QR login failed 3x with 406 but PC cookie worked perfectly for both search and publish.

21. **Cookie auto-save after QR login** — The QR login scripts output cookie JSON to stdout but do NOT auto-save to `~/.config/aione/xhs/default.json`. You must manually parse the stdout JSON, build a semicolon-joined cookie string, and write `{"cookie": "k=v; k2=v2; ..."}` to the file. Forgetting this step leaves the old expired cookie in place and all subsequent API calls will fail silently. See "Saving cookie after QR login" in the Cookie Management section.

22. **`aione auth xhs status` only checks file existence** — The command reports `configured: true` as long as the cookie file exists, even if the cookie inside is expired. Do NOT rely on it for validity checking. Always do the API-level check (search_note with "测试") described in Prerequisites step 4.

## Verification Checklist

- [ ] ~/.hermes/notes/xhs.md read at session start
- [ ] PC cookie validity checked (web_session present + search_note test passed)
- [ ] 30 notes fetched (or as many as available if fewer)
- [ ] Notes sorted by engagement (likes + collects)
- [ ] Hot topic summary drafted
- [ ] Title ≤ 20 chars
- [ ] Desc includes hook + key points + hashtags
- [ ] 3-5 hashtags in #xxx[话题]# format
- [ ] Card image generated (1080x1440, > 50KB)
- [ ] User shown full draft + asked to confirm
- [ ] PC cookie used for publish (Creator cookie as fallback only)
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

## Poster Mode: Aesthetic/Editorial Cards (NOT PPT-style)

**Trigger:** When the topic is aesthetic, mood-based, or lifestyle (梦核, dreamcore, liminal space, aesthetic styles, 氛围感, 情绪板) rather than information-dense (travel guides, checklists, how-to).

**User feedback that triggered this mode:** "太像ppt了" (too much like PPT) + "素材图和梦核没啥关系" (stock photos don't match the dreamcore aesthetic) + "重新设计配图版面，要有设计感，不要和之前一样那么死板"

**Poster Mode principles (DIFFERENT from standard card mode):**
- **Full-screen background image** — the image IS the card, not a decoration on top of a card template
- **Minimal text: 20-30 chars per card** — the user explicitly said "一张图字可以少但是字体一定要大"
- **Giant typography: 90-160px** for main text, 36-42px for descriptions
- **No card boxes, no item lists, no bullet points** — text floats directly over the image
- **Every card has a UNIQUE layout** — do NOT repeat the same layout across cards. One centers text, another puts it bottom-left, another splits left-image/right-text
- **CSS filters on background images** — `filter: contrast(1.2) brightness(0.6) saturate(0.3) sepia(0.4) blur(4px)` etc. to create the aesthetic mood
- **Gradient overlays** — `linear-gradient(180deg, rgba(dark,0.3), rgba(dark,0.7))` to ensure text readability
- **Serif fonts for editorial feel** — `font-family: 'Noto Serif SC', serif` for titles, sans-serif for body
- **Text fills the frame** — large fonts + padding to fill 1080x1440, not tiny text in a corner

**Topic-appropriate imagery is CRITICAL:**
- For 梦核/dreamcore/liminal space: empty rooms, staircases, corridors, old balconies, windows — NOT random landscape photos
- For aesthetic topics: search Unsplash with topic-specific keywords (empty room, corridor, staircase, window, balcony)
- Always verify downloaded image > 1KB (failed downloads return 29 bytes)
- Apply heavy CSS filters (desaturate, darken, hue-rotate) to make stock photos match the aesthetic mood

**Poster Mode layout examples (each card different):**
1. Full-screen image + centered mega text (160px) + English subtitle
2. Image left 45% + text right 55% split, text bottom-aligned
3. Blurred full-screen image + centered text + divider line
4. Full-screen image + text bottom-left, gradient from right
5. Full-screen image with sepia filter + text bottom, warm tones
6. Blurred dark image + centered question text + CTA

**When to use Poster Mode vs Standard Card Mode:**
- Poster Mode: 梦核, aesthetics, mood boards, 氛围感, emotional topics, art styles
- Standard Card Mode (layouts A/B/C): travel guides, food lists, checklists, how-to, comparison, step-by-step

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
