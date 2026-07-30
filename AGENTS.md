# Soulseek FLAC Pipeline — Agent Skills

This project uses gstack-style skills to run a Spotify → Soulseek → MP3 pipeline.
Skills live in `.agents/skills/` and are invoked by name (e.g., `/spotify-playlist-export`).

## Pipeline

Each skill runs step-by-step with a **manual confirmation gate** between steps. No step runs automatically after another.

```
/spotify-playlist-export     [DONE]
        ↓  (manual gate)
/soulseek-flac-search        [BLOCKED — Soulseek connectivity unavailable]
        ↓  (manual gate)
/soulseek-flac-download      [BLOCKED — depends on search]
        ↓  (manual gate)
/flac-to-mp3                [BLOCKED — depends on download]
```

## Available Skills

| Skill | Status | Output |
|-------|--------|--------|
| `/spotify-playlist-export` | **Active** | `~/Desktop/soulseek QT/playlist_{NAME}_{TIMESTAMP}.md` |
| `/soulseek-flac-search` | Blocked | `search_manifest.json` |
| `/soulseek-flac-download` | Blocked | `~/Downloads/soulseek-flac/*.flac` |
| `/flac-to-mp3` | Blocked | `~/Downloads/soulseek-flac/*.mp3` |

## Skills Directory

```
.agents/skills/
  gstack-spotify-playlist-export/
    agents/openai.yaml
    SKILL.md
  gstack-soulseek-flac-search/
    agents/openai.yaml
    SKILL.md               ← blocked
  gstack-soulseek-flac-download/
    agents/openai.yaml
    SKILL.md               ← blocked
  gstack-flac-to-mp3/
    agents/openai.yaml
    SKILL.md
```

## Tools

Tool definitions are in `.agents/tools/tools.md`.

## Prerequisites

Installed tools:
```bash
# Spotify (no credentials needed)
python3 -m pip install --break-system-packages spotipyfree

# FLAC encoding
brew install ffmpeg

# Soulseek (blocked)
python3 -m pip install --break-system-packages aioslsk
```

Verify:
```bash
python3 -c "from SpotipyFree import Spotify; print('spotipyfree OK')"
ffmpeg -version 2>&1 | head -1
```

## Skill Conventions

- **Source of truth:** `~/.claude/skills/gstack/<skill>/SKILL.md.tmpl`
- **Agent SKILL.md:** `.agents/skills/gstack-<skill>/SKILL.md`
- **Manual gates:** Every skill stops and asks for confirmation before handing off
- **Telemetry:** Skills log to `~/.gstack/analytics/skill-usage.jsonl`
- **Learnings:** Durable discoveries are logged via `$GSTACK_BIN/gstack-learnings-log`
