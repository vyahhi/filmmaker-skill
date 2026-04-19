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

## Step 3: Generate Character Portraits

For each character, generate a reference portrait (text→image). This portrait will anchor visual consistency in every scene featuring that character.

Build the prompt as: `<style>. <character_description>. Portrait, facing camera, high detail.`

```bash
curl -s https://api.nunchaku.dev/v1/images/generations \
  -H "Authorization: Bearer $NUNCHAKU_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"nunchaku-qwen-image\",
    \"prompt\": \"<style>. <character_description>. Portrait, facing camera, high detail.\",
    \"n\": 1,
    \"size\": \"1280x720\",
    \"tier\": \"fast\",
    \"response_format\": \"b64_json\"
  }" \
  | jq -r '.data[0].b64_json' \
  | base64 $BASE64_FLAG - > "$OUTPUT_DIR/characters/<name>.jpg"
```

Wait for each call to complete before starting the next. If a call returns a non-200 status or jq returns null, print the error and stop.

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

curl -s https://api.nunchaku.dev/v1/images/edits \
  -H "Authorization: Bearer $NUNCHAKU_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/scene_payload.json \
  | jq -r '.data[0].b64_json' \
  | base64 $BASE64_FLAG - > "$OUTPUT_DIR/scenes/scene_0<N>.jpg"
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

curl -s --max-time 120 https://api.nunchaku.dev/v1/video/animations \
  -H "Authorization: Bearer $NUNCHAKU_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/video_payload.json \
  | jq -r '.data[0].b64_json' \
  | base64 $BASE64_FLAG - > "$OUTPUT_DIR/scenes/scene_0<N>.mp4"
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

- API returns non-2xx → print status + first 500 chars of response body, then stop
- `jq` returns `null` → the response was malformed; print raw response and stop
- API 429 → wait 10 seconds, retry up to 3 times
- API 402 → print "Out of credits — check your dashboard at sundai.nunchaku.dev" and stop
- Any clip file is 0 bytes after decoding → report which scene failed and stop
