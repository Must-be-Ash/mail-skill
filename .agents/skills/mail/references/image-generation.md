# AI Image Generation for Postcards

Generate postcard artwork via DALL-E 3 using the image generation API.

**Model:** DALL-E 3
**Endpoint:** `https://image-generation-api-64k8.onrender.com/v1/image/generate`

## Default aesthetic (when user gives no style direction)

When the user describes their postcard idea without specifying a visual style, bring your own creative direction to the prompt. The target aesthetic is:

**Charming hand-drawn doodle** — clean simple ink lines, light watercolor accents, white background, full character visible, cute and simple composition. Think: a character holding a sign, drawn like a friendly sketch in a notebook. Not heavily rendered, not dark or scratchy, not photo-realistic.

This is a **creative direction**, not a string to paste. Use your judgment to express this aesthetic in the prompt in whatever way best fits the subject. The goal is output that looks like it was hand-sketched with care — clean, warm, and print-friendly.

**Apply this aesthetic when** the user's description focuses on subject or message but gives no style cues:
- "a happy panda holding a sign" → you provide the aesthetic direction
- "flowers and butterflies for grandma" → you provide the aesthetic direction
- "a cat wearing a birthday hat" → you provide the aesthetic direction
- "get well soon with a sloth" → you provide the aesthetic direction

**Do NOT apply this aesthetic when** the user explicitly describes a visual style — use their style as-is, untouched:
- "watercolour colourful hippie style" → their words, their style
- "futuristic sci-fi neon astronaut" → their words, their style
- "photorealistic sunset over the ocean" → their words, their style
- "anime style cute panda" → their words, their style
- "oil painting impressionist flowers" → their words, their style

When in doubt, apply the default — most users describe *what* they want, not *how* it should look.

## Prompt construction

The user describes what they want in plain language. The agent transforms the prompt before sending.

### Rules

1. **Strip postcard/card words** — Never include: `card`, `greeting card`, `postcard`, `mockup`, `print`, `paper`, `envelope`, `frame`, `border`, `photograph`, `photo of`. These cause the model to generate an image OF a physical card instead of the artwork itself.
2. **If no style given, add aesthetic direction** — Translate the default doodle aesthetic above into appropriate prompt language for the subject. Don't mechanically paste a fixed string — craft style language that fits.
3. **Preserve the user's words exactly for the subject** — Do not add creative flourishes, extra elements, or descriptors the user didn't mention. If the user said "a happy panda holding a sign that says get well soon Dan", the subject stays exactly that. The style language does the visual steering, not the subject.
4. **Use "sign" not "banner"** — When the user wants text on something the character is holding, always use "sign". "Banner" causes DALL-E 3 to generate a ribbon/streamer composition instead of a handheld sign.
5. **Quote sign text explicitly and completely** — DALL-E 3 frequently truncates text, especially names at the end of a phrase. When text needs to appear in the image (e.g. on a sign), quote it explicitly in the prompt: `holding a sign with the text "GET WELL SOON DAN" written clearly and completely`. Never describe it as just "saying get well soon Dan" — the explicit quoting and "written clearly and completely" phrasing significantly improves text fidelity.

### Example transformations

| User says | Style detected? | Agent prompt (example — exact wording is yours to craft) |
|-----------|----------------|----------------------------------------------------------|
| "a happy panda holding a sign that says get well soon Dan" | No → add aesthetic direction | `"a happy panda holding a sign with the text 'GET WELL SOON DAN' written clearly and completely, hand-drawn doodle style, clean simple ink lines, light watercolor accents, white background, full character visible, cute and charming"` |
| "watercolour colourful hippie style flowers for Sarah" | Yes → pass through untouched | `"watercolour colourful hippie style flowers for Sarah"` |
| "futuristic sci-fi neon astronaut for Jake's birthday saying Happy Birthday Jake" | Yes → pass through untouched | `"futuristic sci-fi neon astronaut for Jake's birthday saying Happy Birthday Jake"` |
| "thank you with flowers" | No → add aesthetic direction | `"thank you with flowers, hand-drawn doodle style, clean simple ink lines, light watercolor accents, white background, cute and charming"` |
| "merry christmas with a snowman" | No → add aesthetic direction | `"merry christmas with a snowman, hand-drawn doodle style, clean simple ink lines, light watercolor accents, white background, full character visible"` |

## Postcard size → image size mapping

DALL-E 3 supports fixed sizes. Always use landscape for postcards:

| Postcard size | Image size | Aspect ratio |
|---------------|-----------|--------------|
| **4x6** | `1792x1024` | landscape |
| **6x9** | `1792x1024` | landscape |
| **11x6** | `1792x1024` | landscape |

The user never needs to specify dimensions — the skill maps it automatically from the postcard size.

## Full flow

### Step 1: Generate image

```bash
curl -s -X POST "https://image-generation-api-64k8.onrender.com/v1/image/generate" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"<constructed prompt>","size":"1792x1024"}'
```

Response is synchronous — no polling needed:

```json
{
  "provider": "openai",
  "model": "dall-e-3",
  "images": [{"url": "https://...", "b64_json": null}],
  "request_id": "..."
}
```

Extract `images[0].url` immediately.

### Step 2: Download and preview

**This step is MANDATORY and must happen immediately after extracting the URL — do not skip it, do not wait for the user to ask.**

Run both of these in sequence without any user prompt:

```bash
# 1. Download the image
curl -sL '<image_url>' -o ~/Downloads/postcard-artwork.png
```

Then immediately read `~/Downloads/postcard-artwork.png` using the Read tool so the image appears inline in the conversation. Do not just share the URL — the user cannot see temporary cloud URLs. The inline display is the only way they see the image.

After showing it, ask:

> *"Here's the generated artwork. Want to use this for your postcard, or would you like me to regenerate?"*

- If the user approves → continue to Step 3
- If the user wants changes → go back to Step 1 with an adjusted prompt

### Step 3: Convert to 2-page postcard PDF

PostalForm postcards require a 2-page PDF: page 1 = artwork, page 2 = mailing side (blank — PostalForm fills it).

Use Python + Pillow:

```python
from PIL import Image
import sys

img = Image.open(sys.argv[1]).convert("RGB")

sizes = {"4x6": (432, 288), "6x9": (648, 432), "11x6": (792, 432)}
size = sizes.get(sys.argv[2], sizes["6x9"])

img_resized = img.resize(size, Image.LANCZOS)
blank = Image.new("RGB", size, "white")
img_resized.save(sys.argv[3], "PDF", save_all=True, append_images=[blank])
```

Run:
```bash
python3 /tmp/postcard_pdf.py ~/Downloads/postcard-artwork.png 6x9 /tmp/postcard.pdf
```

### Step 4: Base64-encode and pass to PostalForm

```bash
base64 -i /tmp/postcard.pdf
```

Use in the PostalForm payload:
```json
"pdf": "data:application/pdf;base64,<base64-output>"
```

## Cost

Image generation is free at this endpoint (no x402 payment required). PostalForm mailing cost is ~$1–3 USDC, paid via x402.
