# Create API (one-step)

Give it the text of a business's website. It returns an advertising photograph for that
business.

**One AI call.** The website text goes straight into the image prompt — there is no
planner.

> There is a second service, `create-api`, that does the same job in two calls. It is
> slower but returns structured business data.

---

## Contents

1. [What it does](#1-what-it-does)
2. [Files](#2-files)
3. [Setup](#3-setup)
4. [Running the server](#4-running-the-server)
5. [Calling the endpoint](#5-calling-the-endpoint)
6. [The response](#6-the-response)
7. [Errors](#7-errors)
8. [Locales](#8-locales)
9. [Environment variables](#10-environment-variables)
10. [The prompt](#11-the-prompt)

---

## 1. What it does

**One endpoint:**

```
POST /image-transform/create
```

**The pipeline:**

```
1. You send the website text (and optionally a locale, and how many images you want)
2. The text is placed into the prompt, along with the rules
3. GPT-Image-2 generates each image — the calls run in parallel
4. You get them back as base64, in a results array
```

Ask for three and three calls run at once, so three images take about as long as one.
Each gets a different creative angle from `variations.txt` — what to photograph.
Everything else in the prompt is identical, so the images differ in framing and moment
while staying about the same business.

The image contains **no text of any kind** — no headline, price, sign copy, logo or
lettering. It is a purely visual photograph.

There is no uploaded photo and no brand kit. Everything comes from the website text.

---

## 2. Files

```
create-onestep-api/
├── app.py            FastAPI entrypoint, CORS, /health
├── router.py         the routes
├── controller.py     reads the JSON body and validates it
├── service.py        builds the prompt and makes the one call
├── prompt.txt        the prompt template
├── variations.txt    the creative angles, used when asking for more than one image
├── locales.py        the 21 supported markets
└── requirements.txt
```

| File | Responsibility |
|---|---|
| `app.py` | starts the app, enables CORS, adds `/health` |
| `router.py` | maps URL paths to the controller |
| `controller.py` | checks the request is valid, turns errors into status codes |
| `service.py` | fills the prompt template, calls the image model, times it |
| `prompt.txt` | plain text with placeholders, editable |
| `variations.txt` | the angles, one per block, editable |
| `locales.py` | country → locale, and who lives there |

---

## 3. Setup

```bash
cd create-onestep-api
pip install -r requirements.txt
```

Create a `.env` file in this folder:

```
OPENAI_API_KEY=sk-...
```

`.env.example` lists every available setting.

---

## 4. Running the server

**Windows PowerShell** — the `cd` is required, because `prompt.txt` is found relative to
it:

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
curl http://localhost:8300/health
```
```json
{"ok": true}
```

Interactive API docs: <http://localhost:8300/docs>

Port 8300, so it can run at the same time as `create-api` on 8200.

---

## 5. Calling the endpoint

`POST /image-transform/create` with a JSON body. **The request is identical to
`create-api`** — only the port differs, so you can switch between the two services
without changing client code.

| Field | Required | Default | Values |
|---|---|---|---|
| `website_text` | **yes** | — | the site's text, minimum 20 characters |
| `locale` (or `country`) | no | none | `IN` or `en-IN` — either form. See [Locales](#8-locales) |
| `size` | no | `1024x1024` | `1024x1024`, `1024x1536`, `1536x1024`, `auto` |
| `quality` | no | `low` | `low`, `medium`, `high`, `auto` |
| `variations` | no | `1` | `1` to `3` — how many differently-angled images to return |

```bash
curl -X POST http://localhost:8300/image-transform/create \
  -H "Content-Type: application/json" \
  -d '{
    "website_text": "Sharma Dental Care, Andheri West. Family and cosmetic dentistry since 2011. Painless root canals, whitening, braces and implants. Same-day appointments.",
    "locale": "en-IN",
    "size": "1024x1024",
    "quality": "low",
    "variations": 3
  }'
```

**PowerShell** (quoting differs):

```powershell
curl.exe -X POST http://localhost:8300/image-transform/create `
  -H "Content-Type: application/json" `
  -d '{\"website_text\":\"Sharma Dental Care. Family dentistry since 2011.\",\"locale\":\"en-IN\"}'
```

---

## 6. The response

```jsonc
{
  "level": "Create",
  "size": "1024x1024",
  "quality": "low",
  "locale": "en-IN",
  "country": "IN",
  "variations": 3,
  "results": [
    {
      "index": 1,
      "angle": "Service in action",
      "ok": true,
      "image_base64": "iVBORw0KGgo...",   // raw base64, no "data:" prefix
      "mime": "image/png",
      "render_prompt": "...",              // the exact text sent for this image
      "usage": { "input": 724, "output": 196, "total": 920 },
      "timing_ms": { "render": 29600, "total": 29600 }
    }
    // ...one entry per variation
  ],
  "usage": { "input": 2181, "output": 588, "total": 2769 },   // all variations combined
  "timing_ms": { "total": 29600 }                             // wall clock, not the sum
}
```

**Always a results array**, even when `variations` is 1 — then it holds one entry. The
three angles are **Service in action**, **Customer moment** and **The place**.

**One angle can fail without losing the others.** That entry comes back with
`"ok": false` and an `error`, and the rest still contain images. Only when every
variation fails does the endpoint return an error status.

There is **no `business` block** — that comes from the planner, which this service does
not have.

**Saving the images:**

```python
import base64
for r in response["results"]:
    if r["ok"]:
        open(f"out{r['index']}.png", "wb").write(base64.b64decode(r["image_base64"]))
```

---

## 7. Errors

Every failure returns `{"error": "..."}` with a status that says whose problem it is.

| Status | Meaning | Examples |
|---|---|---|
| **400** | the request, or the content | `website_text is required` · `website_text is too short to describe a business` · `unknown locale 'ZZ'` (the message lists every valid value) · `size must be one of: ...` · `request rejected by the OpenAI safety system` |
| **502** | the OpenAI call failed | `image model error: ...` · `image model returned no image` |
| **500** | this service is misconfigured | `OPENAI_API_KEY is not set` · `prompt.txt missing` |

Safety rejections are `400` rather than `500` because they are a fact about the input,
not a server fault.

---

## 8. Locales

The locale decides **who appears in the image**. `en-IN` produces Indian people, `ja-JP`
produces Japanese people, and so on.

Send either form — a country code or a locale string:

```json
{"locale": "IN"}       {"locale": "en-IN"}       {"country": "IN"}
```

21 markets are supported:

```
US  CA  GB  IE  AU  NZ  IN  SG  MY  PH  FR
DE  ES  IT  NL  MX  BR  JP  KR  ZA  AE
```

To get the list programmatically:

```bash
curl http://localhost:8300/locales
```

`locale` is optional. Without it, the image model is given no instruction about who the
people are.

In `prompt.txt` the locale appears **twice** — once as scene direction and once as a hard
rule covering everyone in the frame, including the background. Both blocks are visible in
the file and can be edited.

**Only the locale affects the image.** `locales.py` also stores each market's currency,
timezone, date format and first day of week, so this service agrees with the rest of the
platform — but a Create image contains no text, so a currency symbol or a date has
nothing to appear on. Those four fields are stored and unused.

`locales.py` also holds a `people` description per market, written for this service. It
is what turns `en-IN` into "Indian people".

For a country with more than one large ethnic group the description is **weighted** — it
says which group most faces should be, with the others still present but in the minority.
Listing them as equal alternatives made the model choose at random, which produced people
who did not look like the region. Read these before shipping — they are editorial
wording, not a spec.

---

---

## 9. Environment variables

| Variable | Required | Default | What it does |
|---|---|---|---|
| `OPENAI_API_KEY` | **yes** | — | your OpenAI key |
| `OPENAI_IMAGE_MODEL` | no | `gpt-image-2` | the image model |
| `MAX_WEBSITE_CHARS` | no | `12000` | website text is truncated to this |
| `OPENAI_TIMEOUT` | no | `300` | seconds to wait for the call |

The key is read on the first request, not at startup, so the server starts and `/health`
answers without one.

`MAX_WEBSITE_CHARS` is lower here than in `create-api` (12000 vs 20000). This service
puts the whole website text into the image prompt, so an oversized prompt is a failure
mode the two-step service does not have. A 12,664-character prompt was tested and worked;
12000 stays just inside that.

---

## 10. The prompt

The whole instruction lives in **`prompt.txt`**. It is plain text and can be edited
without touching code. It is read once when the process starts, so a restart is needed
after editing.

Placeholders: `{website_text}`, `{people_scene}`, `{variation_angle}`, `{people_rule}`.
The people placeholders are filled from the locale, the angle from `variations.txt`.

**`variations.txt`** holds the angles, one per block separated by a line of `---`, each
labelled by its `# name` line. Every angle requires people in frame, so the locale rule
always has faces to apply to.

The prompt tells the model to work out what the business is from its website text and
photograph a fitting scene, forbids inventing a claim, award, price, rating or statistic,
requires a believable photograph rather than an illustration, and bans text of any kind.

Editing either file changes output quality — re-test before quoting any number in this
file. **Both are read once at startup, so restart the server after editing.** The same
applies to `locales.py`.
