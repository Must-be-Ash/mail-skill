# AI Image Generation for Postcards

Generate postcard artwork via DALL-E 3 using the image generation API.

**Model:** DALL-E 3
**Endpoint:** `https://image-generation-api-64k8.onrender.com/v1/image/generate`

## Default style

When the user describes their postcard idea without specifying a visual style, apply the default doodle aesthetic by appending this suffix to their prompt:

> `, doodle style, hand-drawn, simple clean ink lines, whimsical illustration, light watercolor accents, white background, full character visible, cute and charming, simple composition`

This produces clean, charming, hand-crafted-feeling artwork that prints beautifully on postcards. Key words explained:
- **simple clean ink lines** — prevents heavy dark scratchy scribbles; keeps it light and readable
- **white background** — forces a clean uncluttered result
- **full character visible** — prevents DALL-E 3 from zooming into a face close-up
- **simple composition** — keeps the scene uncluttered and postcard-appropriate

**Apply the default style when** the user's description focuses on subject or message but gives no style cues:
- "a happy panda holding a sign" → apply default
- "flowers and butterflies for grandma" → apply default
- "a cat wearing a birthday hat" → apply default
- "get well soon with a sloth" → apply default

**Do NOT apply the default style when** the user explicitly describes a visual style:
- "watercolour colourful hippie style" → respect it, don't override
- "futuristic sci-fi neon astronaut" → respect it, don't override
- "photorealistic sunset over the ocean" → respect it, don't override
- "anime style cute panda" → respect it, don't override
- "oil painting impressionist flowers" → respect it, don't override

When in doubt, apply the default — most users describe *what* they want, not *how* it should look.

## Prompt construction

The user describes what they want in plain language. The agent transforms the prompt before sending.

### Rules

1. **Strip postcard/card words** — Never include: `card`, `greeting card`, `postcard`, `mockup`, `print`, `paper`, `envelope`, `frame`, `border`, `photograph`, `photo of`. These cause the model to generate an image OF a physical card instead of the artwork itself.
2. **Apply default style if no style specified** — Append `, doodle style, hand-drawn, simple clean ink lines, whimsical illustration, light watercolor accents, white background, full character visible, cute and charming, simple composition` unless the user already indicated a visual style.
3. **Preserve the user's words exactly** — Do not add creative flourishes, extra elements, or descriptors the user didn't mention. If the user said "a happy panda holding a sign that says get well soon Dan", the subject in the prompt should be exactly that — not "a cheerful happy panda surrounded by balloons and flowers holding a colorful banner". The user's words are the prompt; the style suffix does the visual steering.
4. **Use "sign" not "banner"** — When the user wants text on something the character is holding, always use "sign" in the prompt. "Banner" causes DALL-E 3 to generate a ribbon/streamer composition instead of a handheld sign.

### Example transformations

| User says | Style detected? | Agent prompt |
|-----------|----------------|--------------|
| "a happy panda holding a sign that says get well soon Dan" | No → use default | `"a happy panda holding a sign that says get well soon Dan, doodle style, hand-drawn, simple clean ink lines, whimsical illustration, light watercolor accents, white background, full character visible, cute and charming, simple composition"` |
| "watercolour colourful hippie style flowers for Sarah" | Yes | `"watercolour colourful hippie style flowers for Sarah"` |
| "futuristic sci-fi neon astronaut for Jake's birthday saying Happy Birthday Jake" | Yes | `"futuristic sci-fi neon astronaut saying Happy Birthday Jake"` |
| "thank you with flowers" | No → use default | `"thank you with flowers, doodle style, hand-drawn, simple clean ink lines, whimsical illustration, light watercolor accents, white background, full character visible, cute and charming, simple composition"` |
| "merry christmas with a snowman" | No → use default | `"merry christmas with a snowman, doodle style, hand-drawn, simple clean ink lines, whimsical illustration, light watercolor accents, white background, full character visible, cute and charming, simple composition"` |

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

```bash
curl -sL '<image_url>' -o /tmp/postcard-artwork.png
```

**Always show the image to the user before proceeding.** Read the downloaded file so the user can see it inline in the conversation, then ask:

> *"Here's the generated artwork. Want to use this for your postcard, or would you like me to generate a new one?"*

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
python3 /tmp/postcard_pdf.py /tmp/postcard-artwork.png 6x9 /tmp/postcard.pdf
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
