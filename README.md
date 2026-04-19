# filmmaker-skill

A Claude Code skill that turns a one-line concept into a short film (~17s MP4) using the [Nunchaku](https://sundai.nunchaku.dev/) image and video generation API.

**Pipeline:** concept → script → character portraits → scene images → video clips → stitched film

No Python required. Uses `curl`, `jq`, and `ffmpeg`.

## What it does

1. Writes a 5-scene script with up to 2 characters
2. Generates a reference portrait per character (for visual consistency)
3. Generates each scene image via image-to-image edit anchored to the character portrait
4. Animates each scene into a 3.4s video clip
5. Stitches all clips into a final `final.mp4`

**Cost:** ~$0.15 per film &nbsp;|&nbsp; **Time:** ~3 minutes

## Requirements

- [Claude Code](https://claude.ai/code)
- `jq` — `brew install jq`
- `ffmpeg` — `brew install ffmpeg` (optional — skipped if missing, clips saved individually)
- A Nunchaku API key from [sundai.nunchaku.dev](https://sundai.nunchaku.dev/)

## Install

```bash
# Clone into your Claude Code plugins directory
git clone https://github.com/vyahhi/filmmaker-skill ~/.claude/plugins/filmmaker-skill
```

Then in `~/.claude/settings.json`, add the plugin path (or Claude Code will pick it up automatically from `~/.claude/plugins/`).

## Usage

```bash
# Set your API key
export NUNCHAKU_API_KEY="sk-nunchaku-..."

# Generate a film from a concept
/filmmaker "a lonely astronaut discovers an alien cat on the moon"

# No concept? Claude invents something wild
/filmmaker
```

Output is saved to `output/<title>_<timestamp>/`:

```
output/lonely_astronaut_20260419_120000/
  plan.json          # full script and prompts
  characters/        # reference portraits
  scenes/            # scene images + video clips
  subtitles.srt
  final.mp4
```

## Example

```
/filmmaker "an elderly librarian discovers she can enter books and starts rewriting endings"
```

Film complete in ~3 minutes. Assets in `output/elderly_librarian_*/`.
