# AI Image Generation for Postcards

Generate postcard artwork via recraft-v3 on fal.ai, paid with USDC on Base via x402.

**Model:** recraft-v3 ($0.04/image)
**Endpoint:** `https://fal.x402.paysponge.com/fal-ai/recraft-v3`

## Automatic prompt construction

The user describes what they want in plain language. The agent MUST transform the user's prompt before sending it to the model. This ensures the output is a flat graphic suitable for printing — not a photo of a card.

### Rules

1. **Always set** `"style":"digital_illustration"` in the request body
2. **Always append** this suffix to the user's prompt: `, flat illustration, detailed shading and soft watercolor textures, warm muted color palette, storybook illustration quality, polished refined artwork`
3. **Never include** these words in the prompt: `card`, `greeting card`, `postcard`, `mockup`, `print`, `paper`, `envelope`, `frame`, `border`, `photograph`, `photo of`. These cause the model to generate an image OF a physical card instead of the artwork itself.
4. **Always set** `image_size` based on the postcard size (see ratio mapping below)

### Example transformation

User says: "a birthday card with a giraffe saying happy birthday Ash"

Agent constructs:
```json
{
  "prompt": "a cute giraffe with text saying Happy Birthday Ash, flat illustration, detailed shading and soft watercolor textures, warm muted color palette, storybook illustration quality, polished refined artwork",
  "image_size": {"width": 1800, "height": 1200},
  "style": "digital_illustration"
}
```

Notice: "card" was removed, "flat illustration" and style modifiers were appended, and the user's intent was preserved.

### More examples

| User says | Agent prompt |
|-----------|-------------|
| "get well soon card with a panda astronaut for Dan" | "a cute panda wearing a white astronaut suit waving happily with text saying Get Well Soon Dan, surrounded by stars and planets, solid light blue background, flat illustration, detailed shading and soft watercolor textures, warm muted color palette, storybook illustration quality, polished refined artwork" |
| "thank you with flowers" | "a beautiful bouquet of colorful flowers with text saying Thank You, flat illustration, detailed shading and soft watercolor textures, warm muted color palette, storybook illustration quality, polished refined artwork" |
| "merry christmas with a snowman" | "a cheerful snowman wearing a scarf in a snowy scene with text saying Merry Christmas, flat illustration, detailed shading and soft watercolor textures, warm muted color palette, storybook illustration quality, polished refined artwork" |

## Postcard size → image ratio mapping

Always auto-select the correct dimensions based on the chosen postcard size:

| Postcard size | Image dimensions | Aspect ratio |
|---------------|-----------------|--------------|
| **4x6** | `{"width": 1800, "height": 1200}` | 3:2 landscape |
| **6x9** | `{"width": 1800, "height": 1200}` | 3:2 landscape |
| **11x6** | `{"width": 2200, "height": 1200}` | 11:6 landscape |

The user never needs to specify ratio or dimensions — the skill maps it automatically from the postcard size.

## Full flow

### Step 1: Generate image via x402

```bash
npx awal@latest x402 pay 'https://fal.x402.paysponge.com/fal-ai/recraft-v3' \
  -X POST \
  -d '{"prompt":"<constructed prompt>","image_size":{"width":1800,"height":1200},"style":"digital_illustration"}' \
  --json
```

### Step 2: Poll for result

The response contains a `response_url`. Poll with curl (free, no payment):

```bash
curl -s '<response_url>'
```

Result includes an `images` array:
```json
{
  "images": [{"url": "https://v3b.fal.media/files/...", "content_type": "image/webp"}]
}
```

If response shows `"status": "IN_QUEUE"` or `"IN_PROGRESS"`, wait 3 seconds and poll again. Typical generation time: ~5-8 seconds.

### Step 3: Download and preview

```bash
curl -sL '<image_url>' -o /tmp/postcard-artwork.webp
```

**Always show the image to the user before proceeding.** Read the downloaded file so the user can see it inline, then ask:

> *"Here's the generated artwork. Want to use this for your postcard, or would you like me to generate a new one?"*

- If the user approves → continue to Step 4
- If the user wants changes → go back to Step 1 with an adjusted prompt (costs another $0.04)
- The user can iterate as many times as they want before committing

### Step 4: Convert to 2-page postcard PDF

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
python3 /tmp/postcard_pdf.py /tmp/postcard-artwork.webp 6x9 /tmp/postcard.pdf
```

### Step 5: Base64-encode and pass to PostalForm

```bash
base64 -i /tmp/postcard.pdf
```

Use in the PostalForm payload:
```json
"pdf": "data:application/pdf;base64,<base64-output>"
```

## Cost

$0.04 USDC per image generation, in addition to PostalForm mailing cost.
