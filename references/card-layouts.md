# XHS Card Layout System — 3 Reusable Patterns

Rotate between these 3 layouts when generating multi-card carousels. Using the same
layout for every card makes the carousel monotonous (user corrected: "配图不要每次都这一种模板").

All layouts target 1080×1440px (XHS standard card size). See SKILL.md pitfall #12
for the height budget formula — always estimate before rendering.

## Layout A: Hero Image + Item List

**Best for:** Topics with strong visual appeal (travel, food, fashion, scenic spots).
**Capacity:** 5 items max WITH hero image (340px), 6-7 items without hero.

```
┌──────────────────────────┐
│   [HERO IMAGE 240-340px]  │  ← Unsplash photo as background
│  Tag badge (top-left)     │     with dark gradient overlay
│  Title + subtitle (bottom)│
├──────────────────────────┤
│  [emoji] Title            │
│         Description (2ln) │  ← White card, rounded
│  [emoji] Title            │
│         Description (2ln) │
│  ... (5 max)              │
├──────────────────────────┤
│  Footer: tag    Page X/N  │
└──────────────────────────┘
```

Key CSS:
- Hero: `background-image: linear-gradient(to bottom, rgba(0,0,0,.1), rgba(0,0,0,.3)), url(hero.jpg); background-size: cover`
- Hero tag: absolute positioned top-left, accent background, white text
- Hero title: absolute positioned bottom, `font-size: 44-50px`, `text-shadow: 0 2px 12px rgba(0,0,0,.5)`
- Items: `padding: 10-16px 20-22px; margin-bottom: 7-10px`
- Item title: `font-size: 27-30px`, item desc: `font-size: 22-25px`

## Layout B: Big Number/Icon + Text

**Best for:** Lists, rankings, step-by-step guides, comparisons.
**Capacity:** 6 items (with long descriptions), 8 items (with short descriptions).
No hero image — full vertical space for content.

```
┌──────────────────────────┐
│  [decorative circle bg]   │
│  Tag                      │
│  TITLE (54px)             │
│  Subtitle                 │
│  ───                      │
│  [NUM] Title              │
│        Description (3ln)  │  ← White card, number/icon on left
│  [NUM] Title              │     Numbers use accent color, 36-42px
│        Description (3ln)  │
│  ... (6 max)              │
├──────────────────────────┤
│  Footer: tag    Page X/N  │
└──────────────────────────┘
```

Key CSS:
- Background: light gradient (no hero image)
- Decorative circles: `position: absolute; border-radius: 50%; background: {accent}11`
- Big number/icon: `font-size: 36-42px; font-weight: 900; color: {accent}; min-width: 52-60px`
- Item title: `font-size: 26-29px`, item desc: `font-size: 22-25px`
- Use emoji as the "big number" for non-numbered lists (🍽️, 🚇, etc.)

## Layout C: Timeline / Vertical Flow

**Best for:** Itineraries, chronological steps, cause-effect sequences, day plans.
**Capacity:** 5-6 timeline entries with 2-line descriptions.

```
┌──────────────────────────┐
│  Tag                      │
│  TITLE (54px)             │
│  Subtitle                 │
│  ───                      │
│  09:00  ●─ Title          │
│         │  Description    │  ← Time on left, vertical line with dot,
│  12:00  ●─ Title          │     card on right
│         │  Description    │
│  14:00  ●─ Title          │
│         │  Description    │
│  ...                      │
├──────────────────────────┤
│  Footer: tag    Page X/N  │
└──────────────────────────┘
```

Key CSS:
- Timeline row: `display: flex; gap: 16px; align-items: stretch`
- Time label: `font-size: 24px; font-weight: 700; color: {accent}; min-width: 130px; text-align: right`
- Vertical line: `width: 2px; background: {accent}30; position: relative`
- Dot: `width: 14px; height: 14px; border-radius: 50%; background: {accent}; position: absolute; top: 14px`
- Card: `flex: 1; background: rgba(255,255,255,.92); border-radius: 16px; padding: 14px 20px`

## Rendering Commands

Write all HTML files first, then render sequentially:
```bash
for i in 1 2 3 4 5 6 7 8 9; do
  google-chrome --headless --disable-gpu --no-sandbox \
    --screenshot=/tmp/card_${i}.png \
    --window-size=1080,1440 \
    --default-background-color=00000000 \
    file:///tmp/card_${i}.html 2>/dev/null
done
```

Do NOT use shell `&` — terminal tool rejects it. Run separate terminal calls for parallelism.

## Embedding Photos

Download from Unsplash (free), embed as base64 in HTML for Chrome rendering:
```python
import base64
with open("photo.jpg", "rb") as f:
    b64 = base64.b64encode(f.read()).decode()
img_css = f"url(data:image/jpeg;base64,{b64})"
```

**Verify download succeeded:** check file size > 1KB. Failed downloads return ~29 bytes.
If a URL fails, try a different Unsplash photo ID — not all IDs resolve.

## Oversized Text for Visual Memory ("重点字加大")

Beyond just using large fonts overall (pitfall #13 in SKILL.md), the user wants **specific key phrases visually POPPED** within each card item. Two CSS classes handle this:

- `.big` — for critical numbers/keywords: `font-size: 38px; color: #e63946; font-weight: 900` (vs normal title 32px). E.g. `<span class='big'>100日元一盘</span>` in a 32px title.
- `.hl` — for inline highlights in descriptions: `font-size: 29px; color: #e63946; font-weight: 700` (vs normal desc 27px). E.g. `<span class='hl'>免去了每次买票的麻烦</span>`.

Usage pattern: each card item should have 1-2 `.big` spans in the title and 2-3 `.hl` spans in the description. Don't overuse — if everything is big, nothing stands out. Pick the most actionable/number-heavy phrases to enlarge.

## Batch Card Generation with Python

When generating 9+ cards, write a Python generator script (not hand-write each HTML). Structure:

```python
cards = [
    {"layout": "a", "img": "tokyo", "emoji": "🛫", "tag_text": "出发前必办", "page": "1 / 9",
     "title": "...", "sub": "...",
     "items": [("🛂", "护照+签证", "desc with <span class='hl'>highlights</span>"), ...]},
    # ... more cards
]
renderers = {"a": layout_a, "b": layout_b, "c": layout_c}
for i, card in enumerate(cards):
    html = renderers[card["layout"]](card)
    with open(f"/tmp/jp_cards/jp_card_{i+1}.html", "w") as f:
        f.write(html)
```

Key benefits: all cards share consistent CSS (defined once in `BASE_CSS`), photos embedded as base64 in the generator, easy to tweak all cards by changing one CSS string. Save the generator script to the project dir (e.g. `/home/ubuntu/jp_travel/gen_cards.py`) for re-use.

## Assignment Pattern for 9-Card Travel Guide

| Card | Layout | Has Photo | Topic Example |
|------|--------|-----------|---------------|
| 1    | A      | ✅        | 出发前必办 |
| 2    | B      | ❌        | 交通出行 |
| 3    | A      | ✅        | 必吃美食 |
| 4    | B      | ❌        | 必去景点 |
| 5    | A      | ✅        | 免费打卡 |
| 6    | B      | ❌        | 避坑指南 |
| 7    | A      | ✅        | 购物攻略 |
| 8    | B      | ❌        | 住宿建议 |
| 9    | C      | ❌        | 一日路线 |

Pattern: A→B→A→B→A→B→A→B→C. Alternates visual rhythm, ends with timeline for summary.
