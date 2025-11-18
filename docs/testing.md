# Testing Methodology

This document explains how yt-dlp tests complex extraction logic across 1000+ extractors and thousands of supported websites.

## Table of Contents

- [Testing Philosophy](#testing-philosophy)
- [Test Structure](#test-structure)
- [Extractor Testing](#extractor-testing)
- [Test Types](#test-types)
- [Writing Tests](#writing-tests)
- [Running Tests](#running-tests)
- [Debugging Failed Tests](#debugging-failed-tests)
- [Continuous Integration](#continuous-integration)

---

## Testing Philosophy

### Unique Challenges

Testing a video downloader presents unique challenges:

1. **Live Dependencies**: Most tests require network access to actual websites
2. **Content Volatility**: Videos may be deleted, made private, or geo-restricted
3. **Site Changes**: Websites update their HTML/API structure constantly
4. **Rate Limiting**: Testing hundreds of sites can trigger rate limits
5. **Performance**: Tests must complete in reasonable time

### Testing Approach

yt-dlp balances comprehensive testing with practical constraints:

**Live Tests**: Most extractor tests download actual content from real websites
- **Pros**: Tests real-world behavior, catches API/HTML changes
- **Cons**: Slow, network-dependent, can fail due to external factors

**Unit Tests**: Core utilities and functions are tested in isolation
- **Pros**: Fast, reliable, no network dependency
- **Cons**: Doesn't catch integration issues

**Test Selection**: Tests can be run selectively by extractor or test case
- Full test suite: ~hours
- Single extractor: ~seconds

---

## Test Structure

### Directory Organization

```
test/
├── test_download.py           # Live extractor tests
├── test_YoutubeDL.py          # Core YoutubeDL functionality
├── test_InfoExtractor.py      # InfoExtractor base class
├── test_utils.py              # Utility function tests
├── test_networking.py         # Network layer tests
├── test_postprocessors.py     # Post-processor tests
├── test_jsinterp.py           # JavaScript interpreter tests
├── helper.py                  # Test helper functions
└── testdata/                  # Test data files
    ├── srt/                   # Subtitle test files
    ├── filter/                # Format filter tests
    └── ...
```

### Test Framework

yt-dlp uses Python's built-in `unittest` framework:

```python
import unittest

class TestDownload(unittest.TestCase):
    def test_youtube(self):
        # Test YouTube extraction
        pass
```

Can also run with `pytest`:
```bash
python -m pytest test/
```

---

## Extractor Testing

### Test Definition in Extractors

Every extractor includes test cases in its `_TESTS` attribute:

```python
# From yt_dlp/extractor/soundcloud.py:29
class SoundcloudEmbedIE(InfoExtractor):
    _VALID_URL = r'https?://(?:w|player|p)\.soundcloud\.com/player/?.*?\burl=(?P<id>.+)'

    _TESTS = [{
        # Test case 1: Basic playback
        'url': 'https://w.soundcloud.com/player/?url=https%3A%2F%2Fapi.soundcloud.com%2Fplaylists%2F922213810',
        'only_matching': True,
    }]

    _WEBPAGE_TESTS = [{
        # Test case 2: Embedded player
        'url': 'https://news.sophos.com/en-us/2023/08/10/...',
        'info_dict': {
            'id': '1588847423',
            'ext': 'm4a',
            'title': 'S3 Ep147: What if you type in your password during a meeting?',
            'artists': ['Naked Security'],
            'duration': 942.762,
            'timestamp': 1691624365,
            'upload_date': '20230809',
        },
        'params': {'skip_download': 'm3u8'},
    }]
```

[View in code](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/soundcloud.py#L29)

### Test Case Structure

Each test case is a dictionary with these fields:

```python
{
    # Required: URL to test
    'url': 'https://example.com/video/123',

    # Expected metadata (info_dict)
    'info_dict': {
        'id': 'video_id',
        'ext': 'mp4',                    # File extension
        'title': 'Video Title',
        'description': 'Description',
        'uploader': 'Channel Name',
        'duration': 120,                 # seconds
        'timestamp': 1234567890,         # Unix timestamp
        'upload_date': '20201225',       # YYYYMMDD
        'view_count': int,               # Use `int` for "at least 0"
        'like_count': lambda x: x >= 100,  # Custom validator
        'tags': 'count:5',               # Exactly 5 tags
        'thumbnail': r're:https?://.*\.jpg',  # Regex pattern
    },

    # Optional fields
    'md5': 'abc123...',                  # MD5 of downloaded file
    'params': {
        'skip_download': True,           # Don't download, just extract
        'format': 'bestaudio',           # Format to test
    },
    'skip': 'Reason to skip this test',
    'only_matching': True,               # Only test URL matching, not extraction
    'expected_warnings': [
        'Warning message pattern',       # Expected warning messages
    ],
}
```

### Test Validation Types

#### Exact Match
```python
'info_dict': {
    'title': 'Exact Title',
    'duration': 42,
}
```

#### Type Check
```python
'info_dict': {
    'view_count': int,      # Any integer
    'upload_date': str,     # Any string
}
```

#### Range Check (Lambda)
```python
'info_dict': {
    'like_count': lambda x: x >= 100,
    'duration': lambda x: 30 <= x <= 60,
}
```

#### Regex Match
```python
'info_dict': {
    'thumbnail': r're:https?://.*\.jpg$',
    'url': r're:https://.*\.mp4',
}
```

#### Count Check
```python
'info_dict': {
    'tags': 'count:5',              # Exactly 5 items
    'tags': 'mincount:3',           # At least 3 items
    'tags': 'maxcount:10',          # At most 10 items
}
```

#### Markdown Match
```python
'info_dict': {
    'description': 'md5:abc123...',  # MD5 hash of description
}
```

### Playlist Testing

For playlist extractors:

```python
_TESTS = [{
    'url': 'https://example.com/playlist/123',
    'info_dict': {
        'id': 'playlist_id',
        'title': 'Playlist Title',
    },
    'playlist_count': 25,           # Exactly 25 videos
    'playlist_mincount': 20,        # At least 20 videos
    'playlist_maxcount': 30,        # At most 30 videos
}]
```

---

## Test Types

### 1. Live Extraction Tests (`test_download.py`)

**Purpose**: Test actual extraction from live websites

**How it works**:
```python
# From test/test_download.py:81
def generator(test_case, tname):
    def test_template(self):
        ie = yt_dlp.extractor.get_info_extractor(test_case['name'])()

        # Extract info
        res_dict = ydl.extract_info(
            test_url,
            force_generic_extractor=params.get('force_generic_extractor', False))

        # Validate result
        expect_info_dict(self, res_dict, test_case.get('info_dict', {}))

    return test_template
```

[View in code](https://github.com/yt-dlp/yt-dlp/blob/master/test/test_download.py#L81)

**Test Generation**: Tests are dynamically generated from `_TESTS`:
```python
# For each extractor test case, create a unittest method
for ie_name, test_cases in all_test_cases:
    for test_num, test_case in enumerate(test_cases):
        test_method = generator(test_case, f'{ie_name}_{test_num}')
        test_method.add_ie = ie_name
        setattr(TestDownload, f'test_{ie_name}_{test_num}', test_method)
```

### 2. Unit Tests (`test_utils.py`)

**Purpose**: Test utility functions in isolation

**Example**:
```python
# From test/test_utils.py
class TestUtil(unittest.TestCase):
    def test_parse_duration(self):
        self.assertEqual(parse_duration('1:30'), 90)
        self.assertEqual(parse_duration('2:30:45'), 9045)
        self.assertEqual(parse_duration('1h 30m'), 5400)

    def test_unified_timestamp(self):
        self.assertEqual(unified_timestamp('Dec 31, 2020'), 1609372800)
        self.assertEqual(unified_timestamp('2020-12-31'), 1609372800)
```

### 3. Core Functionality Tests (`test_YoutubeDL.py`)

**Purpose**: Test YoutubeDL class functionality

**Example**:
```python
# Format selection testing
def test_format_selection(self):
    ydl = YoutubeDL({'format': 'best[height<=720]'})
    # Test format selection logic

# Output template testing
def test_output_template(self):
    ydl = YoutubeDL({'outtmpl': '%(title)s-%(id)s.%(ext)s'})
    # Test template rendering
```

### 4. Network Layer Tests (`test_networking.py`)

**Purpose**: Test HTTP backends and network abstraction

**Example**:
```python
def test_urllib_backend(self):
    # Test urllib handler
    pass

def test_request_headers(self):
    # Test header handling
    pass
```

---

## Writing Tests

### Step-by-Step Guide

#### 1. Add Test Case to Extractor

```python
# In yt_dlp/extractor/mysite.py
class MySiteIE(InfoExtractor):
    _VALID_URL = r'https?://mysite\.com/video/(?P<id>[0-9]+)'

    _TESTS = [{
        'url': 'https://mysite.com/video/12345',
        'md5': 'abc123def456...',  # Run once to get actual MD5
        'info_dict': {
            'id': '12345',
            'ext': 'mp4',
            'title': 'Test Video Title',
            'description': 'Test Description',
            'uploader': 'Test Uploader',
            'duration': 120,
        },
    }]
```

#### 2. Run Test to Get Actual Values

```bash
# First run without info_dict to see actual values
python test/test_download.py TestDownload.test_MySite_0
```

**Output**:
```
Expected: {'title': 'Test Video Title'}
Got:      {'title': 'Actual Video Title From Site'}
```

#### 3. Update Test with Actual Values

```python
_TESTS = [{
    'url': 'https://mysite.com/video/12345',
    'md5': '8a3d5e9f7c2b1a...',  # Actual MD5 from test run
    'info_dict': {
        'id': '12345',
        'ext': 'mp4',
        'title': 'Actual Video Title From Site',  # Updated
        'description': 'md5:a1b2c3d4...',  # MD5 if description is long
        'uploader': 'Actual Uploader',
        'duration': 145,  # Actual duration
        'timestamp': 1609459200,
        'upload_date': '20210101',
    },
}]
```

#### 4. Add Multiple Test Cases

```python
_TESTS = [
    {
        # Test case 1: Normal video
        'url': 'https://mysite.com/video/12345',
        'info_dict': {...},
    },
    {
        # Test case 2: Age-restricted video
        'url': 'https://mysite.com/video/67890',
        'info_dict': {...},
    },
    {
        # Test case 3: Playlist
        'url': 'https://mysite.com/playlist/abc',
        'playlist_mincount': 10,
        'info_dict': {...},
    },
    {
        # Test case 4: URL format variation
        'url': 'https://www.mysite.com/v/12345',
        'only_matching': True,  # Just test URL matching
    },
]
```

### Best Practices

1. **Choose Stable Content**: Use videos unlikely to be deleted (official channels, popular content)
2. **Test Edge Cases**: Age-restriction, geo-blocking, playlists, live streams
3. **Use `only_matching` for Alternatives**: For URL pattern variations, just test matching
4. **Skip Downloads When Possible**: Use `'params': {'skip_download': True}` for metadata-only tests
5. **Test Multiple Qualities**: Include tests for different format selections
6. **Document Expected Failures**: Use `'skip'` with reason if test is known to fail temporarily

### Common Patterns

#### Testing Different URL Formats
```python
_TESTS = [
    {
        'url': 'https://site.com/watch?v=abc123',
        'info_dict': {...},
    },
    {
        # Mobile URL
        'url': 'https://m.site.com/watch?v=abc123',
        'only_matching': True,
    },
    {
        # Short URL
        'url': 'https://site.com/v/abc123',
        'only_matching': True,
    },
]
```

#### Testing Age-Restricted Content
```python
{
    'url': 'https://site.com/video/age_restricted',
    'info_dict': {
        'age_limit': 18,
        # ... other fields
    },
}
```

#### Testing Geo-Restricted Content
```python
{
    'url': 'https://site.com/video/geo_blocked',
    'skip': 'Geo-restricted to US only',
}
```

---

## Running Tests

### Run All Tests
```bash
# Run entire test suite
python -m pytest test/

# Or using unittest
python -m unittest discover test/
```

### Run Specific Extractor Tests
```bash
# Test specific extractor (e.g., YouTube)
python test/test_download.py TestDownload.test_youtube

# Test all YouTube test cases
python -m pytest test/test_download.py -k youtube
```

### Run Single Test Case
```bash
# Test specific test case (extractor_name + test index)
python test/test_download.py TestDownload.test_youtube_0
python test/test_download.py TestDownload.test_soundcloud_1
```

### Run with Verbose Output
```bash
# See detailed output
python -m pytest -v test/

# See print statements
python -m pytest -s test/
```

### Skip Download (Faster)
```bash
# Test extraction only, don't download files
python test/test_download.py --skip-download
```

### Environment Variables
```bash
# Run only specific extractors
export YTDL_TEST_ONLY=youtube,vimeo
python test/test_download.py

# Skip specific extractors
export YTDL_TEST_SKIP=youtube
python test/test_download.py
```

---

## Debugging Failed Tests

### 1. Verbose Mode

```bash
# Get detailed extraction log
python test/test_download.py TestDownload.test_youtube_0 -v
```

### 2. Manual Extraction

```bash
# Run yt-dlp directly with verbose output
yt-dlp -v 'https://example.com/video/123'
```

### 3. Print Traffic

```bash
# See all HTTP requests/responses
yt-dlp --print-traffic 'https://example.com/video/123'
```

### 4. Write Pages to Disk

```bash
# Save downloaded HTML/JSON for inspection
yt-dlp --write-pages 'https://example.com/video/123'
```

### 5. Test in Python REPL

```python
from yt_dlp import YoutubeDL
from yt_dlp.extractor import get_info_extractor

# Create extractor
ie = get_info_extractor('YouTube')()

# Set YoutubeDL instance
ydl = YoutubeDL({'verbose': True})
ie.set_downloader(ydl)

# Extract
info = ie.extract('https://youtube.com/watch?v=...')
print(info)
```

### 6. Common Failure Patterns

#### Network Errors
```
TransportError: Connection timeout
```
**Solution**: Retry test, check internet connection

#### Site Changes
```
ExtractorError: Unable to extract video URL
```
**Solution**: Inspect HTML, update extractor regex/parsing logic

#### Metadata Mismatch
```
AssertionError: Expected 'Old Title' but got 'New Title'
```
**Solution**: Update test case with new expected values

#### Geo-Restriction
```
ExtractorError: This video is not available in your country
```
**Solution**: Add `'skip': 'Geo-restricted'` or use VPN for testing

---

## Continuous Integration

### GitHub Actions Workflow

**Location**: `.github/workflows/core.yml`

**Test Matrix**:
- **Python versions**: 3.10, 3.11, 3.12
- **Operating systems**: Ubuntu, Windows, macOS
- **Test types**: Quick tests, core tests, download tests

**Workflow**:
```yaml
name: Core Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.10', '3.11', '3.12']

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install dependencies
      run: pip install -r requirements.txt
    - name: Run tests
      run: python -m pytest test/
```

### Test Selection in CI

**Quick Tests** (~5 minutes):
- Core functionality
- Utility functions
- No live extraction

**Full Tests** (~1 hour):
- All extractors
- Live downloads
- Integration tests

### Handling Flaky Tests

**Retry Logic**:
```python
# From test/test_download.py:39
RETRIES = 3

try_num = 1
while True:
    try:
        res_dict = ydl.extract_info(test_url)
    except (DownloadError, ExtractorError) as err:
        if try_num == RETRIES:
            raise
        print(f'Retrying: {try_num} failed tries')
        try_num += 1
    else:
        break
```

[View in code](https://github.com/yt-dlp/yt-dlp/blob/master/test/test_download.py#L39)

---

## Testing Best Practices Summary

1. **Every extractor needs tests** - At least one test case
2. **Test real content** - Use actual video URLs when possible
3. **Choose stable videos** - Official channels, popular content
4. **Test edge cases** - Playlists, age-restriction, geo-blocking
5. **Use `only_matching` liberally** - For URL format variations
6. **Keep tests fast** - Skip download when metadata is sufficient
7. **Document expected failures** - Use `'skip'` field with reason
8. **Update tests regularly** - Sites change, tests need maintenance
9. **Run locally before PR** - Catch issues early
10. **Be patient with flaky tests** - Network issues happen

---

## Next Steps

- [Development Guide](development.md) - Create your own extractor with tests
- [Python Features](python-features.md) - Learn testing utilities like `traverse_obj`
- [Architecture](architecture.md) - Understand what you're testing
