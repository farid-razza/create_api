# Create API

One endpoint that turns **text into a marketing image**.

You send some text and say which platform it is for. The service picks the right
prompt, generates the image, and returns it at the exact pixel size that platform
needs.

```
POST /image-transform/create
```

**Generated images contain no text.** No words, headlines, captions, prices,
watermarks or logos. The text you send decides *what the scene shows* — it is never
drawn into the image. (One exception: Google Ads *logo* dimensions, covered below.)

---

## Contents

1. [The one thing to understand](#1-the-one-thing-to-understand)
2. [Request fields](#2-request-fields)
3. [Platforms and dimensions](#3-platforms-and-dimensions)
4. [What text to send](#4-what-text-to-send)
5. [The response](#5-the-response)
6. [Errors](#6-errors)
7. [Locales](#7-locales)
8. [Building a UI on this](#8-building-a-ui-on-this)
9. [Setup and running](#9-setup-and-running)
10. [Files](#10-files)
11. [The prompts](#11-the-prompts)

---

## 1. The one thing to understand

**`image_type` decides which prompt runs. `dimension` decides the pixel size.**

That is the whole design. Same URL every time; the payload changes the behaviour.

```jsonc
// a Facebook / Instagram story
{ "input_text": "Book your fall security check today.",
  "image_type": "meta", "dimension": "meta_story" }          → 1080 × 1920

// a website hero from a brand kit
{ "brand_kit": { ... },
  "image_type": "website", "dimension": "website_hero" }     → 1920 × 1080

// no platform named — a general marketing image
{ "input_text": "Sharma Dental Care. Family dentistry since 2011..." }
                                                             → 1024 × 1024
```

That last one is `image_type: "generic"`, the default. If you send no
`image_type`, you get a general marketing image at a standard size.

**You get 3 images per request** (three different angles on the same business).
Google Ads logo dimensions return **1**.

---

## 2. Request fields

| Field | Required | Default | Notes |
|---|---|---|---|
| `input_text` | yes\* | — | The source text. Aliases: `website_text`, `post_text`. Minimum 20 characters. |
| `brand_kit` | yes\* | — | Object or JSON string. **Website hero only.** |
| `image_type` | no | `generic` | `generic` · `meta` · `gbp` · `google_ads` · `website` |
| `dimension` | no | that platform's default | A named key from the table below — **not** a `WxH` string. |
| `locale` | no | none | `IN` or `en-IN`. Also accepted as `country`. |
| `variations` | no | `3` (logos: `1`) | 1 to 3. |
| `quality` | no | `low` | `low` · `medium` · `high` · `auto` |
| `size` | no | — | Legacy. A raw `1024x1024` / `1024x1536` / `1536x1024` / `auto`, **`generic` only**. New integrations should use `dimension`. |

\* Send **`input_text` or `brand_kit`**. A website hero can run on the brand kit
alone; everything else needs `input_text`.

> `quality` defaults to `low`, which is the usual reason an image looks soft. Use
> `"quality": "high"` for anything a client will see. It costs more per image.

**`dimension` is a name, not a size.** Pixel sizes are not unique — Google Ads
"Square" and "Logo Square" are both 1200×1200, and Google Ads landscape (1200×628)
is nearly Meta landscape (1200×630). Only the name says which you want.

---

## 3. Platforms and dimensions

Get this list from **`GET /catalog`** rather than hard-coding it.

### generic — no particular platform (the default)

| `dimension` | You receive |
|---|---|
| `generic_square` ← default | 1024 × 1024 |
| `generic_portrait` | 1024 × 1536 |
| `generic_landscape` | 1536 × 1024 |
| `generic_auto` | model chooses |

### meta — Facebook / Instagram

| `dimension` | You receive | |
|---|---|---|
| `meta_square` ← default | 1080 × 1080 | Feed square |
| `meta_portrait` | 1080 × 1350 | Feed portrait |
| `meta_story` | 1080 × 1920 | Story / Reel |
| `meta_landscape` | 1200 × 630 | Facebook landscape |

### gbp — Google Business Profile

| `dimension` | You receive | |
|---|---|---|
| `gbp_square` ← default | 1080 × 1080 | Recommended |
| `gbp_square_min` | 720 × 720 | Minimum accepted |

### google_ads — Performance Max

| `dimension` | You receive | |
|---|---|---|
| `gads_square` ← default | 1200 × 1200 | photo |
| `gads_landscape` | 1200 × 628 | photo |
| `gads_portrait` | 1200 × 1500 | photo |
| `gads_logo_square` | 1200 × 1200 | **logo — transparent PNG, 1 image** |
| `gads_logo_wide` | 1200 × 300 | **logo — transparent PNG, 1 image** |

### website

| `dimension` | You receive | |
|---|---|---|
| `website_hero` ← default | 1920 × 1080 | takes a brand kit |
| `website_hero_banner` | 1920 × 600 | takes a brand kit |
| `website_section` | 1200 × 628 | that section's own text |
| `website_sidebar_250` | 300 × 250 | card |
| `website_sidebar_600` | 300 × 600 | card |
| `website_sidebar_square` | 300 × 300 | card |
| `website_sidebar_portrait` | 300 × 375 | card |

**Mixing platforms is rejected.** `image_type: "meta"` with
`dimension: "gads_square"` returns 400.

> **Why the delivered size is not what the model generates.** GPT-Image-2 only
> accepts sizes where both edges divide by 16, the pixel count is 655,360–8,294,400,
> and the ratio is at most 3:1. Almost no real platform size qualifies. So each
> dimension is generated at the nearest valid size and centre-cropped to the exact
> delivered pixels. Images are scaled to fill and cropped — **never stretched**.
> You do not need to do anything; you always receive the size in the table.

---

## 4. What text to send

The prompts read the text differently, so send the right kind for the platform.

| `image_type` / dimension | Send | In which field |
|---|---|---|
| `generic` | the business's website text | `input_text` |
| `meta` | the social post text | `input_text` |
| `gbp` | the GBP post text | `input_text` |
| `google_ads` photo sizes | the ad text, or a description of the business | `input_text` |
| `google_ads` logo sizes | a description of the business, so the symbol suits it | `input_text` |
| `website_hero`, `website_hero_banner` | the brand kit (preferred), or plain text about the business | `brand_kit`, or `input_text` |
| `website_section` | that section's own content | `input_text` |
| `website_sidebar_*` | that card's own content | `input_text` |

**Why it matters:** the Meta prompt reads the text as *one post* and depicts that
message. The website hero prompt reads it as *a description of the business* and
depicts what the business does. Sending a business description to `meta` still
works, but you get a generic business shot rather than a post image.

The full text is passed through as-is. URLs, phone numbers and hashtags are left
in; the prompt tells the model not to draw them.

### The brand kit

For `website_hero` and `website_hero_banner`, send the whole brand kit. Only these
**four** fields are used; everything else is ignored:

- `contents_long_description`
- `contents_area_of_focus`
- `style_inspirations_brand_personality_traits`
- `brand_archetype`

Field names match loosely — `brand_archetype`, `brandArchetype` and
`Brand Archetype` all work. The brand kit is accepted as an object **or** as a
JSON string, including one with stray text stuck to it from a copy/paste.

If a brand kit is sent with none of those four fields, you get a 400 naming them.

---

## 5. The response

```jsonc
{
  "level": "Create",
  "image_type": "meta",
  "dimension": "meta_story",
  "size": "1080x1920",           // exactly what you received
  "kind": "photo",               // "photo" or "logo"
  "transparent_png": false,      // true for logo dimensions
  "quality": "low",
  "locale": "en-IN",             // null if none sent
  "country": "IN",
  "variations": 3,
  "results": [
    {
      "index": 1,
      "angle": "Service in action",     // null whenever only one image is returned
      "ok": true,
      "image_base64": "iVBORw0KGgo...", // raw base64, no "data:" prefix
      "mime": "image/png",
      "render_prompt": "...",           // the exact prompt used
      "usage": { "input": 724, "output": 196, "total": 920 },
      "timing_ms": { "render": 20500, "total": 20500 }
    }
    // ...one entry per variation
  ],
  "usage": { "input": 2181, "output": 588, "total": 2769 },  // all combined
  "timing_ms": { "total": 20500 }        // wall clock, not the sum
}
```

**Always a `results` array**, even when it holds one entry.

**One variation can fail without losing the others.** That entry comes back with
`"ok": false` and an `error`, and the rest still contain images. Only if every
variation fails does the request return an error status.

The three variations run **in parallel**, so three images take about as long as one.

**Saving the images:**

```javascript
response.results
  .filter(r => r.ok)
  .forEach(r => { img.src = "data:image/png;base64," + r.image_base64; });
```

---

## 6. Errors

Every failure returns the same shape:

```json
{ "error": "a message explaining what to fix" }
```

**There is only ever one shape** — no arrays, no nested detail objects. You never
need to branch on the response body.

**A field of the wrong type is rejected, not ignored.** `"dimension": 5` returns
`dimension must be a string` rather than quietly falling back to the default and
returning a correct-looking image at the wrong size. Fields the API does not know
are ignored.

| Status | Meaning | Examples |
|---|---|---|
| **400** | something in the request, or the content | `input_text is required (or send a brand_kit for a website hero)` · `image_type must be one of: generic, meta, gbp, google_ads, website` · `dimension 'gads_square' belongs to image_type 'google_ads', not 'meta'` · `dimension 'gads_logo_square' returns at most 1 image(s); 3 requested` · `brand_kit applies to a website hero only` · `unknown locale 'ZZ'` (the message lists every valid value) · `dimension must be a string` · **`request rejected by the OpenAI safety system`** (only when *every* variation was refused) |
| **502** | the image model failed | `image model error: ...` · `image model returned no image` |
| **500** | the service is misconfigured | `OPENAI_API_KEY is not set` · `prompt.txt missing` |

> **Safety blocks are not consistent.** OpenAI sometimes flags an image for a
> category such as `illicit` — security, locksmith and alarm businesses trip this
> most. The same request can pass on one attempt and be blocked on the next, so
> **retrying often just works**. It is not retried automatically. Every prompt
> already asks for lawful, safe scenes to reduce false flags.

---

## 7. Locales

The locale decides **who appears in the image**. `en-IN` produces Indian people,
`ja-JP` produces Japanese people. It applies to every platform.

Send either form:

```json
{ "locale": "IN" }      { "locale": "en-IN" }      { "country": "IN" }
```

21 markets: `US CA GB IE AU NZ IN SG MY PH FR DE ES IT NL MX BR JP KR ZA AE`

Get the list from **`GET /locales`**.

`locale` is **optional**, and there is no default. Omit it and the image model
chooses the people itself — so if the market matters, always send it. The response
echoes `locale: null` when none was used.

Logo dimensions ignore locale — a logo has no people in it.

---

## 8. Building a UI on this

**Build your controls from `GET /catalog`.** It is generated from the backend's own
dimension table, so it cannot fall out of step with what the API accepts.

```jsonc
{
  "default_image_type": "generic",
  "image_types": [
    {
      "image_type": "meta",
      "default_dimension": "meta_square",
      "dimensions": [
        { "dimension": "meta_story", "size": "1080x1920",
          "width": 1080, "height": 1920,
          "aspect": "tall vertical full-screen (9:16)",
          "kind": "photo", "transparent_png": false,
          "max_variations": 3 }
      ]
    }
  ]
}
```

Suggested flow:

1. Platform dropdown ← `image_types[].image_type`
2. Size dropdown ← that platform's `dimensions[]`, preselect `default_dimension`
3. Variations control ← cap at `max_variations`; **hide or disable it when
   `max_variations` is 1** (the logo assets), so the user cannot send a request
   that would be rejected
4. Show `transparent_png: true` as a hint that the result has no background
5. Locale dropdown ← `GET /locales`

Nothing needs hard-coding. Adding a platform or size later shows up in `/catalog`
automatically.

---

## 9. Setup and running

```bash
cd create-onestep-api
pip install -r requirements.txt
```

Create a `.env` in this folder:

```
OPENAI_API_KEY=sk-...
```

**Windows PowerShell** — the `cd` is required so `uvicorn` can import `app`:

```powershell
cd D:\Downloads\transform-image-python\create-onestep-api
D:\Downloads\transform-image-python\.venv\Scripts\python.exe -m uvicorn app:app --reload --port 8300
```

**Anywhere else:**

```bash
uvicorn app:app --reload --port 8300
```

Check it is alive:

```bash
curl http://localhost:8300/health      # {"ok": true}
```

Interactive API docs: <http://localhost:8300/docs>

### Example calls

```bash
# Instagram story
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{"input_text":"Book your fall security check today. Evening slots available.",
       "image_type":"meta","dimension":"meta_story","locale":"en-US","quality":"high"}'

# Google Ads wide logo — transparent PNG, one image
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{"input_text":"A family-run bakery making sourdough and pastries since 1994.",
       "image_type":"google_ads","dimension":"gads_logo_wide"}'

# Website hero from a brand kit — no input_text needed
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{"image_type":"website","dimension":"website_hero","quality":"high",
       "brand_kit":{"contents_long_description":"A family-owned painting company serving Greater Philadelphia since 1979.",
                    "contents_area_of_focus":["Residential painting","Cabinet painting"],
                    "style_inspirations_brand_personality_traits":"Dependable, Family-Oriented",
                    "brand_archetype":"Everyman"}}'
```

**PowerShell** quoting differs:

```powershell
curl.exe -X POST http://localhost:8300/image-transform/create `
  -H "Content-Type: application/json" `
  -d '{\"input_text\":\"Book your fall security check today.\",\"image_type\":\"meta\"}'
```

### Environment variables

| Variable | Required | Default | What it does |
|---|---|---|---|
| `OPENAI_API_KEY` | **yes** | — | your OpenAI key |
| `OPENAI_IMAGE_MODEL` | no | `gpt-image-2` | the image model |
| `MAX_WEBSITE_CHARS` | no | `12000` | text is truncated to this |
| `OPENAI_TIMEOUT` | no | `300` | seconds to wait per call |
| `LOGO_BACKGROUND_TOLERANCE` | no | `30` | 0–255. How aggressively a logo's background is made transparent. Higher removes more but can eat into the logo. |

The key is read on the first request, not at startup, so the server starts and
`/health` answers without one.

---

## 10. Files

```
create-onestep-api/
├── app.py                 FastAPI entrypoint, CORS, /health
├── router.py              the routes
├── controller.py          reads the JSON body and validates it
├── service.py             resolve dimension → prompt → generate → exact size
├── dimensions.py          the 22 dimensions: generated size vs delivered size
├── platform_prompts.py    the meta / gbp / google_ads / website prompts
├── prompt.txt             the `generic` prompt
├── variations.txt         the three creative angles
├── brand_kit.py           pulls the four hero fields out of a brand kit
├── image_postprocess.py   cover-crop to exact size; logo background → transparent
├── locales.py             the 21 markets
└── requirements.txt
```

Two files to read first if you are changing behaviour: **`platform_prompts.py`**
(how images turn out) and **`dimensions.py`** (everything about sizes).

`dimensions.py` validates every generation size **at import**. If a size breaks
GPT-Image-2's rules the service refuses to start, rather than failing later on one
dimension in production.

---

## 11. The prompts

| Platform | Prompt |
|---|---|
| `generic` | `prompt.txt` + an angle from `variations.txt` |
| `meta`, `gbp`, `google_ads`, `website` | `platform_prompts.py` |

The three angles are **Service in action**, **Customer moment** and **The place** —
what to photograph. Everything else in the prompt stays identical between them, so
the images differ in framing while staying about the same business.

The locale block and the variation angle are **appended** to a platform prompt
rather than written into it, so the platform prompts stay exactly as written.

**Prompt files are read once and then cached — restart the server after editing
any of them**, including `locales.py`. Editing a prompt on a running server has no
effect.

### The no-text rule, and its one exception

Every prompt forbids text, invented awards, trust badges, review stars and
fabricated business details. Images get published, so a garbled phone number or a
made-up award would be a real problem.

**The exception is the two Google Ads logo dimensions**, which must produce an
actual logo. Even there the no-text rule holds: the logo is a purely graphical
symbol with **no business name, wordmark, monogram or initials**. It is generated
on a plain white background, which is then flood-filled away to give you a
transparent PNG.
