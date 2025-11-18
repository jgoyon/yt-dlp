# Kaltura Extractor

Embeddable platform architecture and multi-request API patterns.

## Overview

**File**: `yt_dlp/extractor/kaltura.py`
**Complexity**: Medium
**Key Feature**: Multi-request API pattern for embeddable video platform

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/kaltura.py)

---

## The Embeddable Platform Problem

Kaltura is a video platform used by:
- Media companies
- Educational institutions
- Corporate sites
- News organizations

**Challenge**: Same video, different embeddings, various partner IDs.

---

## Special URL Format

```python
_VALID_URL = r'''(?x)
    kaltura:(?P<partner_id>\d+):(?P<id>[0-9a-z_]+)
    |https?://
        (:?www\.|cdnapi(?:sec)?\.)?kaltura\.com(?::\d+)?
        /(?:(?:p/(?P<partner_id_2>\d+))|(?:index\.php/partnerservices2/executeplaylist\?partner_id=(?P<partner_id_3>\d+)))
        ...
'''
```

**Example URLs**:
```
kaltura:12345:abc123def456
https://www.kaltura.com/p/12345/sp/1234500/embedIframeJs/uiconf_id/23456789/partner_id/12345/entry_id/abc123def456
```

**Components**:
- `partner_id`: Account identifier
- `entry_id`: Video identifier
- `uiconf_id`: Player configuration

---

## Multi-Request API Pattern

Kaltura uses batched API requests:

```python
def _call_api(self, partner_id, entry_id):
    """Call Kaltura multi-request API"""
    api_url = f'https://cdnapisec.kaltura.com/api_v3/service/multirequest'

    # Build multi-request payload
    data = {
        '1:service': 'baseEntry',
        '1:action': 'get',
        '1:entryId': entry_id,
        '2:service': 'baseEntry',
        '2:action': 'getContextData',
        '2:entryId': entry_id,
        '2:contextDataParams:objectType': 'KalturaEntryContextDataParams',
        'format': 1,  # JSON
        'ks': '',     # Session key (empty for public videos)
    }

    response = self._download_json(
        api_url, entry_id,
        data=urlencode_postdata(data))

    # Response contains multiple results
    return {
        'entry': response[0],
        'context': response[1],
    }
```

**Multi-request benefits**:
- Single HTTP request
- Multiple operations
- Reduced latency

---

## Service URL Construction

```python
def _get_service_url(self, partner_id):
    """Construct base service URL"""
    return f'https://cdnapisec.kaltura.com/p/{partner_id}/sp/{partner_id}00/playManifest'
```

**Partner-specific URLs**: Each partner has unique service endpoint.

---

## Caption Format Mapping

```python
_CAPTION_TYPES = {
    1: 'srt',    # SubRip
    2: 'ttml',   # Timed Text
    3: 'vtt',    # WebVTT
}

def _extract_captions(self, entry_id, partner_id):
    """Extract caption tracks"""
    captions = {}

    for caption_data in self._download_json(
        f'https://cdnapisec.kaltura.com/api_v3/service/caption_captionasset/action/list',
        entry_id,
        query={'entryId': entry_id}):

        lang = caption_data.get('language')
        caption_type = caption_data.get('format')

        captions.setdefault(lang, []).append({
            'url': caption_data['downloadUrl'],
            'ext': self._CAPTION_TYPES.get(caption_type, 'srt'),
        })

    return captions
```

---

## Playlist Support

```python
def _extract_playlist(self, playlist_id, partner_id):
    """Extract playlist entries"""
    playlist_data = self._download_json(
        'https://cdnapisec.kaltura.com/api_v3/service/playlist/action/execute',
        playlist_id,
        query={
            'id': playlist_id,
            'partnerId': partner_id,
        })

    entries = []
    for item in playlist_data:
        entry_id = item.get('id')
        entries.append(self.url_result(
            f'kaltura:{partner_id}:{entry_id}',
            'Kaltura',
            entry_id))

    return self.playlist_result(entries, playlist_id)
```

---

## Unsmuggling for Embed Context

```python
def _real_extract(self, url):
    # Check if URL has smuggled data (from embed)
    url, smuggled_data = unsmuggle_url(url, {})

    partner_id = smuggled_data.get('partner_id') or self._match_partner_id(url)
    entry_id = smuggled_data.get('entry_id') or self._match_id(url)

    # Use smuggled referrer if available
    if 'referrer' in smuggled_data:
        self._downloader.params.setdefault('http_headers', {})
        self._downloader.params['http_headers']['Referer'] = smuggled_data['referrer']

    return self._extract_video(partner_id, entry_id)
```

**Unsmuggling**: Extract metadata embedded in URL by other extractors.

---

## Flash vs HTML5 Player

```python
def _get_player_type(self, uiconf_id, partner_id):
    """Determine player type"""
    player_config = self._download_json(
        f'https://cdnapisec.kaltura.com/api_v3/service/uiconf/action/get',
        None,
        query={
            'id': uiconf_id,
            'partnerId': partner_id,
        })

    if 'flash' in player_config.get('tags', '').lower():
        return 'flash'
    else:
        return 'html5'
```

**Why matters**: Different players expose different format options.

---

## Key Takeaways

1. **Multi-request API** - Batch operations in single request
2. **Partner-based URLs** - Account-specific endpoints
3. **Embed handling** - Unsmuggle context from parent
4. **Caption mapping** - Format type conversion
5. **Playlist support** - Recursive entry extraction

**Pattern applicability**:
- Any embeddable video platform (JWPlayer, Brightcove)
- Multi-tenant SaaS video services
- Partner-based API architectures

---

[← CDA](09-cda.md) | [Next: Archive.org →](11-archiveorg.md)
