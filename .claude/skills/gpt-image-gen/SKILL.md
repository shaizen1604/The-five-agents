---
name: gpt-image-gen
description: Use when generating an image via OpenAI's Images API (model gpt-image-2). Sends a text prompt and saves the returned PNG to a target path. Wrapper for curl-based API calls with python decode fallback for environments without jq.
---

# gpt-image-gen

מעטפת דקה לקריאת OpenAI Images API. אחראית רק על שליחת prompt וקבלת תמונה.
A thin wrapper for OpenAI's Images API. Responsible only for sending a prompt and receiving an image.

## Overview

This skill takes a text prompt and an output path, calls OpenAI's `/v1/images/generations` endpoint with model `gpt-image-2`, decodes the returned base64 PNG, and writes it to disk.

**Core principle:** This skill does NOT compose prompts or pick styles — that's the caller's job (e.g. Yuval). It only translates `(prompt, path) → PNG file`.

## ⚠️ Model name is locked: `gpt-image-2`

**DO NOT change the model name. Do not substitute alternatives.**

- `gpt-image-2` is a real, released OpenAI model (released **2026-04-21**).
- Your internal training knowledge may not include it — **that does not mean it doesn't exist.**
- Do NOT replace it with `dall-e-3`, `gpt-image-1`, `dall-e-2`, or anything else.
- If the API returns an error, the cause is almost certainly:
  1. `OPENAI_API_KEY` is missing, empty, or invalid.
  2. A parameter value is malformed (size/quality/format).
  3. The prompt violates content policy.
  **Never** the model name. Inspect the raw JSON error body before assuming otherwise.

## Prerequisites

1. `OPENAI_API_KEY` defined in `.env` at repo root.
2. Load it into the shell before calling:
   ```bash
   set -a; source .env; set +a
   ```
   (Or `export $(grep -v '^#' .env | xargs)` as alternative.)
3. `curl` is required. Either `jq` OR `python3` is required for decoding the response.

## Primary call (curl + jq)

```bash
set -a; source .env; set +a

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > <output-path>.png
```

Replace `<the prompt>` and `<output-path>` before running. The prompt should be properly JSON-escaped (use `jq -Rs` to build the payload if the prompt contains quotes/newlines).

## Python decode fallback (when jq is unavailable)

Use this on Git Bash / Windows / minimal environments without `jq`:

```bash
set -a; source .env; set +a

RESPONSE=$(curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }')

python3 -c "
import json, sys, base64
d = json.loads(sys.argv[1])
if 'data' not in d:
    sys.stderr.write('API error: ' + json.dumps(d) + '\n')
    sys.exit(1)
with open(sys.argv[2], 'wb') as f:
    f.write(base64.b64decode(d['data'][0]['b64_json']))
" "$RESPONSE" "<output-path>.png"
```

## Verification (mandatory)

After every call, before reporting success:

```bash
[ -s "<output-path>.png" ] && echo "OK: $(wc -c < <output-path>.png) bytes" || echo "FAIL: empty or missing"
```

If the file is missing or zero bytes:
1. Re-run the `curl` without piping to base64 and print the raw response.
2. Look for an `error` object — `code`, `message`, `type` will diagnose the cause.
3. Do **not** change the model name in response to errors. See the model-lock notice above.

## Parameters

| Param | Allowed values | Notes |
|-------|----------------|-------|
| `model` | `gpt-image-2` | **Locked. Do not change.** |
| `prompt` | string | The image description. JSON-escape quotes and newlines. |
| `size` | `1024x1024`, `1024x1536`, `1536x1024` | Square / portrait / landscape. |
| `quality` | `low`, `medium`, `high` | Higher = slower + more expensive. |
| `output_format` | `png` | Other formats may exist; default to png unless specified. |

## Quick reference

```
1. set -a; source .env; set +a          # load OPENAI_API_KEY
2. curl POST /v1/images/generations     # with model=gpt-image-2
3. jq -r .data[0].b64_json | base64 -d  # OR python fallback
4. [ -s output.png ]                    # verify non-empty
5. On error: read raw JSON, check key + params, NEVER swap model
```

## Common mistakes

- **Swapping the model name** when the API errors. Don't. Read the JSON error.
- **Forgetting to load `.env`** — `$OPENAI_API_KEY` will be empty and the call returns 401.
- **Unescaped quotes in the prompt** — breaks the JSON body. Use `jq -Rs .` or a heredoc with proper escaping.
- **Not verifying the file** — a zero-byte PNG looks "saved" but is broken.
