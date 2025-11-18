# SoundCloud Extractor

Dynamic client ID extraction and API credential management.

## Overview

**File**: `yt_dlp/extractor/soundcloud.py`
**Complexity**: Medium
**Key Technique**: Dynamic client ID scraping from JavaScript
**Why Interesting**: Shows how to handle rotating API credentials

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/soundcloud.py)

---

## The Problem

SoundCloud requires a `client_id` for all API requests:
```
https://api-v2.soundcloud.com/tracks/123?client_id=YOUR_CLIENT_ID
```

**Challenge**: Client IDs change periodically and hardcoding fails.

**Solution**: Extract client ID dynamically from SoundCloud's JavaScript.

---

## Dynamic Client ID Extraction

```python
def _update_client_id(self):
    """Extract client ID from SoundCloud's JavaScript"""
    # Download SoundCloud homepage
    webpage = self._download_webpage('https://soundcloud.com', None)
    
    # Find script URLs
    for script_url in re.findall(r'<script[^>]+src="([^"]+)"', webpage):
        if not script_url.startswith('http'):
            script_url = f'https://soundcloud.com{script_url}'
        
        # Download script
        script = self._download_webpage(script_url, None, fatal=False)
        if not script:
            continue
        
        # Search for client ID pattern
        match = re.search(r'client_id\s*:\s*"([0-9a-zA-Z]{32})"', script)
        if match:
            self._CLIENT_ID = match.group(1)
            return True
    
    return False
```

**Process**:
1. Download SoundCloud homepage
2. Extract all `<script>` tag URLs
3. Download each JavaScript file
4. Search for `client_id:"..."`  pattern
5. Cache the found client ID

---

## Client ID Caching

```python
class SoundcloudBaseIE(InfoExtractor):
    _CLIENT_ID = None  # Class-level cache
    
    def _call_api(self, path, item_id, query=None):
        if not self._CLIENT_ID:
            self._update_client_id()
        
        url = f'{self._API_V2_BASE}{path}'
        return self._download_json(
            url, item_id,
            query={**query, 'client_id': self._CLIENT_ID})
```

**Benefits**:
- Extract once, use for all requests in session
- Automatic refresh on 401/403 errors
- No hardcoded credentials

---

## Automatic Client ID Rotation

```python
def _call_api(self, path, item_id, query=None):
    try:
        return self._download_json(url, item_id, query=query)
    except HTTPError as e:
        if e.status in (401, 403):
            # Client ID expired, refresh and retry
            self._update_client_id()
            return self._download_json(url, item_id, query=query)
        raise
```

**Error handling**:
- Catch 401 Unauthorized
- Automatically extract new client ID
- Retry request
- Transparent to user

---

## Key Takeaways

1. **Dynamic credential extraction** - Don't hardcode API keys
2. **Caching** - Extract once per session
3. **Auto-rotation** - Handle expiration gracefully
4. **JavaScript scraping** - Extract runtime values from scripts
5. **Retry logic** - Automatic recovery from auth failures

**Applicable to**: Any site with rotating API credentials (Vimeo, Dailymotion, etc.)

---

[← YouTube](02-youtube.md) | [Next: Generic →](12-generic.md)
