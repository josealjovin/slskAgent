---
name: soulseek-flac-search
description: |
  Read a playlist_*.md file and search Soulseek (via aioslsk) for each track's FLAC
  version. Produces a search_manifest.json mapping tracks to found files. Use when
  asked to "search soulseek for flac", "find flac files", or "search playlist on soulseek".
  Requires aioslsk and Soulseek account credentials. (gstack)
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
<!-- This file is the agent-side SKILL.md for .agents/skills/gstack-soulseek-flac-search -->

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
  echo '{"skill":"soulseek-flac-search","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"soulseek-pipeline"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
$GSTACK_BIN/gstack-timeline-log '{"skill":"soulseek-flac-search","event":"started","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
```

# Soulseek FLAC Search

## Prerequisites Check

Verify the required Python packages are installed:

```bash
pip show aioslsk >/dev/null 2>&1 || echo "MISSING: aioslsk"
```

If `aioslsk` is missing, abort and tell the user to install it:
```
pip install aioslsk
```

---

## Step 1: Locate the Playlist Markdown File

Use AskUserQuestion to confirm readiness before searching:
> "Ready to search Soulseek for FLAC files matching the tracks in `playlist_{NAME}.md`? (y/n)"

Do not proceed until confirmed.

If no path is provided, glob for it:
```bash
ls ~/Desktop/soulseek\ QT/playlist_*.md 2>/dev/null || echo "NO_PLAYLIST_MD"
```

Read the playlist `.md` file. Extract track titles and artists. Skip header lines (anything before `## Tracks` or lines matching `^\d+\.`).

---

## Step 2: Collect Soulseek Credentials

Check for environment variables:
```bash
echo "SLSK_USERNAME=${SLSK_USERNAME:-not set}"
echo "SLSK_PASSWORD=${SLSK_PASSWORD:+set}"
```

If credentials are missing, use AskUserQuestion to ask for:
1. Soulseek username
2. Soulseek password

---

## Step 3: Search Soulseek for Each Track

Write and run a Python search script using aioslsk:

```bash
cat << 'PYEOF' > /tmp/soulseek_search.py
import asyncio
import json
import os
import sys
import re
from datetime import datetime

try:
    from aioslsk.client import SoulSeekClient
    from aioslsk.settings import Settings, CredentialsSettings
    from aioslsk.events import SearchResultEvent
except ImportError:
    print("ERROR: aioslsk not installed. Run: pip install aioslsk")
    sys.exit(1)

USERNAME = os.environ.get("SLSK_USERNAME")
PASSWORD = os.environ.get("SLSK_PASSWORD")
PLAYLIST_MD = sys.argv[1] if len(sys.argv) > 1 else None
TIMEOUT = int(os.environ.get("SEARCH_TIMEOUT", "30"))

if not USERNAME or not PASSWORD:
    print("ERROR: SLSK_USERNAME and SLSK_PASSWORD must be set")
    sys.exit(1)

if not PLAYLIST_MD:
    import glob
    files = glob.glob(os.path.expanduser("~/Desktop/soulseek QT/playlist_*.md"))
    if files:
        PLAYLIST_MD = sorted(files)[-1]
    else:
        print("ERROR: No playlist_*.md file found")
        sys.exit(1)

with open(PLAYLIST_MD) as f:
    content = f.read()

tracks = []
for line in content.splitlines():
    m = re.match(r'^\s*\d+\.\s*\*\*(.+?)\*\*\s*[-–]\s*(.+?)\s*\(', line)
    if m:
        title = m.group(1).strip()
        artist = m.group(2).strip()
        query = f"{artist} {title} flac"
        tracks.append({"title": title, "artist": artist, "query": query})

print(f"SEARCHING:{len(tracks)} tracks")

async def search_track(client, query, track):
    try:
        request = await asyncio.wait_for(
            client.searches.search(query),
            timeout=TIMEOUT
        )
        await asyncio.sleep(2)

        for item in request.files:
            fn = item.filename.lower()
            if '.flac' in fn or '.flac' in str(item.bitrate).lower():
                size_mb = (item.size or 0) / (1024 * 1024)
                return {
                    "track_title": track["title"],
                    "track_artist": track["artist"],
                    "filename": item.filename,
                    "username": request.username,
                    "size_mb": round(size_mb, 1),
                    "bitrate": item.bitrate,
                }
    except asyncio.TimeoutError:
        pass
    except Exception as e:
        print(f"  [WARN] Search error for '{query}': {e}", file=sys.stderr)
    return None

async def main():
    results = []
    settings = Settings(
        credentials=CredentialsSettings(username=USERNAME, password=PASSWORD)
    )

    async with SoulSeekClient(settings) as client:
        await client.login()
        print(f"LOGGED_IN:{client}")

        for i, track in enumerate(tracks, 1):
            print(f"  [{i}/{len(tracks)}] Searching: {track['query']}")
            result = await search_track(client, track["query"], track)
            if result:
                results.append(result)
                print(f"  FOUND: {result['filename']} from {result['username']} ({result['size_mb']}MB)")
            else:
                print(f"  NOT FOUND: {track['title']}")
            await asyncio.sleep(1)

    manifest = {
        "playlist_md": PLAYLIST_MD,
        "searched_at": datetime.now().isoformat(),
        "total_tracks": len(tracks),
        "found_count": len(results),
        "results": results,
    }

    out_path = os.path.join(
        os.path.dirname(os.path.expanduser(PLAYLIST_MD)),
        "search_manifest.json"
    )
    with open(out_path, "w") as f:
        json.dump(manifest, f, indent=2)

    print(f"MANIFEST:{out_path}")
    print(f"FOUND:{len(results)}/{len(tracks)}")

asyncio.run(main())
PYEOF
python3 /tmp/soulseek_search.py "$PLAYLIST_MD_PATH"
```

Store the manifest path as `$MANIFEST_JSON`.

---

## Step 4: Report Results

Read the manifest and report:
- How many tracks were found vs searched
- A table of found files (filename, user, size)
- Which tracks were not found
- Path to `search_manifest.json`

---

## Manual Handoff to Next Skill

**STOP here. Do not auto-invoke the next skill.** Use AskUserQuestion to confirm:

> "Found {N} FLAC files. Ready to invoke `/soulseek-flac-download` to download them to `~/Downloads/soulseek-flac/`?"

---

## Operational Self-Improvement

Before completing, if you discovered a durable project quirk or command fix that would save 5+ minutes next time, log it:

```bash
$GSTACK_BIN/gstack-learnings-log '{"skill":"soulseek-flac-search","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

Do not log obvious facts or one-time transient errors.

## Telemetry (run last)

After workflow completion, log telemetry. OUTCOME is success/error/abort/unknown.

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
$GSTACK_ROOT/bin/gstack-timeline-log '{"skill":"soulseek-flac-search","event":"completed","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
if [ "$_TEL" != "off" ]; then
  echo '{"skill":"soulseek-flac-search","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
```

---

## Completion Status

- DONE — manifest created with at least one found file
- PARTIAL — manifest created, some tracks not found
- BLOCKED — aioslsk not installed or credentials missing

## Important Rules

- **Anti-DOS:** Do not connect/disconnect rapidly. One connection per session.
- **Rate limit:** Add 1-2s sleep between searches to avoid server bans.
- **Search query:** Append `flac` to queries to filter results. Format: `ARTIST TRACK flac`.
- **FLAC preference:** Always prefer `.flac` files over `.wav` or `.alac`.
- **Only save the manifest** — no downloads happen in this step.
