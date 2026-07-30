---
name: soulseek-flac-download
description: |
  Read a search_manifest.json and download all FLAC files from Soulseek via aioslsk
  into ~/Downloads/soulseek-flac/. Use when asked to "download flac files",
  "download from soulseek", or "fetch the flac files" after a search. (gstack)
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
<!-- This file is the agent-side SKILL.md for .agents/skills/gstack-soulseek-flac-download -->

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
  echo '{"skill":"soulseek-flac-download","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"soulseek-pipeline"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
$GSTACK_BIN/gstack-timeline-log '{"skill":"soulseek-flac-download","event":"started","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
```

# Soulseek FLAC Download

## Prerequisites Check

Verify the required Python packages are installed:

```bash
pip show aioslsk >/dev/null 2>&1 || echo "MISSING: aioslsk"
```

If `aioslsk` is missing, abort and tell the user to install it:
```
pip install aioslsk
```

Verify ffmpeg is available (needed for future encoding):
```bash
which ffmpeg >/dev/null 2>&1 || echo "MISSING: ffmpeg (needed for encoding later)"
```

---

## Step 1: Locate the Search Manifest

Use AskUserQuestion to confirm readiness before downloading:
> "Ready to download {N} FLAC files to `~/Downloads/soulseek-flac/`? (y/n)"

Do not proceed until confirmed.

If no path is provided:
```bash
ls ~/Desktop/soulseek\ QT/search_manifest.json 2>/dev/null || echo "NO_MANIFEST"
```

Read the manifest to confirm results exist.

---

## Step 2: Prepare Download Directory

```bash
DOWNLOAD_DIR=~/Downloads/soulseek-flac
mkdir -p "$DOWNLOAD_DIR"
echo "DOWNLOAD_DIR:$DOWNLOAD_DIR"
```

---

## Step 3: Download FLAC Files via aioslsk

Write and run the download script:

```bash
cat << 'PYEOF' > /tmp/soulseek_download.py
import asyncio
import os
import json
import sys
import aiofiles
from datetime import datetime

try:
    from aioslsk.client import SoulSeekClient
    from aioslsk.settings import Settings, CredentialsSettings
except ImportError:
    print("ERROR: aioslsk not installed")
    sys.exit(1)

USERNAME = os.environ.get("SLSK_USERNAME")
PASSWORD = os.environ.get("SLSK_PASSWORD")
MANIFEST_PATH = sys.argv[1] if len(sys.argv) > 1 else None
DOWNLOAD_DIR = os.environ.get("DOWNLOAD_DIR", os.path.expanduser("~/Downloads/soulseek-flac"))

if not MANIFEST_PATH:
    md_dir = os.path.expanduser("~/Desktop/soulseek QT")
    manifest_path = os.path.join(md_dir, "search_manifest.json")
    if os.path.exists(manifest_path):
        MANIFEST_PATH = manifest_path

if not MANIFEST_PATH or not os.path.exists(MANIFEST_PATH):
    print("ERROR: search_manifest.json not found")
    sys.exit(1)

with open(MANIFEST_PATH) as f:
    manifest = json.load(f)

results = manifest.get("results", [])
print(f"DOWNLOADING:{len(results)} files")

os.makedirs(DOWNLOAD_DIR, exist_ok=True)

downloaded = []
failed = []

async def download_file(client, result):
    username = result["username"]
    filename = result["filename"]
    local_path = os.path.join(DOWNLOAD_DIR, os.path.basename(filename))

    if os.path.exists(local_path):
        print(f"  SKIP (exists): {os.path.basename(filename)}")
        downloaded.append(local_path)
        return

    try:
        print(f"  DOWNLOADING: {filename} from {username}")
        await client.transfers.download(username, filename, local_path)
        downloaded.append(local_path)
        print(f"  DONE: {os.path.basename(filename)} ({os.path.getsize(local_path) / (1024*1024):.1f}MB)")
    except Exception as e:
        failed.append({"filename": filename, "error": str(e)})
        print(f"  FAILED: {filename} — {e}", file=sys.stderr)

async def main():
    settings = Settings(
        credentials=CredentialsSettings(username=USERNAME, password=PASSWORD)
    )

    async with SoulSeekClient(settings) as client:
        await client.login()
        print(f"LOGGED_IN as {USERNAME}")

        for result in results:
            await download_file(client, result)
            await asyncio.sleep(0.5)

    summary = {
        "manifest": MANIFEST_PATH,
        "downloaded_at": datetime.now().isoformat(),
        "downloaded": downloaded,
        "failed": failed,
        "download_dir": DOWNLOAD_DIR,
    }

    summary_path = os.path.join(DOWNLOAD_DIR, "download_summary.json")
    with open(summary_path, "w") as f:
        json.dump(summary, f, indent=2)

    print(f"SUMMARY:{summary_path}")
    print(f"DOWNLOADED:{len(downloaded)}/{len(results)}")
    print(f"FAILED:{len(failed)}")
    if failed:
        print("FAILED_FILES:" + ",".join(f["filename"] for f in failed))

asyncio.run(main())
PYEOF
python3 /tmp/soulseek_download.py "$MANIFEST_PATH"
```

---

## Step 4: Verify Downloads

```bash
ls -lh ~/Downloads/soulseek-flac/*.flac 2>/dev/null | awk '{print $5, $9}'
echo "TOTAL: $(ls ~/Downloads/soulseek-flac/*.flac 2>/dev/null | wc -l | tr -d ' ') files"
```

Report:
- Total files downloaded
- Total size
- Any failed downloads and why
- Path to download summary JSON

---

## Manual Handoff to Next Skill

**STOP here. Do not auto-invoke the next skill.** Use AskUserQuestion to confirm:

> "Downloaded {N} FLAC files to `~/Downloads/soulseek-flac/`. Ready to invoke `/flac-to-mp3` to encode them to MP3 320kbps?"

---

## Operational Self-Improvement

Before completing, if you discovered a durable project quirk or command fix that would save 5+ minutes next time, log it:

```bash
$GSTACK_BIN/gstack-learnings-log '{"skill":"soulseek-flac-download","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

Do not log obvious facts or one-time transient errors.

## Telemetry (run last)

After workflow completion, log telemetry. OUTCOME is success/error/abort/unknown.

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
$GSTACK_ROOT/bin/gstack-timeline-log '{"skill":"soulseek-flac-download","event":"completed","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
if [ "$_TEL" != "off" ]; then
  echo '{"skill":"soulseek-flac-download","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
```

---

## Completion Status

- DONE — all files downloaded successfully
- PARTIAL — some files failed (report which)
- BLOCKED — credentials missing, no manifest found

## Important Rules

- **Download directory:** Always `~/Downloads/soulseek-flac/`
- **Filename collisions:** Skip files that already exist (avoid re-downloading)
- **Soulseek etiquette:** Download from users with available upload slots. Don't max out parallel connections.
- **Manifest is the source of truth** — always read `search_manifest.json` for the download queue.
- **The next step is `/flac-to-mp3`** — pass the download directory to that skill.
