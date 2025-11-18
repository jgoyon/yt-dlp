# Performance Optimization

Techniques to optimize yt-dlp download and extraction performance.

## Download Performance

### External Downloaders

**aria2c** - Multi-connection downloads:
```bash
# Install aria2c
sudo apt install aria2  # Linux
brew install aria2      # macOS

# Use with yt-dlp
yt-dlp --external-downloader aria2c \
       --external-downloader-args '-x 16 -s 16 -k 1M' 'URL'
```

**Benefits**: 5-10x faster for fragmented downloads

### Concurrent Fragments

```bash
# Download multiple fragments simultaneously
yt-dlp --concurrent-fragments 5 'URL'

# Higher values for fast connections
yt-dlp --concurrent-fragments 10 'URL'
```

### Network Optimization

```bash
# Increase buffer size
yt-dlp --buffer-size 64K 'URL'

# HTTP/2 support (with requests backend)
yt-dlp --prefer-free-formats 'URL'
```

## Extractor Performance

### Lazy Loading

yt-dlp supports lazy extractor loading:

```bash
# Generate lazy extractors
python devscripts/make_lazy_extractors.py
```

**Impact**: Reduces startup time from ~2s to ~0.3s

### Skip Unnecessary Extraction

```bash
# Metadata only (no format extraction)
yt-dlp --skip-download --write-info-json 'URL'

# Flat extraction for playlists (faster)
yt-dlp --flat-playlist 'URL'
```

## Memory Management

### Streaming Downloads

yt-dlp streams data to disk - no memory issues with large files.

### Large Playlists

```bash
# Process playlist incrementally
yt-dlp --playlist-start 1 --playlist-end 100 'URL'

# Or use flat extraction
yt-dlp --flat-playlist --dump-json 'URL'
```

## Caching Strategies

### HTTP Cache

```bash
# Cache HTTP requests
yt-dlp --cache-dir ~/.cache/yt-dlp 'URL'
```

### Archive Files

```bash
# Skip previously downloaded
yt-dlp --download-archive archive.txt 'URL'
```

**Impact**: Instant skip of known videos

### OAuth Token Caching

Automatic for extractors using OAuth (Vimeo, etc.)

## Batch Operation Optimization

### Parallel Downloads

```bash
# Download multiple URLs in parallel
cat urls.txt | xargs -P 4 -I {} yt-dlp {}
```

### Rate Limiting Trade-offs

```bash
# Faster but may trigger rate limits
yt-dlp --no-sleep-interval 'URL'

# Balanced approach
yt-dlp --sleep-interval 1 'URL'
```

## Post-Processing Optimization

### Skip Unnecessary PP

```bash
# Don't remux if not needed
yt-dlp --no-post-overwrites 'URL'

# Skip thumbnail embedding for speed
yt-dlp --no-embed-thumbnail 'URL'
```

### FFmpeg Optimization

```bash
# Use hardware acceleration
yt-dlp --postprocessor-args 'ffmpeg:-hwaccel cuda' 'URL'

# Faster preset
yt-dlp --postprocessor-args 'ffmpeg:-preset ultrafast' 'URL'
```

## Configuration for Maximum Speed

```
# ~/.config/yt-dlp/config

# External downloader
external-downloader aria2c
external-downloader-args "aria2c:-x 16 -s 16 -k 1M"

# Concurrent downloads
concurrent-fragments 8

# No delays
no-sleep-interval

# Skip unnecessary operations
no-post-overwrites
no-write-thumbnail

# Format preference (faster to merge)
format "bestvideo[ext=mp4]+bestaudio[ext=m4a]/best"
```

---

[← Back to Documentation](README.md)
