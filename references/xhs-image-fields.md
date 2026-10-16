# XHS Search Result Image Fields

When extracting images from XHS search results (for reference/analysis purposes), the field structure is:

## Cover Image

```json
{
  "cover": {
    "width": 1440,
    "height": 1920,
    "url_default": "http://sns-webpic-qc.xhscdn.com/.../nc_n_webp_mw_1",
    "url_pre": "http://sns-webpic-qc.xhscdn.com/.../nc_n_webp_prv_1"
  }
}
```

- `url_default`: full-size webp image
- `url_pre`: preview/thumbnail version
- URLs are HTTP — convert to HTTPS for downloading
- CDN domain: `sns-webpic-qc.xhscdn.com`

## Image List (multi-image notes)

```json
{
  "image_list": [
    {
      "width": 1440,
      "height": 1920,
      "info_list": [
        {"url": "http://...", "image_scene": "WB_DFT"},
        {"url": "http://...", "image_scene": "WB_PRV"}
      ]
    }
  ]
}
```

- `WB_DFT` = default image
- `WB_PRV` = preview image
- Each image has multiple resolutions/scenes

## Downloading Images

```python
import requests
headers = {
    "User-Agent": "Mozilla/5.0 ...",
    "Referer": "https://www.xiaohongshu.com/",
}
r = requests.get(url, headers=headers, timeout=15)
```

**IMPORTANT**: The Referer header is required — without it, XHS CDN may return 403.

## Caveat: Do NOT Re-upload XHS Images

Downloading XHS CDN images and re-uploading them as note images (even embedded in rendered HTML cards) triggers `code=-1` publish rejection. XHS fingerprints its own CDN images. See SKILL.md pitfall #19.

These images are useful for:
- Reference/analysis during content planning
- Showing the user what aesthetic to aim for
- Understanding what visual styles are trending

But NOT for:
- Using as backgrounds in cards you plan to publish
- Re-uploading in any form
