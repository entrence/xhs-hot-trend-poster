# XHS Publishing Pitfalls (Session-Verified)

## Cookie Management
- PC cookie (`~/.config/aione/xhs/default.json`) and Creator cookie (`~/.config/aione/xhs/creator.json`) are **separate sessions** with independent expiry.
- Quick validity check: `web_session` in cookie string. If missing → expired, needs re-login.
- QR login scripts: `/home/ubuntu/xhs_qr_login.py` (PC), `/home/ubuntu/xhs_creator_qr_login.py` (Creator). Run both in parallel.
- Both scripts output `{"cookies": {...}}` JSON on success. Save as `{"cookie": "k=v; k2=v2; ..."}` (semicolon-joined string) to respective JSON file.
- **Creator QR endpoint is more IP-controlled than PC.** PC login can succeed while Creator login fails with HTTP 406 simultaneously. Retrying immediately usually fails — wait hours or publish manually.

## Search
- `aione xhs note search` CLI has a bug (`TypeError: missing 'self'`). Call XHS_Apis directly via Python.
- `search_note()` returns tuple `(success: bool, msg: str, result: dict)`. Use `isinstance(result, dict)` then `result.get("data", {}).get("items", [])`. Do NOT use `len(result)` or `result[2]`.
- Run XHS Python scripts via terminal with aione venv: `/home/ubuntu/.hermes/venvs/aione/bin/python3 script.py`. execute_code uses Hermes's own venv which lacks curl_cffi.

## Publish
- `note_info["type"]` must be `0` (public). Upstream default `1` is private. Always set explicitly.
- Images must be passed as bytes: `with open(path, "rb") as f: img = f.read()` then `note_info["images"] = [img]`.
- `get_topic()` has a bug (KeyError 'msg'). Skip it: use `note_info["topics"] = []` and write `#tag[话题]#` directly in desc.
- `rap_fingerprint` path has a double-nesting bug. Pre-check and cp file:
  `cp ~/.local/share/aione/upstreams/Spider_XHS/xhs_utils/xhs_creator/js/rap_fingerprint_creator.json ~/.local/share/aione/upstreams/Spider_XHS/xhs_utils/xhs_creator/xhs_utils/xhs_creator/js/`
- Missing deps: `opencv-python` + `numpy` (pip in aione venv), `crypto-js` (npm in Spider_XHS dir), `export NODE_PATH=~/.local/share/aione/upstreams/Spider_XHS/node_modules` at runtime.

## Engagement Count Parsing
XHS returns string counts like "1.2万". Parse:
```python
def parse_count(s):
    s = str(s)
    if "万" in s:
        return int(float(s.replace("万","")) * 10000)
    try:
        return int(s)
    except:
        return 0
```

## Card Image Generation
- Use HTML + Chrome headless: `google-chrome --headless --disable-gpu --no-sandbox --screenshot=output.png --window-size=1080,1440 --default-background-color=00000000 file:///path/to/template.html`
- Requires: `fonts-noto-cjk` (Chinese), `fonts-noto-color-emoji` (emoji)
- NotoColorEmoji.ttf is a bitmap font — Pillow can't render it. Use Chrome headless for emoji support.
- Verify output: file size > 50KB means content rendered correctly.

### Overflow Prevention (hit twice in one session)
- **Estimate total height before rendering.** See SKILL.md pitfall #12 for the height budget formula.
- Cards with hero images (240-340px) should cap at **5 items**.
- Cards without hero can fit **6-7 short items** (1-2 lines each).
- **6 items × 3-line descriptions + hero image = guaranteed overflow.**
- When fixing overflow post-render, reduce in this order:
  1. Hero image height (340→240px) — biggest single win
  2. Item padding/margin (16px→10px, margin 10px→7px)
  3. Font sizes (reduce by 15-20%: 30px→27px, 25px→22px)
- Always check if the footer page number is visible after rendering.

### Multi-Layout Carousel Pattern
- **Rotate between 3 layouts** — same layout × N cards = monotonous (user corrected: "配图不要每次都这一种模板").
- Layout A = hero-image + item list; B = big-number + text; C = timeline. See `references/card-layouts.md` for full templates.
- Assignment pattern for 9-card guides: A→B→A→B→A→B→A→B→C
- 3-4 cards out of 9 should have real photos (Unsplash, embedded as base64)
- **Verify Unsplash downloads**: file > 1KB. Failed downloads return ~29 bytes.

### Content Quality
- Each card item needs **2-3 sentences** with specifics (prices, times, locations, tips). One-liners get sent back (user: "描述的太浅了").
- Use **oversized fonts** for visual memory (user: "通过放大某种字体来加强视觉记忆"): numbers 42px+, section titles 54px+.
- XHS cards viewed on phones — minimum readable: 21px body, 27px item titles, 48px+ main title.
