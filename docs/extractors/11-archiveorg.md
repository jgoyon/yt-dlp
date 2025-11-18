# Archive.org Extractor

Multi-format handling and metadata-rich extraction.

## Overview

**File**: `yt_dlp/extractor/archiveorg.py`
**Complexity**: Medium
**Key Feature**: Multiple file format support from single item

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/archiveorg.py)

---

## The Internet Archive

archive.org preserves:
- Historical videos
- Software/games
- Books and documents
- Audio recordings
- Web pages (Wayback Machine)

**Challenge**: Items often have multiple file formats to choose from.

---

## Metadata API

```python
def _real_extract(self, url):
    item_id = self._match_id(url)

    # Fetch item metadata
    metadata = self._download_json(
        f'https://archive.org/metadata/{item_id}',
        item_id)

    return self._extract_from_metadata(metadata)
```

**Metadata includes**:
- All files associated with item
- Format information
- Technical details
- Descriptive metadata

---

## Multi-Format Selection

```python
def _extract_formats(self, files, item_id):
    """Extract all available formats"""
    formats = []

    for file_data in files:
        # Skip non-media files
        if file_data.get('format') not in ['MPEG4', 'h.264', 'Ogg Video']:
            continue

        # Original file (best quality)
        if file_data.get('original'):
            preference = 10
        else:
            preference = -1

        formats.append({
            'url': f'https://archive.org/download/{item_id}/{file_data["name"]}',
            'format_id': file_data.get('format'),
            'filesize': int_or_none(file_data.get('size')),
            'width': int_or_none(file_data.get('width')),
            'height': int_or_none(file_data.get('height')),
            'preference': preference,
        })

    return formats
```

**Format types**:
- **Original**: Uploaded file (best quality)
- **Derivatives**: Transcoded versions
- **Thumbnails**: Preview images

---

## YouTube Video Fallback

Some archive.org items are YouTube videos:

```python
def _real_extract(self, url):
    item_id = self._match_id(url)
    metadata = self._download_json(...)

    # Check if item is YouTube video
    youtube_id = traverse_obj(metadata, ('metadata', 'identifier-access'))

    if youtube_id:
        # Delegate to YouTube extractor
        return self.url_result(
            f'https://youtube.com/watch?v={youtube_id}',
            'Youtube',
            youtube_id)

    # Otherwise extract from archive.org
    return self._extract_from_metadata(metadata)
```

---

## Extensive Metadata Extraction

```python
def _extract_metadata(self, metadata):
    """Extract descriptive metadata"""
    meta = metadata.get('metadata', {})

    return {
        'id': metadata.get('identifier'),
        'title': meta.get('title'),
        'description': meta.get('description'),
        'uploader': meta.get('creator'),
        'upload_date': unified_strdate(meta.get('date')),
        'duration': parse_duration(meta.get('runtime')),
        'license': meta.get('license'),
        'tags': meta.get('subject', '').split(';') if meta.get('subject') else None,
    }
```

**Metadata fields**:
- `title`: Item title
- `creator`: Uploader/creator
- `date`: Publication date
- `license`: Content license (Creative Commons, Public Domain, etc.)
- `subject`: Tags/categories

---

## Thumbnail Selection

```python
def _extract_thumbnail(self, files, item_id):
    """Select best thumbnail"""
    thumbnails = []

    for file_data in files:
        # Look for image files
        if file_data.get('format') in ['JPEG', 'PNG', 'Item Image']:
            thumbnails.append({
                'url': f'https://archive.org/download/{item_id}/{file_data["name"]}',
                'width': int_or_none(file_data.get('width')),
                'height': int_or_none(file_data.get('height')),
            })

    # Default thumbnail
    if not thumbnails:
        thumbnails.append({
            'url': f'https://archive.org/services/img/{item_id}',
        })

    return thumbnails
```

---

## Format Extension Detection

```python
def _determine_format(self, file_data):
    """Determine format from file metadata"""
    # Explicit format field
    if 'format' in file_data:
        return file_data['format'].lower()

    # Infer from filename
    filename = file_data.get('name', '')
    ext = determine_ext(filename)

    format_map = {
        'mp4': 'MPEG4',
        'ogv': 'Ogg Video',
        'mkv': 'Matroska',
        'mov': 'QuickTime',
    }

    return format_map.get(ext, ext)
```

---

## Original Quality Preference

```python
# Prefer original files over derivatives
for fmt in formats:
    if 'original' in fmt.get('format_note', '').lower():
        fmt['preference'] = 100
    elif 'derivative' in fmt.get('format_note', '').lower():
        fmt['preference'] = -1
```

**Why**: Original files are unprocessed, highest quality.

---

## License Information Parsing

```python
_LICENSE_MAP = {
    'cc0': 'Creative Commons Zero',
    'cc-by': 'Creative Commons Attribution',
    'cc-by-sa': 'Creative Commons Attribution-ShareAlike',
    'cc-by-nc': 'Creative Commons Attribution-NonCommercial',
    'publicdomain': 'Public Domain',
}

def _extract_license(self, metadata):
    """Extract and normalize license info"""
    license_str = traverse_obj(metadata, ('metadata', 'license'))

    if not license_str:
        return None

    # Normalize license string
    license_lower = license_str.lower()
    for key, full_name in self._LICENSE_MAP.items():
        if key in license_lower:
            return full_name

    return license_str
```

---

## Key Takeaways

1. **Multi-format handling** - Choose from multiple file versions
2. **Metadata richness** - Extract comprehensive item information
3. **Original preference** - Prioritize unprocessed files
4. **YouTube fallback** - Delegate when appropriate
5. **License extraction** - Identify content usage rights

**Use cases**:
- Historical video archives
- Public domain content
- Creative Commons media
- Research materials

**Pattern applicability**:
- Any multi-file item system
- Digital library platforms
- Media archive services

---

[← Kaltura](10-kaltura.md) | [Back to Overview](00-overview.md)
