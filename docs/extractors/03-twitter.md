# Twitter/X Extractor

GraphQL API integration and JavaScript interpretation.

## Overview

**File**: `yt_dlp/extractor/twitter.py`
**Complexity**: Complex
**Key Techniques**: GraphQL queries, Bearer token auth, JavaScript interpretation

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/twitter.py)

---

## GraphQL API Pattern

Twitter uses GraphQL for data queries:

```python
_GRAPHQL_ENDPOINT = 'https://twitter.com/i/api/graphql'

# Hardcoded operation hash (reverse-engineered from Twitter web app)
_OPERATION_HASH = 'zZXycP0V6H7m-2r0mOnFcA'

def _call_graphql_api(self, endpoint, video_id):
    query_url = f'{self._GRAPHQL_ENDPOINT}/{self._OPERATION_HASH}/{endpoint}'

    return self._download_json(
        query_url, video_id,
        headers={
            'Authorization': f'Bearer {self._BEARER_TOKEN}',
            'X-Guest-Token': self._guest_token,
        })
```

**Key Points**:
- GraphQL uses operation hashes instead of query strings
- Requires bearer token + guest token
- Hashes must be reverse-engineered from web app

---

## Authentication Flow

```python
def _fetch_guest_token(self):
    """Get guest token for unauthenticated access"""
    guest_token = self._download_json(
        'https://api.twitter.com/1.1/guest/activate.json',
        None, 'Downloading guest token',
        headers={
            'Authorization': f'Bearer {self._BEARER_TOKEN}',
        })

    return guest_token['guest_token']
```

**Bearer token**: Hardcoded in extractor (from Twitter web app)
**Guest token**: Fetched per session for anonymous access

---

## Flow-Based Login

For protected/private tweets:

```python
def _perform_login(self, username, password):
    flow_token = None

    # Multi-step login flow
    for subtask in ['LoginEnterUserIdentifierSSO', 'LoginEnterPassword', 'AccountDuplicationCheck']:
        response = self._download_json(
            'https://api.twitter.com/1.1/onboarding/task.json',
            None, f'Performing subtask: {subtask}',
            json={'flow_token': flow_token, 'subtask_inputs': [...]})

        flow_token = response['flow_token']
```

**Flow-based auth**: Multi-step challenge-response system

---

## Video Format Extraction

```python
# Progressive download formats
for variant in video_info['variants']:
    if variant.get('content_type') == 'video/mp4':
        formats.append({
            'url': variant['url'],
            'format_id': f"http-{variant.get('bitrate', 0)}",
            'ext': 'mp4',
            'tbr': variant.get('bitrate'),
        })

# HLS manifest (if available)
if 'm3u8' in str(video_info):
    formats.extend(self._extract_m3u8_formats(
        hls_url, video_id, 'mp4'))
```

**Twitter serves**:
- Progressive MP4 files (multiple bitrates)
- HLS streams (for some videos)

---

## JavaScript Number Interpretation

```python
from ..jsinterp import js_number_to_string

# Twitter sometimes returns numbers as JavaScript expressions
view_count = js_number_to_string(data.get('view_count_string'))
```

**Why needed**: Twitter API sometimes returns `"1.2e6"` instead of `1200000`

---

## Key Takeaways

1. **GraphQL APIs** - Operation hashes, not query strings
2. **Multi-token auth** - Bearer + guest tokens
3. **Flow-based login** - Step-by-step challenge system
4. **JavaScript interpretation** - Handle JS number formats
5. **Dual format support** - Progressive + HLS

**Applicable to**: Any GraphQL-based API (GitHub, Shopify, Facebook)

---

[← YouTube](02-youtube.md) | [Next: Twitch →](04-twitch.md)
