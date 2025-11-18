# CDA Extractor

Custom encryption schemes and proprietary decryption algorithms.

## Overview

**File**: `yt_dlp/extractor/cda.py`
**Complexity**: Medium-Complex
**Key Technique**: Custom video URL decryption using character shifting

[View source](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/cda.py)

---

## The Encryption Problem

CDA.pl (Polish video hosting) encrypts video URLs:

**Encrypted**:
```
_XDDD_https://video123_CDA_pl/video456_LOL_.mp4_ENDXDDD_
```

**Decrypted**:
```
https://video123.cda.pl/video456.mp4
```

**Challenge**: Reverse-engineer decryption algorithm.

---

## Custom Decryption Algorithm

```python
def decrypt_file(file_str):
    """Decrypt CDA's proprietary URL encryption"""
    # Step 1: Remove magic strings
    for magic in ['_XDDD_', '_CDA_', '_ADC_', '_LOL_', '_ENDXDDD_']:
        file_str = file_str.replace(magic, '')

    # Step 2: Character shifting cipher
    decrypted = ''
    for char in file_str:
        if char.isalpha():
            # Shift character by 14 positions in printable ASCII
            shifted = chr(33 + (ord(char) - 33 + 14) % 94)
            decrypted += shifted
        else:
            decrypted += char

    # Step 3: Domain replacements
    decrypted = decrypted.replace('_', '.')

    return decrypted
```

**Algorithm steps**:
1. Remove marker strings (`_XDDD_`, etc.)
2. Apply character-shift cipher (ROT-like)
3. Replace underscores with dots

---

## ROT13 Encoding

CDA also uses ROT13 for some data:

```python
import codecs

def decode_rot13(encoded_str):
    """Decode ROT13-encoded string"""
    return codecs.decode(encoded_str, 'rot_13')

# Example
encoded = "uggcf://ivqrb.pqn.cy/ivqrb.zc4"
decoded = decode_rot13(encoded)
# Result: "https://video.cda.pl/video.mp4"
```

**ROT13**: Simple letter substitution cipher (A→N, B→O, etc.)

---

## HMAC Password Hashing

For age-restricted content:

```python
import hmac
import hashlib

def generate_age_confirmation(video_id, password):
    """Generate age confirmation token"""
    # Hash password with HMAC-SHA256
    token = hmac.new(
        video_id.encode(),
        password.encode(),
        hashlib.sha256
    ).hexdigest()

    return token
```

---

## Multi-Quality Fetching

```python
def _real_extract(self, url):
    video_id = self._match_id(url)

    # Fetch video page
    webpage = self._download_webpage(url, video_id)

    # Find encrypted URLs for each quality
    qualities = {}
    for quality_match in re.finditer(
        r'quality:\s*"([^"]+)",\s*file:\s*"([^"]+)"',
        webpage):

        quality = quality_match.group(1)
        encrypted_url = quality_match.group(2)

        # Decrypt URL
        decrypted_url = self.decrypt_file(encrypted_url)
        qualities[quality] = decrypted_url

    # Build format list
    formats = []
    for quality, url in qualities.items():
        height = int_or_none(quality.rstrip('p'))
        formats.append({
            'url': url,
            'format_id': quality,
            'height': height,
        })

    return {
        'id': video_id,
        'formats': formats,
    }
```

---

## Bearer Token Caching

```python
_BEARER_TOKEN = None

def _get_bearer_token(self):
    """Fetch and cache bearer token"""
    if self._BEARER_TOKEN:
        return self._BEARER_TOKEN

    # Request token
    token_response = self._download_json(
        'https://api.cda.pl/oauth/token',
        None, 'Fetching bearer token')

    self._BEARER_TOKEN = token_response['access_token']
    return self._BEARER_TOKEN
```

---

## Android Device Spoofing

```python
# Random Android device selection
_ANDROID_DEVICES = [
    'SM-G9900',   # Samsung Galaxy S21
    'Pixel 5',    # Google Pixel 5
    'M2101K6G',   # Xiaomi Mi 11
]

def _get_user_agent(self):
    """Generate Android user-agent"""
    device = random.choice(self._ANDROID_DEVICES)
    return f'Mozilla/5.0 (Linux; Android 11; {device})'
```

---

## Key Takeaways

1. **Custom encryption** - Proprietary URL obfuscation
2. **Character shifting** - ROT-like cipher implementation
3. **ROT13 decoding** - Standard substitution cipher
4. **HMAC hashing** - Secure password verification
5. **Multi-quality support** - Decrypt multiple quality URLs

**Learning points**:
- Reverse-engineer obfuscation schemes
- Implement custom decryption
- Handle region-specific encryption

**Ethical note**: Only decrypt your own content or with permission.

---

[← HotStar](08-hotstar.md) | [Next: Kaltura →](10-kaltura.md)
