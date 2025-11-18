# CLI Usage Patterns

Common command-line workflows and usage patterns for yt-dlp.

## Table of Contents

- [Basic Usage](#basic-usage)
- [Format Selection](#format-selection)
- [Output Templates](#output-templates)
- [Batch Operations](#batch-operations)
- [Authentication](#authentication)
- [Advanced Workflows](#advanced-workflows)
- [Integration with Other Tools](#integration-with-other-tools)

---

## Basic Usage

### Download Single Video

```bash
# Download best quality
yt-dlp 'https://youtube.com/watch?v=VIDEO_ID'

# Download to specific directory
yt-dlp -P /path/to/downloads 'URL'

# Download with custom filename
yt-dlp -o '%(title)s.%(ext)s' 'URL'
```

### Download Playlist

```bash
# Download entire playlist
yt-dlp 'https://youtube.com/playlist?list=PLAYLIST_ID'

# Download first 10 videos
yt-dlp --playlist-end 10 'PLAYLIST_URL'

# Download videos 5-15
yt-dlp --playlist-start 5 --playlist-end 15 'PLAYLIST_URL'

# Reverse playlist order
yt-dlp --playlist-reverse 'PLAYLIST_URL'
```

### Download Channel

```bash
# Download all videos from channel
yt-dlp 'https://youtube.com/@ChannelName/videos'

# Download only recent uploads
yt-dlp --dateafter 20240101 'CHANNEL_URL'
```

---

## Format Selection

### Quality Selection

```bash
# Best quality (video+audio)
yt-dlp -f 'bestvideo+bestaudio/best' 'URL'

# Best quality up to 1080p
yt-dlp -f 'bestvideo[height<=1080]+bestaudio/best' 'URL'

# Best quality with specific codec
yt-dlp -f 'bestvideo[vcodec^=av01]+bestaudio' 'URL'

# Worst quality (for testing)
yt-dlp -f 'worstvideo+worstaudio/worst' 'URL'
```

### Audio-Only Downloads

```bash
# Extract best audio
yt-dlp -f 'bestaudio' -x 'URL'

# Extract and convert to MP3
yt-dlp -f 'bestaudio' -x --audio-format mp3 'URL'

# Specific audio quality
yt-dlp -f 'bestaudio[abr<=128]' -x 'URL'
```

### List Available Formats

```bash
# Show all available formats
yt-dlp -F 'URL'

# Example output:
# ID  EXT   RESOLUTION FPS │ FILESIZE   TBR PROTO │ VCODEC        VBR ACODEC      ABR
# ────────────────────────────────────────────────────────────────────────────────────
# 140 m4a   audio only     │   128k m3u8  │ audio only        mp4a.40.2  128k
# 298 mp4   1280x720   60  │  1.87G 3055k https │ h264         3055k video only
```

### Format Filtering

```bash
# MP4 format only
yt-dlp -f 'bestvideo[ext=mp4]+bestaudio[ext=m4a]' 'URL'

# Exclude certain formats
yt-dlp -f 'bestvideo[ext!=webm]+bestaudio' 'URL'

# Filesize constraint
yt-dlp -f 'best[filesize<50M]' 'URL'

# Frame rate preference
yt-dlp -f 'bestvideo[fps<=30]+bestaudio' 'URL'
```

---

## Output Templates

### Template Syntax

```bash
# Basic template
yt-dlp -o '%(title)s.%(ext)s' 'URL'

# Include uploader
yt-dlp -o '%(uploader)s - %(title)s.%(ext)s' 'URL'

# Include date
yt-dlp -o '%(upload_date)s - %(title)s.%(ext)s' 'URL'

# Organize by uploader
yt-dlp -o '%(uploader)s/%(title)s.%(ext)s' 'URL'
```

### Available Fields

Common template fields:
- `%(title)s` - Video title
- `%(id)s` - Video ID
- `%(ext)s` - File extension
- `%(uploader)s` - Uploader name
- `%(upload_date)s` - Upload date (YYYYMMDD)
- `%(duration)s` - Duration in seconds
- `%(view_count)s` - View count
- `%(like_count)s` - Like count
- `%(playlist)s` - Playlist name
- `%(playlist_index)s` - Video position in playlist
- `%(channel)s` - Channel name
- `%(resolution)s` - Resolution (e.g., 1920x1080)

### Advanced Templates

```bash
# Playlist with numbering
yt-dlp -o '%(playlist)s/%(playlist_index)02d - %(title)s.%(ext)s' 'PLAYLIST_URL'

# Conditional formatting
yt-dlp -o '%(uploader)s/%(title)s [%(id)s].%(ext)s' 'URL'

# Sanitized filenames
yt-dlp -o '%(title).200B.%(ext)s' 'URL'  # Limit to 200 bytes

# Include quality in filename
yt-dlp -o '%(title)s [%(resolution)s].%(ext)s' 'URL'
```

---

## Batch Operations

### Download from File

```bash
# Create list of URLs
cat > urls.txt <<EOF
https://youtube.com/watch?v=VIDEO1
https://youtube.com/watch?v=VIDEO2
https://youtube.com/watch?v=VIDEO3
EOF

# Download all
yt-dlp -a urls.txt

# Download with format
yt-dlp -a urls.txt -f 'bestvideo+bestaudio'
```

### Archive Management

```bash
# Skip already downloaded videos
yt-dlp --download-archive archive.txt 'CHANNEL_URL'

# How it works:
# - Creates archive.txt if doesn't exist
# - Records video IDs of downloaded videos
# - Skips videos found in archive on subsequent runs
```

**Archive file format**:
```
youtube VIDEO_ID1
youtube VIDEO_ID2
vimeo 12345678
```

### Resume Interrupted Downloads

```bash
# Resume automatically (default behavior)
yt-dlp -c 'URL'

# Force re-download
yt-dlp --no-continue 'URL'
```

---

## Authentication

### Cookies from Browser

```bash
# Use cookies from Chrome
yt-dlp --cookies-from-browser chrome 'URL'

# Use cookies from Firefox
yt-dlp --cookies-from-browser firefox 'URL'

# Specify profile
yt-dlp --cookies-from-browser 'chrome:Profile 1' 'URL'
```

**Supported browsers**:
- chrome, chromium
- brave
- edge
- firefox
- opera
- safari
- vivaldi

### Username/Password

```bash
# Provide credentials directly
yt-dlp -u USERNAME -p PASSWORD 'URL'

# Prompt for password
yt-dlp -u USERNAME 'URL'  # Will prompt

# Use netrc
yt-dlp -n 'URL'  # Uses ~/.netrc
```

**netrc format** (`~/.netrc`):
```
machine youtube.com
login myemail@example.com
password mypassword

machine twitch.tv
login myusername
password mypassword
```

### Two-Factor Authentication

```bash
# Some sites support 2FA
yt-dlp -u USERNAME -p PASSWORD --twofactor CODE 'URL'
```

---

## Advanced Workflows

### Geo-Restriction Bypass

```bash
# Use proxy
yt-dlp --proxy 'socks5://127.0.0.1:1080' 'URL'

# Use geo-verification proxy
yt-dlp --geo-verification-proxy 'PROXY' 'URL'

# Fake IP address
yt-dlp --xff 'GB' 'URL'  # Pretend to be from Great Britain
```

### Rate Limiting

```bash
# Limit download speed
yt-dlp --limit-rate 1M 'URL'  # 1 MB/s

# Add sleep interval between downloads
yt-dlp --sleep-interval 5 'PLAYLIST_URL'  # 5 seconds

# Random sleep
yt-dlp --min-sleep-interval 2 --max-sleep-interval 10 'PLAYLIST_URL'
```

### Subtitle Download

```bash
# Download all subtitles
yt-dlp --write-subs --all-subs 'URL'

# Download auto-generated subs
yt-dlp --write-auto-subs 'URL'

# Specific language
yt-dlp --write-subs --sub-langs en,es,fr 'URL'

# Embed subtitles in video
yt-dlp --write-subs --embed-subs 'URL'

# Convert format
yt-dlp --write-subs --sub-format srt 'URL'
```

### Thumbnail Download

```bash
# Download thumbnail
yt-dlp --write-thumbnail 'URL'

# Embed thumbnail in file
yt-dlp --embed-thumbnail 'URL'

# Convert thumbnail format
yt-dlp --convert-thumbnails jpg 'URL'
```

### Metadata Embedding

```bash
# Embed metadata
yt-dlp --embed-metadata 'URL'

# Embed chapters
yt-dlp --embed-chapters 'URL'

# Add metadata via template
yt-dlp --parse-metadata 'title:%(artist)s - %(track)s' 'URL'
```

### SponsorBlock Integration

```bash
# Remove sponsored segments
yt-dlp --sponsorblock-remove sponsor 'URL'

# Remove all SponsorBlock categories
yt-dlp --sponsorblock-remove all 'URL'

# Mark chapters but don't remove
yt-dlp --sponsorblock-mark all 'URL'
```

---

## Integration with Other Tools

### Pipe to FFmpeg

```bash
# Custom FFmpeg processing
yt-dlp -f best 'URL' -o - | ffmpeg -i - -c:v libx264 output.mp4
```

### Use External Downloader

```bash
# Use aria2c (faster for fragments)
yt-dlp --external-downloader aria2c 'URL'

# Use wget
yt-dlp --external-downloader wget 'URL'

# With aria2c options
yt-dlp --external-downloader aria2c \
       --external-downloader-args '-x 16 -s 16 -k 1M' 'URL'
```

### Post-Processing

```bash
# Convert to MP4 after download
yt-dlp --recode-video mp4 'URL'

# Extract audio and convert
yt-dlp -x --audio-format mp3 --audio-quality 0 'URL'

# Merge format after download
yt-dlp --merge-output-format mkv 'URL'
```

### Scripting

```bash
# Get JSON metadata without downloading
yt-dlp --dump-json --skip-download 'URL'

# Get direct video URL
yt-dlp -g 'URL'

# Get video title
yt-dlp --get-title 'URL'

# Get video ID
yt-dlp --get-id 'URL'

# Use in scripts
video_url=$(yt-dlp -g 'URL')
mpv "$video_url"
```

---

## Common Workflows

### Archive Entire Channel

```bash
# Complete channel backup
yt-dlp \
  --download-archive archive.txt \
  -o '%(channel)s/%(upload_date)s - %(title)s [%(id)s].%(ext)s' \
  --embed-thumbnail \
  --embed-metadata \
  --write-description \
  --write-info-json \
  --write-subs --all-subs \
  'CHANNEL_URL'
```

### Music Download

```bash
# Download music playlist
yt-dlp \
  -x --audio-format mp3 --audio-quality 0 \
  --embed-thumbnail \
  --embed-metadata \
  --parse-metadata 'title:%(artist)s - %(track)s' \
  -o '%(artist)s/%(album)s/%(track_number)02d - %(track)s.%(ext)s' \
  'PLAYLIST_URL'
```

### Video Collection

```bash
# Best quality video collection
yt-dlp \
  -f 'bestvideo[ext=mp4][height<=1080]+bestaudio[ext=m4a]/best[ext=mp4]' \
  --merge-output-format mp4 \
  --embed-thumbnail \
  --embed-subs \
  -o '%(uploader)s/%(title)s [%(id)s].%(ext)s' \
  -a urls.txt
```

### Live Stream Recording

```bash
# Record live stream
yt-dlp --wait-for-video 60 'LIVESTREAM_URL'

# Monitor for live and download
yt-dlp --wait-for-video 60 --retry-sleep 30 'CHANNEL_URL'
```

---

## Configuration File

Save common options in config file:

**Location**: `~/.config/yt-dlp/config` or `%APPDATA%\yt-dlp\config` (Windows)

**Example**:
```
# Default format
-f bestvideo[height<=1080]+bestaudio/best

# Output template
-o ~/Videos/%(uploader)s/%(title)s [%(id)s].%(ext)s

# Always embed metadata
--embed-thumbnail
--embed-metadata

# Subtitle defaults
--write-subs
--sub-langs en,es

# Use archive
--download-archive ~/Videos/archive.txt

# Rate limiting
--limit-rate 5M

# Cookies from browser
--cookies-from-browser chrome
```

See [Configuration](configuration.md) for details.

---

## Next Steps

- [Configuration](configuration.md) - Config file formats and options
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
- [Features](features.md) - Comprehensive feature list
