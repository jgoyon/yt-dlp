# Vimeo Extractor

OAuth token management, JWT validation, and multi-client architecture.

## Overview

**File**: `yt_dlp/extractor/vimeo.py`
**Complexity**: Medium
**Key Techniques**: OAuth 2.0, JWT tokens, multi-client API

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/vimeo.py)

---

## OAuth 2.0 Client Credentials

Vimeo uses OAuth for API authentication:

```python
# Different OAuth clients for different platforms
_OAUTH_CLIENTS = {
    'android': {
        'client_id': '90123456789abcdef01234567890',
        'client_secret': 'abcdef0123456789abcdef0123456789abcdef01',
    },
    'ios': {
        'client_id': 'abcdef0123456789abcdef01234567890',
        'client_secret': '0123456789abcdef0123456789abcdef01234567',
    },
}

def _fetch_oauth_token(self, video_id, client='android'):
    """Request OAuth access token"""
    client_id = self._OAUTH_CLIENTS[client]['client_id']
    client_secret = self._OAUTH_CLIENTS[client]['client_secret']

    # Base64 encode credentials
    credentials = base64.b64encode(
        f'{client_id}:{client_secret}'.encode()).decode()

    # Request token
    token_response = self._download_json(
        'https://api.vimeo.com/oauth/authorize/client',
        video_id, 'Fetching OAuth token',
        headers={
            'Authorization': f'Basic {credentials}',
            'Content-Type': 'application/json',
        },
        data=json.dumps({
            'grant_type': 'client_credentials',
            'scope': 'public private',
        }).encode())

    return token_response['access_token']
```

**Why multiple clients?**
- Different clients expose different video qualities
- Fallback if one client is rate-limited
- Access to platform-specific features

---

## JWT Token Expiry Checking

```python
def _get_valid_token(self, video_id):
    """Get cached token or fetch new one if expired"""
    if self._CACHED_TOKEN:
        try:
            # Decode JWT without verification (we trust the source)
            payload = jwt_decode_hs256(self._CACHED_TOKEN)

            # Check if token expires in more than 2 minutes
            if payload['exp'] - time.time() > 120:
                return self._CACHED_TOKEN
        except Exception:
            pass

    # Token invalid/expired, fetch new one
    self._CACHED_TOKEN = self._fetch_oauth_token(video_id)
    return self._CACHED_TOKEN
```

**JWT structure**:
```json
{
    "user_id": 123456,
    "scope": "public private",
    "exp": 1234567890,  // Expiry timestamp
    "iat": 1234567000   // Issued at timestamp
}
```

**Why 2-minute buffer?** Prevents token expiring during request.

---

## Config URL Pattern

Vimeo uses "config URL" for video data:

```python
def _extract_config_url(self, webpage, video_id):
    """Extract config URL from player initialization"""
    config_url = self._search_regex(
        r'"config_url":\s*"([^"]+)"',
        webpage, 'config URL')

    # Config URL contains video_id and signature
    # https://player.vimeo.com/video/123456/config?s=abc123def456

    return config_url
```

**Config URL provides**:
- All available video files
- Format information (quality, codec)
- Subtitle tracks
- Progressive vs DASH delivery

---

## Format Extraction

```python
def _extract_formats(self, config_url, video_id):
    token = self._get_valid_token(video_id)

    config = self._download_json(
        config_url, video_id,
        headers={'Authorization': f'Bearer {token}'})

    files = config['request']['files']

    formats = []

    # Progressive MP4 files
    if 'progressive' in files:
        for progressive in files['progressive']:
            formats.append({
                'url': progressive['url'],
                'format_id': progressive.get('quality', 'progressive'),
                'width': progressive.get('width'),
                'height': progressive.get('height'),
                'ext': 'mp4',
            })

    # HLS streams
    if 'hls' in files:
        formats.extend(self._extract_m3u8_formats(
            files['hls']['cdns'][files['hls']['default_cdn']]['url'],
            video_id, 'mp4', m3u8_id='hls'))

    # DASH streams
    if 'dash' in files:
        formats.extend(self._extract_mpd_formats(
            files['dash']['cdns'][files['dash']['default_cdn']]['url'],
            video_id))

    return formats
```

---

## Viewer Authentication

For private/password-protected videos:

```python
def _verify_video_password(self, url, video_id, password):
    """Submit password for protected video"""
    return self._download_json(
        f'{url}/check-password',
        video_id, 'Verifying video password',
        data=urlencode_postdata({
            'password': password,
            '_method': 'POST',
        }),
        headers={
            'X-Requested-With': 'XMLHttpRequest',
        })
```

---

## Referer Smuggling

Some embed-only videos require referer:

```python
def _extract_embed_only(self, url, video_id):
    """Handle embed-only videos"""
    # Get referrer from URL
    url, smuggled_data = unsmuggle_url(url, {})
    referrer = smuggled_data.get('http_headers', {}).get('Referer')

    # Add referer to requests
    if referrer:
        self._downloader.params.setdefault('http_headers', {})
        self._downloader.params['http_headers']['Referer'] = referrer

    return self._real_extract(url)
```

**Referer smuggling**: Passing referrer information through URL encoding to bypass restrictions.

---

## Key Takeaways

1. **OAuth 2.0** - Client credentials flow for API access
2. **JWT validation** - Check token expiry before use
3. **Token caching** - Reuse tokens across requests
4. **Multi-client strategy** - Try different clients for best results
5. **Config URL pattern** - Centralized video metadata
6. **Format diversity** - Progressive, HLS, and DASH support

**Best practices**:
- Cache OAuth tokens per session
- Always check JWT expiry
- Have fallback clients ready
- Handle authentication errors gracefully

---

[← TikTok](06-tiktok.md) | [Next: HotStar →](08-hotstar.md)
