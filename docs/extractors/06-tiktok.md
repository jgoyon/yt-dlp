# TikTok Extractor

Mobile app API emulation and device spoofing techniques.

## Overview

**File**: `yt_dlp/extractor/tiktok.py`
**Complexity**: Complex
**Key Technique**: Mobile app emulation to bypass web restrictions

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/tiktok.py)

---

## Device ID Generation

TikTok's mobile API requires device identification:

```python
def _generate_device_id(self):
    """Generate fake Android device ID"""
    return str(random.randint(
        7250000000000000000,
        7351147085025500000))
```

**Why random range?** Matches real Android device ID format.

---

## Mobile App Emulation

```python
def _call_mobile_api(self, video_id):
    device_id = self._generate_device_id()

    # Mimic TikTok Android app
    headers = {
        'User-Agent': 'com.zhiliaoapp.musically/2022600040 (Linux; U; Android 11; en_US; Pixel 5; Build/RP1A; Cronet/58.0.2991.0)',
    }

    query = {
        'device_id': device_id,
        'iid': device_id,  # Installation ID (same as device ID)
        'aid': '1233',     # App ID for TikTok
        'app_name': 'musical_ly',
        'version_code': '260604',
        'version_name': '26.6.4',
    }

    return self._download_json(
        'https://api16-normal-c-useast1a.tiktokv.com/aweme/v1/feed/',
        video_id,
        headers=headers,
        query=query)
```

**Parameters explained**:
- `device_id`: Identifies "device"
- `iid`: Installation ID
- `aid`: TikTok app identifier
- `version_code`/`version_name`: App version

---

## Why Mobile API Works Better

**Web API limitations**:
- More bot detection
- Lower quality videos
- Rate limiting
- Frequent captchas

**Mobile API advantages**:
- Better format availability
- Higher quality streams
- Less restrictive
- More stable

**Trade-off**: Must maintain app version info as TikTok updates.

---

## Hybrid Extraction Strategy

```python
def _real_extract(self, url):
    video_id = self._match_id(url)

    # Try mobile API first
    try:
        return self._extract_mobile_api(video_id)
    except (ExtractorError, KeyError):
        pass

    # Fall back to web scraping
    try:
        return self._extract_from_webpage(video_id)
    except ExtractorError:
        pass

    # Last resort: SIGI state extraction
    return self._extract_sigi_state(video_id)
```

**Three-tier approach**:
1. Mobile API (best quality, may fail)
2. Web scraping (moderate quality)
3. SIGI state (JavaScript state object)

---

## SIGI State Extraction

TikTok embeds data in JavaScript:

```python
def _extract_sigi_state(self, webpage):
    """Extract data from SIGI_STATE JavaScript object"""
    sigi_data = self._search_regex(
        r'<script[^>]*>\s*window\[.SIGI_STATE.\]\s*=\s*({.+?})\s*</script>',
        webpage, 'sigi state')

    data = json.loads(sigi_data)

    # Navigate nested structure
    video_data = traverse_obj(data, (
        'ItemModule', video_id, 'video'))

    return {
        'url': video_data.get('downloadAddr'),
        'width': video_data.get('width'),
        'height': video_data.get('height'),
    }
```

**SIGI_STATE**: JavaScript object containing video metadata embedded in page.

---

## User-Agent Importance

```python
# Desktop user-agent (web scraping)
desktop_ua = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Mobile user-agent (mobile API)
mobile_ua = 'com.zhiliaoapp.musically/2022600040 (Linux; Android 11)'
```

**TikTok checks User-Agent** to determine:
- Which API endpoint to use
- What quality to serve
- Whether to apply bot detection

---

## Watermark Handling

TikTok videos have watermarks:

```python
# Some formats include watermark
formats.append({
    'url': video_url,
    'format_id': 'download-watermark',
    'format_note': 'Watermarked',
    'preference': -1,  # Lower preference
})

# Try to find watermark-free URL
if 'watermark' not in video_url:
    formats.append({
        'url': video_url,
        'format_id': 'download-nowatermark',
        'format_note': 'No watermark',
        'preference': 10,  # Higher preference
    })
```

---

## Key Takeaways

1. **Device emulation** - Generate realistic device IDs
2. **Mobile app mimicking** - Better than web scraping
3. **User-Agent spoofing** - Critical for API access
4. **Multi-tier fallback** - Try multiple methods
5. **JavaScript state extraction** - Parse embedded data objects

**Applicable to**: Instagram, Snapchat, other mobile-first platforms

**Ethical note**: Respect platform ToS, use for personal content only.

---

[← Twitch](04-twitch.md) | [Next: Vimeo →](07-vimeo.md)
