---
name: filmmaker
description: Turn a one-line concept into a short film using the Nunchaku API. Generates character portraits, scene images, and animated video clips, then stitches them into a final MP4.
---

# Filmmaker Skill

Turn a concept into a ~17s short film using the Nunchaku image/video generation API.

## Trigger

Invoked via `/filmmaker "<concept>"` or `/filmmaker` (no args).

## Requirements Check

Before doing anything else:

1. Check `NUNCHAKU_API_KEY` is set: `echo $NUNCHAKU_API_KEY`. If empty, print:
   > "Set your API key first: export NUNCHAKU_API_KEY=sk-nunchaku-..."
   Then stop.

2. Check `jq` is installed: `which jq`. If missing, print:
   > "Install jq first: brew install jq"
   Then stop.

3. Check `ffmpeg` is installed: `which ffmpeg`. Note if missing — you'll skip stitching but continue.

4. Detect OS for base64 flag:
   - macOS → `BASE64_FLAG="-i"`
   - Linux → `BASE64_FLAG="-w0"`

## Step 1: Invent or Parse Concept

If invoked with no arguments, invent a surreal, unexpected movie concept yourself. Be bold — a mundane concept is a failure. Example style: "a sentient traffic cone falls in love with a GPS satellite" or "an elderly librarian discovers she can enter books and starts changing their endings."

## Step 2: Write plan.json

Think carefully, then write a `plan.json` to the output folder. Rules:

- Max 2 unique characters
- Exactly 1 character per scene, 5 scenes total
- At least 1 character appears in multiple scenes
- `style`: a global visual style string prepended to every image prompt (e.g. "cinematic 35mm film, warm golden tones, shallow depth of field")
- `image_prompt`: scene composition only — do NOT repeat the character description, you will prepend it at call time
- `video_prompt`: describe motion/action for the 3.4s clip
- `narration`: subtitle text that fits spoken in ~5 seconds

```json
{
  "title": "...",
  "style": "...",
  "characters": [
    { "name": "...", "description": "detailed physical description: face, hair, clothing, expression, art style" }
  ],
  "scenes": [
    {
      "index": 1,
      "character": "character name",
      "image_prompt": "scene composition, setting, lighting, mood",
      "video_prompt": "motion description",
      "narration": "subtitle line"
    }
  ]
}
```

Create the output folder and save the plan:

```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SAFE_TITLE=$(echo "<title>" | tr ' ' '_' | tr -cd '[:alnum:]_-')
OUTPUT_DIR="output/${SAFE_TITLE}_${TIMESTAMP}"
mkdir -p "$OUTPUT_DIR/characters" "$OUTPUT_DIR/scenes"
# write plan.json to $OUTPUT_DIR/plan.json
```

## API Call Helper

Use this function for ALL API calls to handle retries automatically. Define it once at the start:

```bash
nunchaku_call() {
  local ENDPOINT="$1"   # e.g. /v1/images/generations
  local PAYLOAD="$2"    # path to JSON payload file
  local OUT="$3"        # output file path
  local RETRIES=10
  for i in $(seq 1 $RETRIES); do
    RESP=$(curl -s --max-time 180 "https://api.nunchaku.dev${ENDPOINT}" \
      -H "Authorization: Bearer $NUNCHAKU_API_KEY" \
      -H "Content-Type: application/json" \
      -d @"$PAYLOAD")
    CODE=$(echo "$RESP" | jq -r '.error.code // empty' 2>/dev/null)
    if [ "$CODE" = "concurrent_limit_exceeded" ]; then
      echo "Concurrent limit — waiting 30s (attempt $i/$RETRIES)..."
      sleep 30; continue
    fi
    if [ "$CODE" = "rate_limit_exceeded" ]; then
      echo "Rate limited — waiting 15s (attempt $i/$RETRIES)..."
      sleep 15; continue
    fi
    # Handle Cloudflare/server timeout (HTTP 524, 502, 503) — non-JSON response
    if echo "$RESP" | grep -q "error code:"; then
      echo "Server error ($(echo $RESP | head -c 50)) — waiting 20s (attempt $i/$RETRIES)..."
      sleep 20; continue
    fi
    B64=$(echo "$RESP" | jq -r '.data[0].b64_json // empty' 2>/dev/null)
    if [ -z "$B64" ] || [ "$B64" = "null" ]; then
      echo "ERROR: unexpected response: $(echo $RESP | head -c 500)"; exit 1
    fi
    echo "$B64" | base64 $BASE64_FLAG - > "$OUT"
    SIZE=$(wc -c < "$OUT")
    if [ "$SIZE" -lt 1000 ]; then
      echo "ERROR: output file too small ($SIZE bytes) — decode likely failed"; exit 1
    fi
    return 0
  done
  echo "ERROR: all $RETRIES attempts failed"; exit 1
}
```

**IMPORTANT:** Never run API calls in the background. Always run them sequentially — the free plan allows only 1 simultaneous task.

## Step 3: Generate Character Portraits

For each character, generate a reference portrait (text→image). Write prompt to a temp payload file to avoid shell quoting issues:

```bash
cat > /tmp/portrait_<name>.json <<EOF
{
  "model": "nunchaku-qwen-image",
  "prompt": "<style>. <character_description>. Portrait, facing camera, high detail.",
  "n": 1,
  "size": "1280x720",
  "tier": "fast",
  "response_format": "b64_json"
}
EOF

nunchaku_call /v1/images/generations /tmp/portrait_<name>.json "$OUTPUT_DIR/characters/<name>.jpg"
echo "<name> portrait done."
```

Wait for each call to complete before starting the next.

## Step 4: Generate Scene Images (image→image edit)

For each scene (index 1–5), generate the scene image using the matching character's portrait as input.

Build the full prompt as: `<style>. <character_description>. <image_prompt>.`

Write a temp JSON payload file to avoid shell quoting issues with large base64 strings:

```bash
IMG_B64=$(base64 $BASE64_FLAG "$OUTPUT_DIR/characters/<character_name>.jpg")
cat > /tmp/scene_payload.json <<EOF
{
  "model": "nunchaku-qwen-image-edit",
  "prompt": "<style>. <character_description>. <image_prompt>.",
  "url": "data:image/jpeg;base64,$IMG_B64",
  "n": 1,
  "size": "1280x720",
  "tier": "fast",
  "response_format": "b64_json"
}
EOF

nunchaku_call /v1/images/edits /tmp/scene_payload.json "$OUTPUT_DIR/scenes/scene_0<N>.jpg"
```

Print progress: `Scene N/5 image done.`

## Step 5: Animate Each Scene (image→video)

For each scene image, generate a 3.4s video clip.

```bash
IMG_B64=$(base64 $BASE64_FLAG "$OUTPUT_DIR/scenes/scene_0<N>.jpg")
cat > /tmp/video_payload.json <<EOF
{
  "model": "nunchaku-wan2.2-lightning-i2v",
  "prompt": "<video_prompt>",
  "n": 1,
  "size": "1280x720",
  "num_frames": 81,
  "num_inference_steps": 4,
  "guidance_scale": 1.0,
  "response_format": "b64_json",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,$IMG_B64"}},
        {"type": "text", "text": "<video_prompt>"}
      ]
    }
  ]
}
EOF

nunchaku_call /v1/video/animations /tmp/video_payload.json "$OUTPUT_DIR/scenes/scene_0<N>.mp4"
```

Print progress: `Scene N/5 video done (~30s per clip, please wait).`

## Step 6: Write Subtitles

Write `subtitles.srt` from the plan narration fields. Scene N starts at `(N-1)*5` seconds.

```
1
00:00:00,000 --> 00:00:05,000
<scene 1 narration>

2
00:00:05,000 --> 00:00:10,000
<scene 2 narration>

...
```

Save to `$OUTPUT_DIR/subtitles.srt`.

## Step 7: Stitch Clips

If `ffmpeg` is available:

```bash
for i in 01 02 03 04 05; do
  echo "file 'scenes/scene_$i.mp4'"
done > "$OUTPUT_DIR/concat.txt"

ffmpeg -f concat -safe 0 -i "$OUTPUT_DIR/concat.txt" -c copy "$OUTPUT_DIR/final.mp4" -y
```

If `ffmpeg` is not available, skip this step and print:
> "Clips saved to $OUTPUT_DIR/scenes/. Install ffmpeg to stitch: brew install ffmpeg"

## Step 8: Report

Print a summary:

```
Film complete!
Title: <title>
Output: <OUTPUT_DIR>/
  characters/  — <N> reference portraits
  scenes/      — 5 images + 5 video clips
  subtitles.srt
  final.mp4    ✓  (or "ffmpeg not found — clips saved individually")

Concept: <concept>
```

## Error Handling

- `concurrent_limit_exceeded` → wait 15s, retry up to 5 times (handled by `nunchaku_call`)
- API 429 / `rate_limit_exceeded` → wait 10s, retry up to 5 times (handled by `nunchaku_call`)
- API 402 → print "Out of credits — check your dashboard at sundai.nunchaku.dev" and stop
- `jq` returns `null` or empty → the response was malformed; print raw response and stop
- Output file < 1000 bytes after decoding → base64 decode failed; report and stop
- **Never run API calls in background** — the free plan allows only 1 simultaneous task
