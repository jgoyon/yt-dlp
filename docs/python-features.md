# Python Features & Coding Style

This document explores the Python patterns, features, and coding conventions used throughout the yt-dlp project.

## Table of Contents

- [Python Version & Requirements](#python-version--requirements)
- [Coding Style & Conventions](#coding-style--conventions)
- [Advanced Python Patterns](#advanced-python-patterns)
- [Functional Programming](#functional-programming)
- [Error Handling Philosophy](#error-handling-philosophy)
- [Data Structures & Utilities](#data-structures--utilities)
- [Import Organization](#import-organization)

---

## Python Version & Requirements

### Minimum Version: Python 3.10+

**Rationale**: Python 3.10 introduced several features used in yt-dlp:
- Pattern matching with `match`/`case` statements
- Better type hinting support
- Union types with `|` operator
- Improved error messages

### Key Dependencies

From `pyproject.toml`:

```toml
[project]
requires-python = ">=3.10"

dependencies = []  # No mandatory dependencies!

[project.optional-dependencies]
default = [
    "brotli; platform_python_implementation=='CPython'",
    "certifi",
    "mutagen",
    "pycryptodomex",
    "requests>=2.32.2,<3",
    "urllib3>=1.26.17,<3",
    "websockets>=13.0",
]
```

**Design Philosophy**: yt-dlp has **zero mandatory dependencies** for core functionality, relying primarily on the standard library. Optional dependencies enhance features but aren't required.

---

## Coding Style & Conventions

### PEP 8 Compliance

yt-dlp generally follows [PEP 8](https://pep8.org/), with some project-specific conventions.

**Linting Tools**:
- `ruff` - Fast Python linter
- `autopep8` - Auto-formatting

### Naming Conventions

#### Classes
```python
# PascalCase for class names
class InfoExtractor:
    pass

class YoutubeIE(InfoExtractor):
    pass

class YoutubeDL:
    pass
```

#### Functions & Methods
```python
# snake_case for functions and methods
def _real_extract(self, url):
    pass

def download_webpage(self, url, video_id):
    pass
```

#### Private vs Public
```python
# Leading underscore for "internal" methods
class InfoExtractor:
    def extract(self, url):
        """Public API"""
        return self._real_extract(url)

    def _real_extract(self, url):
        """Internal implementation - override in subclasses"""
        pass

    def _download_webpage(self, url, video_id):
        """Internal helper - not part of public API"""
        pass
```

#### Constants
```python
# UPPER_CASE for module-level constants
ENGLISH_MONTH_NAMES = [
    'January', 'February', 'March', 'April', 'May', 'June',
    'July', 'August', 'September', 'October', 'November', 'December'
]

DATE_FORMATS = (
    '%d %B %Y',
    '%d %b %Y',
    # ...
)
```

#### Extractor Naming
```python
# Pattern: [SiteName]IE for Info Extractor
class YoutubeIE(InfoExtractor):
    IE_NAME = 'youtube'  # Identifier
    IE_DESC = 'YouTube'  # Human-readable description

class VimeoIE(InfoExtractor):
    IE_NAME = 'vimeo'
    IE_DESC = 'Vimeo'
```

### Module Organization

**Import Order**:
```python
# 1. Standard library imports
import os
import sys
import re
import json
from collections import defaultdict

# 2. Third-party imports (if any)
import requests

# 3. Local application imports
from .utils import (
    ExtractorError,
    traverse_obj,
    unified_timestamp,
)
from .networking import Request
```

[Example](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/common.py#L1-L103)

### Docstrings

```python
class InfoExtractor:
    """Information Extractor class.

    Information extractors are classes that, given a URL, extract
    information about the video (or videos) the URL refers to. This
    information includes the real video URL, the video title, author and
    others. The information is stored in a dictionary which is then
    passed to the YoutubeDL.

    The type field determines the type of the result.
    """

    def _download_webpage(self, url, video_id):
        """Download webpage and return content as string

        Arguments:
        url -- URL to download
        video_id -- Video identifier for error messages

        Returns decoded webpage content
        """
```

---

## Advanced Python Patterns

### 1. Custom Descriptors & Properties

#### `classproperty` - Property for Class Methods

**Location**: `yt_dlp/utils/_utils.py:5028`

```python
class classproperty:
    """Property access for class methods with optional caching"""
    def __new__(cls, func=None, *args, **kwargs):
        if not func:
            return functools.partial(cls, *args, **kwargs)
        return super().__new__(cls)

    def __init__(self, func, *, cache=False):
        functools.update_wrapper(self, func)
        self.func = func
        self._cache = {} if cache else None

    def __get__(self, instance, owner):
        if self._cache is None:
            return self.func(owner)
        elif owner not in self._cache:
            self._cache[owner] = self.func(owner)
        return self._cache[owner]
```

**Usage Example**:
```python
# From yt_dlp/downloader/external.py:87
class ExternalFD(FileDownloader):
    @classproperty
    def EXE_NAME(cls):
        return cls.get_basename()

# Access without instantiation
aria2c_name = Aria2cFD.EXE_NAME
```

[View in code](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/downloader/external.py#L87-L89)

**Why it's useful**: Allows computing values at the class level without creating instances, useful for extracting metadata about extractors or downloaders.

---

### 2. Decorators

#### `@functools.cache` - Function Result Caching

```python
# From yt_dlp/utils/_utils.py:172
@functools.cache
def preferredencoding():
    """Get preferred encoding.

    Returns the best encoding scheme for the system, based on
    locale.getpreferredencoding() and some further tweaks.
    """
    try:
        pref = locale.getpreferredencoding()
        'TEST'.encode(pref)
    except Exception:
        pref = 'UTF-8'

    return pref
```

[View in code](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/utils/_utils.py#L172)

**Effect**: First call computes and caches result; subsequent calls return cached value instantly.

#### `@functools.cached_property` - Lazy Property Evaluation

```python
# From yt_dlp/downloader/external.py:91
@functools.cached_property
def exe(self):
    """Get external executable path"""
    return self.get_exe()
```

**Effect**: Property computed on first access, then cached for the instance lifetime.

#### Custom Decorators for Error Handling

```python
# From yt_dlp/YoutubeDL.py:185
def _catch_unsafe_extension_error(func):
    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):
        try:
            return func(self, *args, **kwargs)
        except _UnsafeExtensionError as error:
            self.report_error(
                f'The extracted extension ({error.extension!r}) is unusual '
                'and will be skipped for safety reasons. '
                f'If you believe this is an error{bug_reports_message(",")}')

    return wrapper
```

[View in code](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/YoutubeDL.py#L185)

**Pattern**: Wraps methods to catch specific exceptions and convert them to user-friendly error messages.

---

### 3. Context Managers

#### Using `contextlib.contextmanager`

```python
@contextlib.contextmanager
def locked_file(filename, mode='r', encoding=None):
    """Context manager for file locking"""
    import fcntl

    f = open(filename, mode, encoding=encoding)
    try:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)
        yield f
    finally:
        fcntl.flock(f.fileno(), fcntl.LOCK_UN)
        f.close()
```

**Usage**:
```python
with locked_file('archive.txt', 'a') as f:
    f.write(f'{video_id}\n')
```

#### Context Manager for Resource Cleanup

```python
# From download process
with open(filename, 'wb') as f:
    while True:
        chunk = response.read(10240)
        if not chunk:
            break
        f.write(chunk)
```

---

### 4. Generators & Lazy Evaluation

#### Playlist Iteration

```python
def _entries(self):
    """Generator yielding playlist entries"""
    page_num = 0
    while True:
        page = self._download_json(url, page_num)
        for item in page.get('items', []):
            yield self.url_result(item['url'], ie='Youtube')

        if not page.get('hasMore'):
            break
        page_num += 1
```

**Benefits**:
- Memory efficient for large playlists
- Allows early termination
- Lazy evaluation - only fetch pages as needed

#### Using `itertools` for Efficiency

```python
import itertools

# Chain multiple iterables
all_formats = itertools.chain(video_formats, audio_formats)

# Flatten nested lists
flat_list = itertools.chain.from_iterable(nested_lists)

# Create infinite sequences
counter = itertools.count(start=1)
```

---

### 5. Advanced Dictionary Operations

#### `traverse_obj` - Safe Nested Data Access

**Location**: `yt_dlp/utils/traversal.py:38`

This is one of the most important utilities in yt-dlp, providing safe navigation of nested data structures.

```python
def traverse_obj(
        obj, *paths, default=NO_DEFAULT, expected_type=None, get_all=True,
        casesense=True, is_user_input=NO_DEFAULT, traverse_string=False):
    """
    Safely traverse nested `dict`s and `Iterable`s

    >>> obj = [{}, {"key": "value"}]
    >>> traverse_obj(obj, (1, "key"))
    'value'

    Each of the provided `paths` is tested and the first producing
    a valid result will be returned.
    """
```

**Key Features**:
- Multiple path attempts
- Type validation
- Default values
- Case-insensitive keys
- Safe access (no KeyError or IndexError)

**Usage Examples**:

```python
# Simple key access
data = {'user': {'name': 'John', 'age': 30}}
name = traverse_obj(data, ('user', 'name'))
# Result: 'John'

# Try multiple paths
video_id = traverse_obj(data,
    ('video', 'id'),           # Try first path
    ('content', 'videoId'),    # Try second path
    default='unknown')         # Fallback value

# Filter by type
numbers = traverse_obj(data, (..., {int}))
# Returns all integer values from nested structure

# Access list items
items = [{'id': 1}, {'id': 2}, {'id': 3}]
ids = traverse_obj(items, (..., 'id'))
# Result: [1, 2, 3]

# Function-based filtering
adults = traverse_obj(users, (..., {lambda x: x.get('age', 0) >= 18}))
```

[View implementation](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/utils/traversal.py#L38)

**Why it's crucial**: Extractors parse JSON/HTML from thousands of sites with varying structures. `traverse_obj` provides robust, concise data extraction that handles missing keys gracefully.

**Real-world Example**:
```python
# From an actual extractor
video_info = self._download_json(api_url, video_id)

# Extract nested data safely
title = traverse_obj(video_info, ('data', 'attributes', 'title'))
duration = traverse_obj(video_info, ('data', 'attributes', 'duration'), expected_type=int)
formats = traverse_obj(video_info, ('data', 'relationships', 'mediaFiles', 'data', ..., 'attributes'))
```

---

### 6. Type Hints (Modern Python)

yt-dlp uses type hints for better IDE support and documentation:

```python
from typing import Any, Optional, Union

def _download_json(
    self,
    url: str,
    video_id: str,
    note: str = 'Downloading JSON metadata',
    fatal: bool = True,
    query: Optional[dict] = None
) -> Union[dict, list, None]:
    """Download and parse JSON"""
```

**Union types with `|` (Python 3.10+)**:
```python
def process_format(self, fmt: dict | None) -> dict | None:
    if fmt is None:
        return None
    return {'url': fmt['url']}
```

---

## Functional Programming

### List Comprehensions

```python
# Extract all video IDs
video_ids = [entry['id'] for entry in entries if entry.get('type') == 'video']

# Transform formats
formats = [{
    'url': fmt['url'],
    'height': fmt.get('height'),
    'tbr': fmt.get('bitrate', 0) / 1000,
} for fmt in raw_formats]
```

### Generator Expressions

```python
# More memory efficient for large datasets
total_size = sum(fmt.get('filesize', 0) for fmt in formats)

# Check if any format meets criteria
has_hd = any(fmt.get('height', 0) >= 720 for fmt in formats)
```

### `filter()` and `map()`

```python
# Filter formats by quality
hd_formats = filter(lambda f: f.get('height', 0) >= 720, formats)

# Transform list
urls = map(lambda fmt: fmt['url'], formats)
urls = list(urls)  # Convert to list if needed
```

### `functools.reduce()` for Aggregation

```python
from functools import reduce
import operator

# Merge multiple dictionaries
merged = reduce(operator.or_, [dict1, dict2, dict3], {})
```

### Lambda Functions

```python
# Sort formats by quality
formats.sort(key=lambda f: f.get('height', 0), reverse=True)

# Find best format
best = max(formats, key=lambda f: (f.get('height', 0), f.get('tbr', 0)))
```

---

## Error Handling Philosophy

### Exception Hierarchy

```python
class YoutubeDLError(Exception):
    """Base exception for yt-dlp"""
    pass

class ExtractorError(YoutubeDLError):
    """Error during extraction"""
    def __init__(self, msg, tb=None, expected=False, video_id=None):
        super().__init__(msg)
        self.expected = expected  # Whether error was anticipated
        self.video_id = video_id

class DownloadError(YoutubeDLError):
    """Error during download"""
    pass

class PostProcessingError(YoutubeDLError):
    """Error during post-processing"""
    pass

class GeoRestrictedError(ExtractorError):
    """Content is geo-restricted"""
    pass
```

### Expected vs Unexpected Errors

yt-dlp distinguishes between errors that are part of normal operation (e.g., video unavailable) and actual bugs:

```python
# Expected error - user-facing message
raise ExtractorError('This video is private', expected=True)

# Unexpected error - likely a bug
raise ExtractorError('Failed to parse API response')
```

### Try-Catch Patterns

#### Graceful Degradation

```python
# Try to get high-quality metadata, fall back to basic
try:
    metadata = self._download_json(api_url, video_id)
except ExtractorError:
    metadata = self._parse_html(webpage)
```

#### Multiple Fallback Attempts

```python
# Try multiple extraction methods
for attempt in [self._extract_api, self._extract_webpage, self._extract_embed]:
    try:
        return attempt(url, video_id)
    except ExtractorError:
        continue
raise ExtractorError('All extraction methods failed')
```

#### Context-Aware Error Messages

```python
try:
    video_url = info['url']
except KeyError:
    raise ExtractorError(
        f'Unable to extract video URL for {video_id}',
        video_id=video_id
    )
```

---

## Data Structures & Utilities

### NO_DEFAULT Sentinel

Instead of using `None` (which might be a valid value), yt-dlp uses a sentinel:

```python
# From yt_dlp/utils/_utils.py:61
class NO_DEFAULT:
    pass

def get_value(data, key, default=NO_DEFAULT):
    if key not in data:
        if default is NO_DEFAULT:
            raise KeyError(key)
        return default
    return data[key]
```

**Why**: Distinguishes between "not provided" and "explicitly None".

### LazyList - Lazy List Evaluation

```python
# From yt_dlp/utils/_utils.py
class LazyList(collections.abc.Sequence):
    """Lazy list that evaluates items on access"""

    def __init__(self, iterable):
        self._iterable = iterable
        self._cache = []
        self._exhausted = False

    def __getitem__(self, index):
        # Populate cache up to index
        while len(self._cache) <= index and not self._exhausted:
            try:
                self._cache.append(next(self._iterable))
            except StopIteration:
                self._exhausted = True
                raise IndexError
        return self._cache[index]
```

**Usage**: Represents potentially infinite sequences (like playlist pages) as a list.

### OrderedDict & orderedSet

```python
# Maintain insertion order while removing duplicates
def orderedSet(iterable):
    """Return list with duplicates removed, preserving order"""
    return list(dict.fromkeys(iterable))

# Example
formats = orderedSet([fmt1, fmt1, fmt2, fmt3, fmt2])
# Result: [fmt1, fmt2, fmt3]
```

---

## Import Organization

### Conditional Imports

```python
# Import only when needed
def _extract_with_phantomjs(self, url):
    from .extractor.openload import PhantomJSwrapper
    phantom = PhantomJSwrapper(self)
    return phantom.extract(url)
```

**Why**: Reduces startup time by not importing heavy dependencies unless required.

### Import Aliases

```python
# Shorten long module names
from ..networking import HEADRequest, Request
from ..networking.exceptions import HTTPError, network_exceptions

# Use aliasing for clarity
from ..compat import urllib_req_to_req as compat_urllib_request
```

### Lazy Module Loading

yt-dlp can generate "lazy extractors" to improve startup performance:

```bash
python devscripts/make_lazy_extractors.py
```

This creates wrapper functions that import extractors only when first used:

```python
# Generated lazy extractor
def YoutubeIE(*args, **kwargs):
    from .youtube import YoutubeIE as real_class
    return real_class(*args, **kwargs)
```

---

## Code Quality Tools

### Linting with Ruff

```bash
# Check code
ruff check yt_dlp/

# Auto-fix issues
ruff check --fix yt_dlp/
```

### Formatting with autopep8

```bash
# Format code to PEP 8
autopep8 --in-place --recursive yt_dlp/
```

### Running Tests

```bash
# Run all tests
python -m pytest test/

# Run specific test
python test/test_download.py TestDownload.test_youtube

# Run with verbose output
python -m pytest -v test/
```

---

## Best Practices Summary

1. **No mandatory dependencies** - Rely on standard library when possible
2. **Graceful degradation** - Always have fallback strategies
3. **Safe data access** - Use `traverse_obj` for nested structures
4. **Type hints** - Document expected types
5. **Lazy evaluation** - Use generators for memory efficiency
6. **Descriptive errors** - Help users understand what went wrong
7. **Consistent naming** - Follow established conventions
8. **Functional style** - Prefer comprehensions and pure functions
9. **Context managers** - Ensure resource cleanup
10. **Test coverage** - Every extractor has tests

---

## Next Steps

- [Testing Methodology](testing.md) - Learn how yt-dlp tests complex code
- [Architecture](architecture.md) - See how components fit together
- [Development Guide](development.md) - Apply these patterns to new extractors
