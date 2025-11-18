# Twitch Extractor

Live streaming, GraphQL operations, and real-time video handling.

## Overview

**File**: `yt_dlp/extractor/twitch.py`
**Complexity**: Complex
**Key Features**: Live streams, VODs, GraphQL, 2FA support

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/twitch.py)

---

## GraphQL Operation Hashes

Twitch uses GraphQL with hardcoded operation hashes:

```python
_OPERATION_HASHES = {
    'VideoMetadata': '49b5b8f268cdeb259d75b58dcb0c1a748e3b575003448a2333dc5cdafd49adad',
    'VideoAccessToken': '0828119ded1c13477966434e15800ff57ddacf13ba1911c129dc2200705b0712',
    'ClipsCards': 'b73ad2bfaecfd30a9e6c28fada15bd97032c83ec77a0440766a56fe0bd632777',
}

def _call_gql_api(self, operation, video_id):
    return self._download_json(
        'https://gql.twitch.tv/gql',
        video_id,
        data=json.dumps([{
            'operationName': operation,
            'extensions': {
                'persistedQuery': {
                    'version': 1,
                    'sha256Hash': self._OPERATION_HASHES[operation],
                }
            },
            'variables': {...},
        }]).encode())
```

**Key insight**: GraphQL operations identified by SHA256 hash, not query text

---

## Access Token System

Twitch requires access tokens for video URLs:

```python
def _get_access_token(self, video_id, kind='vod'):
    """Get signed token to access video"""
    token = self._call_gql_api('VideoAccessToken', video_id)

    return {
        'token': token['value'],
        'signature': token['signature'],
    }
```

**Token includes**:
- Temporary authorization
- Signature for validation
- Prevents URL sharing

---

## Live vs VOD Detection

```python
def _real_extract(self, url):
    video_id = self._match_id(url)

    # Try live stream first
    try:
        return self._extract_live(video_id)
    except ExtractorError:
        # Fall back to VOD
        return self._extract_vod(video_id)
```

**Different handling**:
- **Live**: Real-time HLS stream, no seeking
- **VOD**: Full video, chapter support, quality selection

---

## HLS Playlist Manipulation

```python
# Get master playlist
access_token = self._get_access_token(video_id)
m3u8_url = f'https://usher.ttvnw.net/vod/{video_id}.m3u8'

# Add access token to URL
m3u8_url = update_url_query(m3u8_url, {
    'token': access_token['token'],
    'sig': access_token['signature'],
    'allow_source': 'true',
    'player': 'twitchweb',
})

# Extract formats
formats = self._extract_m3u8_formats(
    m3u8_url, video_id, 'mp4',
    preference=10,  # Prefer source quality
    m3u8_id='hls')
```

**Quality selection**:
- `source`: Original stream quality (best)
- `720p60`, `480p`, `360p`, etc.: Transcoded qualities

---

## Two-Factor Authentication

```python
def _perform_login(self, username, password):
    # Initial login
    login_response = self._download_json(
        'https://passport.twitch.tv/login',
        None, 'Logging in',
        data=urlencode_postdata({
            'username': username,
            'password': password,
        }))

    # Check if 2FA required
    if login_response.get('error_code') == 'captcha_required':
        raise ExtractorError('Captcha required')

    if login_response.get('error_code') == '3012':  # 2FA
        # Prompt for 2FA code
        twofa_token = self._get_twofa_token()

        # Submit 2FA
        self._download_json(
            'https://passport.twitch.tv/login',
            None, 'Submitting 2FA',
            data={**login_data, 'authy_token': twofa_token})
```

---

## Live Stream Features

**DVR Support** (rewind live streams):
```python
if stream_data.get('type') == 'live':
    info_dict.update({
        'is_live': True,
        'was_live': False,
    })

    # Some live streams support DVR (rewind)
    if 'dvr' in stream_data.get('broadcast_type', ''):
        info_dict['live_start_time'] = unified_timestamp(stream_data.get('created_at'))
```

---

## Clips Extraction

Twitch Clips use different API:

```python
class TwitchClipsIE(TwitchBaseIE):
    _VALID_URL = r'https?://clips\.twitch\.tv/(?P<id>[^/?#]+)'

    def _real_extract(self, url):
        clip_id = self._match_id(url)

        # Use ClipsCards GraphQL operation
        clip_data = self._call_gql_api('ClipsCards', clip_id)

        # Extract video URLs from clip data
        quality_options = clip_data.get('videoQualities', [])
        formats = [{
            'url': quality['sourceURL'],
            'format_id': quality.get('quality'),
            'height': int_or_none(quality.get('quality')),
        } for quality in quality_options]

        return {
            'id': clip_id,
            'title': clip_data.get('title'),
            'formats': formats,
        }
```

---

## Key Takeaways

1. **GraphQL with hashes** - Operations identified by SHA256
2. **Access tokens** - Required for video URL access
3. **Live vs VOD** - Different extraction paths
4. **HLS manipulation** - Add auth params to playlist URL
5. **2FA support** - Multi-step authentication
6. **DVR awareness** - Detect rewindable live streams

**Best practices**:
- Try multiple extraction methods (live → VOD)
- Cache access tokens per video
- Handle authentication flows gracefully

---

[← Twitter](03-twitter.md) | [Next: SoundCloud →](05-soundcloud.md)
