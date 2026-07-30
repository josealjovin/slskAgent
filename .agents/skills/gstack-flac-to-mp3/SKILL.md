---
name: flac-to-mp3
description: |
  Batch-encode all .flac files in ~/Downloads/soulseek-flac/ to .mp3 at 320kbps
  using ffmpeg, preserving metadata. Deletes the original .flac files afterward.
  Use when asked to "encode to mp3", "convert flac to mp3", "encode the downloads",
  or "make mp3s". (gstack)
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
<!-- This file is the agent-side SKILL.md for .agents/skills/gstack-flac-to-mp3 -->

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
  echo '{"skill":"flac-to-mp3","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"soulseek-pipeline"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
$GSTACK_BIN/gstack-timeline-log '{"skill":"flac-to-mp3","event":"started","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
```

# FLAC to MP3 320kbps Batch Encoder

## Prerequisites Check

Verify `ffmpeg` is installed:

```bash
which ffmpeg >/dev/null 2>&1 && ffmpeg -version 2>&1 | head -1 || echo "MISSING: ffmpeg"
```

If `ffmpeg` is missing, abort and tell the user to install it:
```
brew install ffmpeg
```

---

## Step 1: Locate FLAC Files

Use AskUserQuestion to confirm readiness before encoding:
> "Ready to encode all FLAC files in `~/Downloads/soulseek-flac/` to MP3 320kbps? Original .flac files will be deleted after encoding. (y/n)"

Do not proceed until confirmed.

```bash
FLAC_DIR=~/Downloads/soulseek-flac
mkdir -p "$FLAC_DIR"
echo "FLAC_FILES:$(ls "$FLAC_DIR"/*.flac 2>/dev/null | wc -l | tr -d ' ')"
ls -lh "$FLAC_DIR"/*.flac 2>/dev/null || echo "NO_FLAC_FILES"
```

If no `.flac` files are found and no directory is specified, use AskUserQuestion to ask for the source directory.

---

## Step 2: Verify FFmpeg LAME Support

```bash
ffmpeg -encoders 2>/dev/null | grep -i "libmp3lame" || echo "LAME_MISSING"
```

If LAME/libmp3lame is missing, tell the user to reinstall ffmpeg with libmp3lame:
```
brew reinstall ffmpeg
```

---

## Step 3: Batch Encode FLAC → MP3 (320kbps)

Run the batch conversion:

```bash
FLAC_DIR=~/Downloads/soulseek-flac
cd "$FLAC_DIR"

ENCODED=0
SKIPPED=0
FAILED=0

for flac in *.flac; do
    [ -e "$flac" ] || continue
    base="${flac%.flac}"
    mp3="${base}.mp3"

    if [ -f "$mp3" ]; then
        echo "SKIP (mp3 exists): $mp3"
        SKIPPED=$((SKIPPED + 1))
        continue
    fi

    echo "ENCODING: $flac → $mp3"
    ffmpeg -y -i "$flac" \
        -acodec libmp3lame \
        -q:a 0 \
        -b:a 320k \
        -map_metadata 0 \
        "$mp3" 2>/dev/null

    if [ $? -eq 0 ] && [ -f "$mp3" ]; then
        orig_size=$(stat -f%z "$flac" 2>/dev/null || stat -c%s "$flac" 2>/dev/null)
        new_size=$(stat -f%z "$mp3" 2>/dev/null || stat -c%s "$mp3" 2>/dev/null)
        ratio=$(echo "scale=1; $new_size * 100 / $orig_size" | bc 2>/dev/null || echo "?")
        echo "  DONE: ${mp3} (${ratio}% of original)"
        ENCODED=$((ENCODED + 1))
    else
        echo "  FAILED: $flac"
        FAILED=$((FAILED + 1))
    fi
done

echo "ENCODED:$ENCODED"
echo "SKIPPED:$SKIPPED"
echo "FAILED:$FAILED"
```

---

## Step 4: Report Encoding Results

```bash
echo "=== MP3 OUTPUT ==="
ls -lh ~/Downloads/soulseek-flac/*.mp3 | awk '{print $5, $9}'
echo "TOTAL_MP3S: $(ls ~/Downloads/soulseek-flac/*.mp3 2>/dev/null | wc -l | tr -d ' ')"
echo "TOTAL_FLACS_REMAINING: $(ls ~/Downloads/soulseek-flac/*.flac 2>/dev/null | wc -l | tr -d ' ')"
```

---

## Step 5: Delete Original FLAC Files (User Confirmed Intent)

Confirm with AskUserQuestion before deleting:
> "All FLAC files have been encoded to MP3 at 320kbps. Delete the original `.flac` files to free up disk space? (y/n)"

If confirmed:

```bash
cd ~/Downloads/soulseek-flac
for flac in *.flac; do
    [ -e "$flac" ] || continue
    echo "DELETING: $flac"
    rm "$flac"
done
echo "FLAC_DELETED: $(ls ~/Downloads/soulseek-flac/*.flac 2>/dev/null | wc -l | tr -d ' ') remaining"
```

---

## Operational Self-Improvement

Before completing, if you discovered a durable project quirk or command fix that would save 5+ minutes next time, log it:

```bash
$GSTACK_BIN/gstack-learnings-log '{"skill":"flac-to-mp3","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

Do not log obvious facts or one-time transient errors.

## Telemetry (run last)

After workflow completion, log telemetry. OUTCOME is success/error/abort/unknown.

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
$GSTACK_ROOT/bin/gstack-timeline-log '{"skill":"flac-to-mp3","event":"completed","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
if [ "$_TEL" != "off" ]; then
  echo '{"skill":"flac-to-mp3","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
```

---

## Completion Status

- DONE — all FLAC files encoded to MP3 and originals deleted
- PARTIAL — encoded successfully, originals kept
- BLOCKED — ffmpeg missing or no FLAC files found

## Important Rules

- **Bitrate:** Always `-b:a 320k` (CBR 320kbps). Do not use lower bitrates.
- **Metadata:** Always use `-map_metadata 0` to preserve ID3 tags from FLAC.
- **Skip existing:** If the MP3 already exists, skip (don't re-encode).
- **Delete only after confirmation** — never delete files without asking first.
- **Output directory:** Always `~/Downloads/soulseek-flac/`
- **FFmpeg quality:** `-q:a 0` maps to LAME V0 (~245kbps VBR). Combined with `-b:a 320k` gives maximum quality CBR.
