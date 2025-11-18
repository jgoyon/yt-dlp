# Configuration

Configuration file formats, locations, and options for yt-dlp.

## Table of Contents

- [Configuration File Locations](#configuration-file-locations)
- [Configuration Syntax](#configuration-syntax)
- [Common Configurations](#common-configurations)
- [Extractor-Specific Options](#extractor-specific-options)
- [Environment Variables](#environment-variables)

---

## Configuration File Locations

yt-dlp searches for configuration files in this order:

### 1. Portable Configuration
**Location**: Same directory as yt-dlp executable
- `yt-dlp.conf`

**Use case**: Portable installations on USB drives

### 2. Home Directory (if `--paths home=PATH` is set)
**Location**: Specified home path
- `PATH/yt-dlp.conf`

### 3. User Configuration
**Linux/macOS**:
- `~/.config/yt-dlp/config`
- `~/.config/yt-dlp/config.txt`
- `~/.config/yt-dlp.conf`

**Windows**:
- `%APPDATA%\yt-dlp\config`
- `%APPDATA%\yt-dlp\config.txt`

### 4. System Configuration
**Linux**:
- `/etc/yt-dlp.conf`
- `/etc/yt-dlp/config`

**macOS**:
- `/Library/Preferences/yt-dlp.conf`
- `/Library/Application Support/yt-dlp/config`

**Windows**:
- `C:\ProgramData\yt-dlp\config`

### Priority
Later configurations override earlier ones:
1. Portable
2. Home
3. User
4. System
5. Command-line arguments (highest priority)

---

## Configuration Syntax

### Basic Format

```
# Comments start with #
# One option per line
# Do NOT include leading dashes

# Good
format bestvideo+bestaudio

# Bad (don't do this)
--format bestvideo+bestaudio
```

### Options with Values

```
# Simple value
output ~/Videos/%(title)s.%(ext)s

# Value with spaces (use quotes)
output "~/My Videos/%(title)s.%(ext)s"

# Multiple values
sub-langs en,es,fr

# Boolean flags (no value)
embed-thumbnail
embed-metadata
```

### Multi-line Options

```
# Long format specification
format "bestvideo[height<=1080][ext=mp4]+bestaudio[ext=m4a]/best"

# Or split logically
format bestvideo[height<=1080]
format +bestaudio[ext=m4a]/best
```

---

## Common Configurations

### Quality Defaults

```
# Best quality up to 1080p
format bestvideo[height<=1080]+bestaudio/best

# Prefer MP4
format bestvideo[ext=mp4]+bestaudio[ext=m4a]/best

# Audio only
format bestaudio
extract-audio
audio-format mp3
audio-quality 0
```

### Output Organization

```
# Organized by uploader
output ~/Videos/%(uploader)s/%(title)s [%(id)s].%(ext)s

# Organized by date
output ~/Videos/%(upload_date>%Y-%m-%d)s - %(title)s.%(ext)s

# Playlist organization
output ~/Videos/%(playlist)s/%(playlist_index)02d - %(title)s.%(ext)s

# Restrict filename length
output "%(title).200B.%(ext)s"
```

### Metadata & Extras

```
# Embed everything
embed-thumbnail
embed-metadata
embed-subs
embed-chapters

# Download auxiliary files
write-description
write-info-json
write-annotations
write-thumbnail
write-subs
all-subs
sub-langs en,es,fr,de
```

### Download Management

```
# Use archive
download-archive ~/Videos/archive.txt

# Rate limiting
limit-rate 5M
sleep-interval 3
max-sleep-interval 10

# Concurrent downloads
concurrent-fragments 5

# Use external downloader
external-downloader aria2c
external-downloader-args "aria2c:-x 16 -s 16 -k 1M"
```

### Authentication

```
# Browser cookies
cookies-from-browser chrome

# Or specific profile
cookies-from-browser "chrome:Profile 1"

# Netrc for credentials
netrc
```

### Post-Processing

```
# Convert thumbnails
convert-thumbnails jpg

# Recode videos
recode-video mp4

# FFmpeg arguments
postprocessor-args "ffmpeg:-c:a aac -b:a 192k"

# SponsorBlock
sponsorblock-mark all
sponsorblock-remove sponsor,intro,outro
```

---

## Extractor-Specific Options

Use `--extractor-args` for site-specific options.

### YouTube

```
# Skip DASH manifest
extractor-args youtube:skip=dash

# Use specific player client
extractor-args youtube:player_client=android,web

# Get HLS manifest instead
extractor-args youtube:player_skip=configs,webpage
```

### Twitch

```
# Disable ads
extractor-args twitch:api_key=YOUR_KEY
```

### TikTok

```
# Use specific device ID
extractor-args tiktok:device_id=DEVICE_ID

# Use web API instead of mobile
extractor-args tiktok:api=web
```

### Multiple Extractors

```
# Separate with semicolon
extractor-args "youtube:skip=dash;twitch:api_key=KEY"
```

---

## Environment Variables

### Download Paths

```bash
# Set download directory
export YTDL_OUTPUT_PATH="~/Videos/%(title)s.%(ext)s"
```

### Proxy Settings

```bash
# HTTP proxy
export HTTP_PROXY="http://proxy.example.com:8080"
export HTTPS_PROXY="http://proxy.example.com:8080"

# SOCKS proxy
export ALL_PROXY="socks5://127.0.0.1:1080"
```

### Python-Specific

```bash
# Force specific Python encoding
export PYTHONIOENCODING="utf-8"

# Python unbuffered output
export PYTHONUNBUFFERED=1
```

---

## Complete Example Configurations

### Archiver Configuration

**Use case**: Archive channel content

```
# ~/.config/yt-dlp/config

# Quality: Best up to 1080p, prefer MP4
format bestvideo[height<=1080][ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]/best

# Output organization
output ~/Archive/%(channel)s/%(upload_date)s - %(title)s [%(id)s].%(ext)s

# Archive tracking
download-archive ~/Archive/archive.txt

# Metadata
embed-thumbnail
embed-metadata
write-description
write-info-json
write-subs
all-subs
sub-langs en

# Rate limiting (be nice to servers)
limit-rate 5M
sleep-interval 5
max-sleep-interval 15

# Authentication
cookies-from-browser chrome

# Ignore errors and continue
ignore-errors
no-abort-on-error
```

### Music Downloader Configuration

**Use case**: Download music playlists

```
# Audio only
extract-audio
audio-format mp3
audio-quality 0

# Output with metadata
output ~/Music/%(artist)s/%(album)s/%(track_number)02d - %(track)s.%(ext)s

# Parse metadata from title
parse-metadata "title:%(artist)s - %(track)s"

# Embed artwork
embed-thumbnail
embed-metadata

# No video files
no-write-thumbnail
format bestaudio/best
```

### Streamer Configuration

**Use case**: Record live streams

```
# Wait for live stream
wait-for-video 60
retry-sleep 30

# Live stream options
live-from-start
hls-use-mpegts

# Output
output ~/Streams/%(uploader)s/%(upload_date)s - %(title)s.%(ext)s

# Continue on error
ignore-errors
no-abort-on-error

# No post-processing (for speed)
no-post-overwrites
```

### Minimal Configuration

**Use case**: Fast, simple downloads

```
# Best quality, fast
format best

# Simple output
output ~/Downloads/%(title)s.%(ext)s

# Use aria2c for speed
external-downloader aria2c
external-downloader-args "aria2c:-x 16 -s 16"

# No extras
no-write-thumbnail
no-write-description
```

---

## Option Reference

### Format Selection
```
format FORMAT
format-sort FIELD[:DIR]
merge-output-format FORMAT
```

### Output
```
output TEMPLATE
output-na-placeholder TEXT
restrict-filenames
no-overwrites
continue
no-continue
no-part
```

### Download
```
concurrent-fragments N
rate-limit RATE
retries N
fragment-retries N
skip-unavailable-fragments
external-downloader CMD
external-downloader-args ARGS
```

### Filesystem
```
batch-file FILE
download-archive FILE
no-download-archive
paths KEY:PATH
```

### Verbosity
```
quiet
no-warnings
simulate
skip-download
print-json
dump-json
```

### Authentication
```
username USERNAME
password PASSWORD
twofactor TWOFACTOR
netrc
cookies FILE
cookies-from-browser BROWSER[:PROFILE]
```

### Post-Processing
```
extract-audio
audio-format FORMAT
audio-quality QUALITY
recode-video FORMAT
remux-video FORMAT
postprocessor-args NAME:ARGS
embed-thumbnail
embed-subs
embed-metadata
embed-chapters
```

### SponsorBlock
```
sponsorblock-mark CATS
sponsorblock-remove CATS
sponsorblock-chapter-title TEMPLATE
```

---

## Testing Your Configuration

```bash
# Show effective configuration
yt-dlp --config-locations

# Ignore all config files
yt-dlp --ignore-config 'URL'

# Verbose output shows options used
yt-dlp -v 'URL'

# Simulate to test without downloading
yt-dlp --simulate 'URL'
```

---

## Next Steps

- [CLI Usage](cli-usage.md) - Command-line patterns
- [Troubleshooting](troubleshooting.md) - Common config issues
- [Features](features.md) - All available options
