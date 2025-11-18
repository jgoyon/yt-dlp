# HotStar Extractor

DRM/Widevine support and HMAC authentication.

## Overview

**File**: `yt_dlp/extractor/hotstar.py`
**Complexity**: Medium-Complex
**Key Techniques**: Widevine DRM indication, HMAC auth, JWT decoding

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/hotstar.py)

---

## HMAC-Based API Authentication

HotStar uses HMAC for API request signing:

```python
import hmac
import hashlib

# Hardcoded encryption key (reverse-engineered)
_AKAMAI_ENCRYPTION_KEY = b'\x05\xfc\x1a\x01\xca\xc9\x4b\xc4\x12\xfc\x53\x12\x07\x75\xf9\xee'

def _sign_request(self, url):
    """Sign API request with HMAC"""
    # Create HMAC signature
    st = int(time.time())
    exp = st + 6000
    auth = f'st={st}~exp={exp}~acl=/*'

    signature = hmac.new(
        self._AKAMAI_ENCRYPTION_KEY,
        auth.encode(),
        hashlib.sha256
    ).hexdigest()

    return f'{url}?{auth}~hmac={signature}'
```

**Why HMAC?**
- Prevents API abuse
- Time-limited requests (expiry)
- Signature validation on server

---

## JWT Token Decoding

Check subscription status from JWT:

```python
import jwt

def _check_subscription(self, token):
    """Decode JWT to check subscription status"""
    try:
        payload = jwt.decode(token, options={'verify_signature': False})

        subscription = payload.get('subscriptionStatus')
        if subscription != 'SUBSCRIBED':
            raise ExtractorError(
                'This content requires subscription',
                expected=True)

        return payload
    except jwt.DecodeError:
        raise ExtractorError('Invalid authentication token')
```

**JWT contains**:
- User subscription status
- Content access permissions
- Region information

---

## Widevine DRM Indication

```python
def _extract_formats(self, data, video_id):
    formats = []

    for source in data.get('sources', []):
        if source.get('type') == 'hls':
            formats.extend(self._extract_m3u8_formats(
                source['url'], video_id, 'mp4'))

        elif source.get('type') == 'dash':
            dash_formats = self._extract_mpd_formats(
                source['url'], video_id)

            # Mark DRM-protected formats
            for fmt in dash_formats:
                if 'drm' in source:
                    fmt['has_drm'] = True
                    fmt['format_note'] = 'DRM protected (Widevine)'

            formats.extend(dash_formats)

    return formats
```

**DRM handling**:
- Indicate which formats require Widevine
- User needs compatible player/browser
- yt-dlp cannot decrypt DRM content (legal reasons)

---

## Device Spoofing

```python
# Mimic Android TV for better access
_HEADERS = {
    'User-Agent': 'Hotstar;in.startv.hotstar/13.1.3 (Android/11)',
    'X-HS-Platform': 'androidtv',
    'X-HS-AppVersion': '13.1.3',
}

def _call_api(self, endpoint, video_id):
    return self._download_json(
        f'https://api.hotstar.com/o/v1/{endpoint}',
        video_id,
        headers=self._HEADERS)
```

**Why Android TV?**
- Better format availability
- Less restrictive DRM
- Higher quality streams

---

## Subscription-Aware Headers

```python
def _get_headers(self, is_premium=False):
    """Get headers based on content type"""
    headers = self._HEADERS.copy()

    if is_premium:
        # Premium content requires auth token
        if self._AUTH_TOKEN:
            headers['X-HS-UserToken'] = self._AUTH_TOKEN
        else:
            raise ExtractorError(
                'Premium content requires login',
                expected=True)

    return headers
```

---

## Key Takeaways

1. **HMAC signatures** - Time-limited API authentication
2. **JWT decoding** - Extract subscription info without validation
3. **DRM indication** - Mark formats requiring Widevine
4. **Device spoofing** - Android TV for better access
5. **Conditional headers** - Different auth for premium content

**Important**: DRM content cannot be decrypted by yt-dlp (legal/technical limits)

---

[← Vimeo](07-vimeo.md) | [Next: CDA →](09-cda.md)
