# Soulseek FLAC Pipeline

CLI routine: fetch a Spotify playlist → search Soulseek for FLAC files → download them → encode to MP3 320kbps.

**Status: Step 1 complete.** Steps 2–4 are paused (Soulseek connectivity unavailable).

---

## Current State

| Step | Skill | Status |
|------|-------|--------|
| 1. Export playlist | `/spotify-playlist-export` | **DONE** — `playlist_Afternoon_Acoustic_20260730_152751.md` (25 tracks) |
| 2. Search Soulseek | `/soulseek-flac-search` | **BLOCKED** — Soulseek connection timed out |
| 3. Download FLAC | `/soulseek-flac-download` | **BLOCKED** — depends on step 2 |
| 4. Encode to MP3 | `/flac-to-mp3` | **BLOCKED** — depends on step 3 |

---

## Installed Dependencies

| Tool | Status | Version |
|------|--------|---------|
| spotipyfree | **Installed** | 1.9.13 |
| spotipy | Installed | 2.26.0 |
| ffmpeg | **Installed** | 7.1 |
| aioslsk | Not installed | — |

Verify:
```bash
python3 -c "from SpotipyFree import Spotify; print('spotipyfree OK')"
ffmpeg -version 2>&1 | head -1
```

---

## Install Dependencies

```bash
# Spotify export (no credentials needed)
python3 -m pip install --break-system-packages spotipyfree

# FLAC encoding (ffmpeg already installed)
brew install ffmpeg

# Soulseek (blocked — connectivity issue)
python3 -m pip install --break-system-packages aioslsk
```

---

## Running the Pipeline

Each step requires manual confirmation before the next skill is invoked.

### Step 1 — Export Spotify Playlist

```
/spotify-playlist-export
```

- Asks for the Spotify playlist URL
- Fetches all tracks, writes `~/Desktop/soulseek QT/playlist_{NAME}_{TIMESTAMP}.md`
- Confirms before continuing

### Step 2 — Search Soulseek for FLAC

```
/soulseek-flac-search
```

- Reads the `.md` file from step 1
- Searches Soulseek for each track's FLAC version
- Confirms before downloading

### Step 3 — Download FLAC Files

```
/soulseek-flac-download
```

- Reads `search_manifest.json`
- Downloads all found FLACs to `~/Downloads/soulseek-flac/`
- Confirms before encoding

### Step 4 — Encode to MP3 320kbps

```
/flac-to-mp3
```

- Reads `~/Downloads/soulseek-flac/*.flac`
- Batch-encodes to MP3 320kbps via ffmpeg
- Asks before deleting original FLAC files

---

## Architecture

```
/spotify-playlist-export     →  ~/Desktop/soulseek QT/playlist_{NAME}_{TIMESTAMP}.md
        ↓ (manual gate)
/soulseek-flac-search      →  ~/Desktop/soulseek QT/search_manifest.json  [BLOCKED]
        ↓ (manual gate)
/soulseek-flac-download     →  ~/Downloads/soulseek-flac/*.flac
        ↓ (manual gate)
/flac-to-mp3                →  ~/Downloads/soulseek-flac/*.mp3
                                (original .flac deleted after confirmation)
```

---

## Skills Location

```
~/.claude/skills/gstack/
  spotify-playlist-export/SKILL.md.tmpl   ← edit here to update
  soulseek-flac-search/SKILL.md.tmpl      ← blocked
  soulseek-flac-download/SKILL.md.tmpl   ← blocked
  flac-to-mp3/SKILL.md.tmpl
```

Agent-side copies:
```
.agents/skills/gstack-spotify-playlist-export/SKILL.md
.agents/skills/gstack-soulseek-flac-search/SKILL.md
.agents/skills/gstack-soulseek-flac-download/SKILL.md
.agents/skills/gstack-flac-to-mp3/SKILL.md
```

---

## Design Decisions

- **spotipyfree over spotipy:** No credentials needed for public playlists.
- **FFmpeg LAME:** Single tool, native metadata handling with `-map_metadata 0`.
- **320kbps CBR:** User requirement.
- **Delete FLAC after encode:** Saves disk space. MP3 is the deliverable.
- **Manual gates between steps:** Each skill waits for confirmation before handing off.
- **aioslsk over slskd:** No daemon. Scripts connect directly to Soulseek.
