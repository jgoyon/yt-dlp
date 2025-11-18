# Development Guide

Guide to creating new extractors and contributing to yt-dlp.

## Creating a New Extractor

### Step 1: Create Extractor File

```bash
# Create file in yt_dlp/extractor/
touch yt_dlp/extractor/mysite.py
```

### Step 2: Basic Template

```python
from .common import InfoExtractor

class MySiteIE(InfoExtractor):
    _VALID_URL = r'https?://(?:www\.)?mysite\.com/video/(?P<id>[0-9]+)'
    _TESTS = [{
        'url': 'https://mysite.com/video/12345',
        'info_dict': {
            'id': '12345',
            'ext': 'mp4',
            'title': 'Test Video',
        },
    }]

    def _real_extract(self, url):
        video_id = self._match_id(url)
        
        # Download webpage
        webpage = self._download_webpage(url, video_id)
        
        # Extract data
        title = self._html_search_regex(
            r'<h1[^>]*>([^<]+)</h1>',
            webpage, 'title')
        
        video_url = self._html_search_regex(
            r'src="([^"]+\.mp4)"',
            webpage, 'video URL')
        
        return {
            'id': video_id,
            'title': title,
            'url': video_url,
        }
```

### Step 3: Register Extractor

Add to `yt_dlp/extractor/_extractors.py`:
```python
from .mysite import MySiteIE
```

### Step 4: Test

```bash
python test/test_download.py TestDownload.test_MySite
```

## Best Practices

1. **Use helper methods**: `_download_webpage()`, `_search_regex()`, `traverse_obj()`
2. **Handle errors gracefully**: Use `ExtractorError` with `expected=True`
3. **Add comprehensive tests**: Test multiple URL formats and edge cases
4. **Document in code**: Add comments for complex logic
5. **Follow naming conventions**: `MySiteIE` for extractor class

## Common Patterns

### API Extraction
```python
data = self._download_json(api_url, video_id)
title = traverse_obj(data, ('video', 'title'))
```

### Multiple Formats
```python
formats = []
for fmt in video_data['formats']:
    formats.append({
        'url': fmt['url'],
        'height': fmt.get('height'),
        'ext': fmt.get('ext'),
    })

return {
    'id': video_id,
    'title': title,
    'formats': formats,
}
```

### HLS/DASH Formats
```python
formats = self._extract_m3u8_formats(
    m3u8_url, video_id, 'mp4')

# or

formats = self._extract_mpd_formats(
    mpd_url, video_id)
```

## See Also
- [Python Features](python-features.md) - Coding patterns
- [Testing](testing.md) - Writing tests
- [Extractor Examples](extractors/) - Real-world examples

---

[← Back to Documentation](README.md)
