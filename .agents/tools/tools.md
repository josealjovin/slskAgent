# Pipeline Tools

External tools and Python packages used by this pipeline.

## Python Packages

### spotipyfree

**Package:** `spotipyFree` (pip) — no credentials required for public playlists
**Import:** `from SpotipyFree import Spotify`
**Install:** `python3 -m pip install --break-system-packages spotipyfree`

Used by: `/spotify-playlist-export`

### spotipy

**Package:** `spotipy` (pip) — requires Spotify Developer credentials (Premium account owner)
**Import:** `from spotipy import Spotify`
**Install:** `python3 -m pip install --break-system-packages spotipy`
**Env vars:** `SPOTIPY_CLIENT_ID`, `SPOTIPY_CLIENT_SECRET`

Used by: `/spotify-playlist-export` (when credentials are available)

---

## External Tools

### ffmpeg

**Command:** `ffmpeg`
**Install:** `brew install ffmpeg`
**Purpose:** FLAC → MP3 320kbps encoding

Used by: `/flac-to-mp3`

Encoding flags:
```
ffmpeg -y -i input.flac -acodec libmp3lame -q:a 0 -b:a 320k -map_metadata 0 output.mp3
```

| Flag | Value | Purpose |
|------|-------|---------|
| `-acodec libmp3lame` | LAME encoder | MP3 codec |
| `-q:a 0` | V0 VBR | Transparent quality (~245kbps) |
| `-b:a 320k` | CBR 320kbps | Maximum quality constant bitrate |
| `-map_metadata 0` | inherit all | Copy FLAC ID3 tags to MP3 |
| `-y` | overwrite | Don't prompt for existing files |

---

## Environment Variables

| Variable | Required | Tool | Notes |
|----------|----------|------|-------|
| `SPOTIPY_CLIENT_ID` | No* | spotipy | From developer.spotify.com/dashboard. Owner of the app must have Premium Spotify. |
| `SPOTIPY_CLIENT_SECRET` | No* | spotipy | Same account as Client ID. |
| `SLSK_USERNAME` | Yes | Soulseek (blocked) | Soulseek account username. |
| `SLSK_PASSWORD` | Yes | Soulseek (blocked) | Soulseek account password. |

*Required for spotipy. Not needed for spotipyfree (no-auth, public playlists only).

---

## Installed Tool Versions

```bash
python3 -c "from SpotipyFree import Spotify; print('spotipyfree OK')"
python3 -c "import spotipy; print('spotipy', spotipy.__version__)"
ffmpeg -version 2>&1 | head -1
```

Expected output:
- `spotipyfree OK`
- `spotipy 2.26.0` (if spotipy is installed)
- `ffmpeg version 7.x`
