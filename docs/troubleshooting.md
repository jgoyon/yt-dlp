# Troubleshooting

Common issues and solutions when using yt-dlp.

## Table of Contents

- [Download Failures](#download-failures)
- [Extractor Errors](#extractor-errors)
- [Network Issues](#network-issues)
- [Post-Processing Problems](#post-processing-problems)
- [Debugging Techniques](#debugging-techniques)
- [Platform-Specific Issues](#platform-specific-issues)

---

## Download Failures

### HTTP 403 Forbidden

**Symptom**: `ERROR: unable to download video data: HTTP Error 403: Forbidden`

**Causes & Solutions**:

1. **Bot Detection**
   ```bash
   # Use browser impersonation
   yt-dlp --impersonate chrome110 'URL'

   # Or use cookies from browser
   yt-dlp --cookies-from-browser chrome 'URL'
   ```

2. **Geo-Restriction**
   ```bash
   # Use proxy
   yt-dlp --proxy socks5://127.0.0.1:1080 'URL'

   # Or geo-bypass
   yt-dlp --geo-bypass 'URL'
   ```

3. **Rate Limiting**
   ```bash
   # Add delays
   yt-dlp --sleep-interval 5 'URL'
   ```

### HTTP 401 Unauthorized

**Symptom**: `ERROR: HTTP Error 401: Unauthorized`

**Solution**: Requires authentication

```bash
# Use browser cookies
yt-dlp --cookies-from-browser chrome 'URL'

# Or username/password
yt-dlp -u USERNAME -p PASSWORD 'URL'

# Check if login worked
yt-dlp -v --cookies-from-browser chrome 'URL'
```

### Video Unavailable

**Symptom**: `ERROR: Video unavailable. This video has been removed`

**Possible reasons**:
- Video deleted or made private
- Geo-restricted content
- Age-restricted content
- Copyright takedown

**Solutions**:
```bash
# For age-restricted content
yt-dlp --cookies-from-browser chrome 'URL'

# For geo-restricted
yt-dlp --proxy PROXY_URL 'URL'

# Skip unavailable videos in playlist
yt-dlp --ignore-errors 'PLAYLIST_URL'
```

### Format Not Available

**Symptom**: `ERROR: Requested format not available`

**Solution**: Check available formats

```bash
# List formats
yt-dlp -F 'URL'

# Use more flexible format selection
yt-dlp -f 'bestvideo+bestaudio/best' 'URL'

# Or just get best available
yt-dlp 'URL'
```

---

## Extractor Errors

### Unable to Extract

**Symptom**: `ERROR: Unable to extract video data`

**Causes**:
1. Site changed their HTML/API
2. JavaScript protection
3. Cloudflare protection

**Solutions**:

```bash
# Update yt-dlp
yt-dlp -U

# Use cookies from browser (bypasses some protections)
yt-dlp --cookies-from-browser chrome 'URL'

# Try generic extractor
yt-dlp --force-generic-extractor 'URL'

# Increase verbosity to see what's failing
yt-dlp -v 'URL'
```

### Signature Extraction Failed (YouTube)

**Symptom**: `ERROR: Signature extraction failed`

**Solution**:

```bash
# Update yt-dlp
yt-dlp -U

# Force update extractors
python -m pip install --upgrade yt-dlp

# Use different player client
yt-dlp --extractor-args 'youtube:player_client=android' 'URL'
```

### JavaScript Interpreter Errors

**Symptom**: `ERROR: Could not interpret JavaScript`

**Solution**:

```bash
# Update yt-dlp
yt-dlp -U

# Try different extractor args
yt-dlp --extractor-args 'youtube:player_skip=js' 'URL'
```

---

## Network Issues

### Connection Timeout

**Symptom**: `ERROR: Connection timeout`

**Solutions**:

```bash
# Increase timeout
yt-dlp --socket-timeout 30 'URL'

# Retry with delays
yt-dlp --retries 10 --retry-sleep 5 'URL'

# Use different network backend
yt-dlp --downloader-args 'http:-T 60' 'URL'
```

### SSL Certificate Errors

**Symptom**: `ERROR: SSL: CERTIFICATE_VERIFY_FAILED`

**Solutions**:

```bash
# Update certificates
pip install --upgrade certifi

# As last resort (not recommended)
yt-dlp --no-check-certificates 'URL'
```

### Slow Download Speed

**Solutions**:

```bash
# Use aria2c for multi-threaded downloads
yt-dlp --external-downloader aria2c \
       --external-downloader-args '-x 16 -s 16' 'URL'

# Increase concurrent fragments
yt-dlp --concurrent-fragments 5 'URL'

# Check if rate limited
yt-dlp --limit-rate 10M 'URL'  # Remove limit to test
```

---

## Post-Processing Problems

### FFmpeg Not Found

**Symptom**: `ERROR: ffprobe/avprobe and ffmpeg/avconv not found`

**Solution**: Install FFmpeg

```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# macOS (Homebrew)
brew install ffmpeg

# Windows (Chocolatey)
choco install ffmpeg

# Or download from https://ffmpeg.org/download.html
```

### Merge Failed

**Symptom**: `ERROR: Unable to merge formats`

**Solutions**:

```bash
# Check FFmpeg is working
ffmpeg -version

# Try different merge format
yt-dlp --merge-output-format mkv 'URL'

# Skip merge and keep separate files
yt-dlp -f 'bestvideo,bestaudio' 'URL'
```

### Encoding Issues

**Symptom**: Characters not displaying correctly

**Solutions**:

```bash
# Set encoding explicitly
export PYTHONIOENCODING=utf-8
yt-dlp 'URL'

# Windows (PowerShell)
$env:PYTHONIOENCODING='utf-8'
yt-dlp 'URL'

# Restrict filenames to ASCII
yt-dlp --restrict-filenames 'URL'
```

---

## Debugging Techniques

### Verbose Output

```bash
# See what yt-dlp is doing
yt-dlp -v 'URL'

# Even more verbose
yt-dlp -vv 'URL'

# Print all traffic
yt-dlp --print-traffic 'URL'
```

### Save Pages for Inspection

```bash
# Save downloaded pages
yt-dlp --write-pages 'URL'

# Creates files like:
# - [id]-watch.html
# - [id]-player.js
```

### Test Without Downloading

```bash
# Simulate download
yt-dlp --simulate 'URL'

# Get JSON metadata
yt-dlp --dump-json 'URL'

# List formats without downloading
yt-dlp -F 'URL'
```

### Check Configuration

```bash
# Show config file locations
yt-dlp --config-locations

# Ignore config files
yt-dlp --ignore-config 'URL'
```

### Extractor-Specific Debugging

```bash
# YouTube: try different player
yt-dlp --extractor-args 'youtube:player_client=android' 'URL'

# Force IPv4
yt-dlp --force-ipv4 'URL'

# Force IPv6
yt-dlp --force-ipv6 'URL'
```

---

## Platform-Specific Issues

### Windows

**Path Length Issues**:
```bash
# Enable long paths in Windows 10+
# Or use shorter output template
yt-dlp -o "%(title).50B.%(ext)s" 'URL'
```

**Permission Errors**:
```bash
# Run as administrator or change download directory
yt-dlp -P C:\Users\USERNAME\Videos 'URL'
```

**Encoding Issues**:
```powershell
# Set UTF-8 encoding
$env:PYTHONIOENCODING='utf-8'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

### macOS

**SSL Issues**:
```bash
# Install certificates
/Applications/Python*/Install\ Certificates.command
```

**Permission Errors**:
```bash
# Check file permissions
chmod +x /path/to/yt-dlp
```

### Linux

**Missing Libraries**:
```bash
# Ubuntu/Debian
sudo apt install python3-pip python3-setuptools

# Arch Linux
sudo pacman -S python-pip

# For browser cookie extraction
sudo apt install python3-secretstorage python3-keyring
```

---

## Common Error Messages

### "This video is not available"
- Video deleted or private
- Geo-restricted
- Try: `--cookies-from-browser chrome` or `--proxy`

### "Sign in to confirm your age"
- Age-restricted content
- Try: `--cookies-from-browser chrome`

### "The uploader has not made this video available in your country"
- Geo-restriction
- Try: `--proxy socks5://127.0.0.1:1080` or VPN

### "Unsupported URL"
- URL pattern not recognized
- Try: `yt-dlp -U` (update)
- Or: `--force-generic-extractor`

### "Requested format not available"
- Format doesn't exist
- Try: `yt-dlp -F 'URL'` to list available
- Or: Use flexible format: `-f best`

---

## Getting Help

### Check Existing Issues

Search GitHub issues: https://github.com/yt-dlp/yt-dlp/issues

### Report a Bug

Include:
1. Full verbose output (`-vv`)
2. Full command used
3. yt-dlp version (`yt-dlp --version`)
4. Python version (`python --version`)
5. Operating system
6. URL (if public)

```bash
# Get all relevant info
yt-dlp --version
python --version
yt-dlp -vv 'URL' 2>&1 | tee debug.log
```

### Community Support

- **Discord**: https://discord.gg/H5MNcFW63r
- **GitHub Discussions**: https://github.com/yt-dlp/yt-dlp/discussions
- **Wiki**: https://github.com/yt-dlp/yt-dlp/wiki/FAQ

---

## Next Steps

- [CLI Usage](cli-usage.md) - Common usage patterns
- [Configuration](configuration.md) - Config file options
- [Architecture](architecture.md) - Understanding how it works
