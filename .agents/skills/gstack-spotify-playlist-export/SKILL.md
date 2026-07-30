---
name: spotify-playlist-export
description: |
  Fetch all tracks from a Spotify playlist URL via SpotipyFree and write
  them to a timestamped markdown file. No credentials required for public playlists.
  Use when asked to "export this playlist", "scrape spotify playlist",
  "get playlist tracks", or "spotify to markdown". (gstack)
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->

## Preamble (run first)

```bash
GSTACK_ROOT="$HOME/.claude/skills/gstack"
GSTACK_BIN="$GSTACK_ROOT/bin"
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
find ~/.gstack/sessions -mmin +120 -type f -exec rm {} + 2>/dev/null || true
_TEL=$($GSTACK_BIN/gstack-config get telemetry 2>/dev/null || echo "off")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
mkdir -p ~/.gstack/analytics
if [ "$_TEL" != "off" ]; then
  echo '{"skill":"spotify-playlist-export","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"soulseek-pipeline"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
$GSTACK_BIN/gstack-timeline-log '{"skill":"spotify-playlist-export","event":"started","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
```

# Spotify Playlist Export

## Prerequisites Check

Verify the required Python package is installed:

```bash
python3 -c "from SpotipyFree import Spotify; print('OK')" 2>/dev/null || echo "MISSING: spotipyfree"
```

If `spotipyfree` is missing, tell the user to install it:
```
python3 -m pip install --break-system-packages spotipyfree
```

---

## Step 1: Ask for Playlist URL

Use AskUserQuestion to ask the user for the Spotify playlist URL. Wait for their response before proceeding.

```
AskUserQuestion → question: "Paste the Spotify playlist URL"
```

Do not proceed until the user provides a URL.

---

## Step 2: Fetch Playlist Metadata and Tracks

Write the Python script to a temp file, then run it with the user-provided URL:

```bash
PLAYLIST_URL="<the URL the user provided>"

cat << 'PYEOF' > /tmp/spotify_export.py
import sys
import re
from datetime import datetime

try:
    from SpotipyFree import Spotify
except ImportError:
    print("ERROR: spotipyfree not installed. Run: pip install spotipyfree")
    sys.exit(1)

url = sys.argv[1]

match = re.search(r'playlist/([A-Za-z0-9]+)', url)
if not match:
    print("ERROR: Could not parse playlist ID from URL")
    sys.exit(1)

playlist_id = match.group(1)

s = Spotify()

try:
    pl = s.playlist(playlist_id)
except Exception as e:
    print(f"ERROR: Failed to fetch playlist: {e}")
    sys.exit(1)

name = pl.get("name", "Unknown")
description = pl.get("description", "")
owner = pl.get("owner", {}).get("name", "")
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

tracks = []
items = pl.get("content", {}).get("items", [])
for item in items:
    track_data = item.get("itemV2", {}).get("data", {})
    if track_data.get("__typename") != "Track":
        continue
    title = track_data.get("name", "")
    album = track_data.get("albumOfTrack", {}).get("name", "")
    artists = [a["profile"]["name"] for a in track_data.get("artists", {}).get("items", [])]
    duration_ms = track_data.get("duration", {}).get("totalMilliseconds", 0)
    tracks.append({
        "title": title,
        "artist": ", ".join(artists),
        "album": album,
        "duration_ms": duration_ms,
    })

safe_name = re.sub(r'[^\w\-_. ]+', '', name).strip().replace(' ', '_')
filename = f"playlist_{safe_name}_{timestamp}.md"
filepath = f"/Users/jose.aljovin/Desktop/soulseek QT/{filename}"

lines = [
    f"# {name}",
    "",
    f"- **Description:** {description}",
    f"- **Owner:** {owner}",
    f"- **Tracks:** {len(tracks)}",
    f"- **Exported:** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
    "",
    "## Tracks",
    "",
]
for i, t in enumerate(tracks, 1):
    mins, secs = divmod(t["duration_ms"] // 1000, 60)
    lines.append(f"{i:3}. **{t['title']}** — {t['artist']} ({mins}:{secs:02d})")

with open(filepath, "w") as f:
    f.write("\n".join(lines))

print(f"EXPORTED:{filepath}")
print(f"TRACKS:{len(tracks)}")
PYEOF
python3 /tmp/spotify_export.py "$PLAYLIST_URL"
```

Report the output filepath.

---

## Step 3: Verify Output

Read the generated file and confirm it has the track listing. Report:
- Playlist name and total track count
- Full path to the generated `.md` file
- The file is saved in `~/Desktop/soulseek QT/`

---

## Manual Handoff to Next Skill

**STOP here.** Use AskUserQuestion to confirm:

> "The playlist has been exported to `playlist_{NAME}.md`. Ready to continue with the next step?"

---

## Operational Self-Improvement

Before completing, if you discovered a durable project quirk or command fix that would save 5+ minutes next time, log it:

```bash
$GSTACK_BIN/gstack-learnings-log '{"skill":"spotify-playlist-export","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

## Telemetry (run last)

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
$GSTACK_ROOT/bin/gstack-timeline-log '{"skill":"spotify-playlist-export","event":"completed","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
if [ "$_TEL" != "off" ]; then
  echo '{"skill":"spotify-playlist-export","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
```

---

## Completion Status

- DONE — `.md` file created with all tracks
- BLOCKED — spotipyfree not installed or API error

## Important Rules

- **No credentials required.** spotipyfree uses Spotify's free endpoint — works for public playlists without API keys.
- **Duration may be 0:00** for some tracks — the free endpoint does not always return duration data.
- **Public playlists only.** Private playlists require user OAuth (not covered here).
