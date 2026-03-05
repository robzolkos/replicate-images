---
name: Image Generation with Replicate
description: This skill should be used when the user asks to "generate an image", "create an image", "make a picture", "generate a photo", "create artwork", "make an illustration", "generate a logo", "create a visual", "create a cover image for a fizzy card", or any request involving AI image generation, image creation, or producing images from text descriptions.
---

# Image Generation with Replicate

Generate images from text prompts using the Replicate API. This skill covers the full workflow: constructing the API request, submitting it, polling for completion, and returning the image URL.

## Prerequisites

The environment variable `REPLICATE_API_TOKEN` must be set. If it is not available, inform the user and instruct them to get a token from https://replicate.com/account/api-tokens.

## Workflow

### 1. Choose a Model

Default to `google/nano-banana-2` unless the user specifies a different model. Common alternatives:

| Model | Identifier | Notes |
|-------|-----------|-------|
| Nano Banana 2 | `google/nano-banana-2` | Gemini 3.1 Flash, fast and high quality (default) |
| Nano Banana Pro | `google/nano-banana-pro` | Google Gemini 3 Pro, highest quality |
| Nano Banana | `google/nano-banana` | Original Gemini 2.5 Flash Image |
| Flux Schnell | `black-forest-labs/flux-schnell` | Fast, good quality |
| Flux Dev | `black-forest-labs/flux-dev` | Higher quality, slower |

If the user asks for a specific model by name, use that model's Replicate identifier.

### 2. Create a Prediction

Submit the image generation request using the Replicate HTTP API:

```bash
curl -s -X POST "https://api.replicate.com/v1/models/{model_owner}/{model_name}/predictions" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "prompt": "the user prompt here",
      "aspect_ratio": "1:1",
      "resolution": "2K",
      "output_format": "jpg"
    }
  }'
```

The response contains an `id` field and a `urls.get` field for polling.

**Nano Banana 2 input parameters:**

| Parameter | Default | Options |
|-----------|---------|---------|
| prompt | (required) | Text description of the image |
| aspect_ratio | `1:1` | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`, `1:4`, `4:1`, `1:8`, `8:1` |
| resolution | `2K` | `512px`, `1K`, `2K`, `4K` |
| output_format | `jpg` | `jpg`, `png` |
| image_input | `[]` | Array of image URLs (up to 14) for editing/reference |

**Aspect ratio selection:**
- Default to `1:1` unless the user specifies otherwise
- "landscape" → `16:9`, "portrait" → `9:16`, "wide" → `21:9`
- **Fizzy card cover image** → `4:1` (ultra-wide). When the user mentions the image is for a "fizzy card", "card cover", or "fizzy cover image", always use `4:1` aspect ratio. Note: `8:1` causes timeouts on Nano Banana 2.

### 3. Poll for Completion

Poll the prediction status until it completes:

```bash
curl -s "https://api.replicate.com/v1/predictions/{prediction_id}" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN"
```

Check the `status` field:
- `starting` or `processing` → wait 2 seconds, poll again
- `succeeded` → extract the output URL
- `failed` or `canceled` → report the error from the `error` field

Use a polling loop with `sleep 2` between attempts. Maximum 60 seconds of polling.

### 4. Return the Result

Extract the image URL from the `output` field of the completed prediction. For Flux models, `output` is typically an array — return the first element. For some models, `output` is a single URL string.

Present the URL to the user. The URL is temporary and hosted by Replicate.

## Implementation Pattern

Combine all steps into a single bash execution:

```bash
# Create prediction
RESPONSE=$(curl -s -X POST "https://api.replicate.com/v1/models/google/nano-banana-2/predictions" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input": {"prompt": "USER_PROMPT", "aspect_ratio": "1:1", "resolution": "2K", "output_format": "jpg"}}')

PREDICTION_ID=$(echo "$RESPONSE" | jq -r '.id')

if [ -z "$PREDICTION_ID" ] || [ "$PREDICTION_ID" = "null" ]; then
  echo "Error creating prediction:"
  echo "$RESPONSE" | jq .
  exit 1
fi

# Poll for completion
for i in $(seq 1 30); do
  RESULT=$(curl -s "https://api.replicate.com/v1/predictions/$PREDICTION_ID" \
    -H "Authorization: Bearer $REPLICATE_API_TOKEN")
  STATUS=$(echo "$RESULT" | jq -r '.status')

  if [ "$STATUS" = "succeeded" ]; then
    echo "$RESULT" | jq -r '.output | if type == "array" then .[0] else . end'
    exit 0
  elif [ "$STATUS" = "failed" ] || [ "$STATUS" = "canceled" ]; then
    echo "Generation failed:"
    echo "$RESULT" | jq -r '.error // .logs'
    exit 1
  fi

  sleep 2
done

echo "Timed out waiting for image generation"
exit 1
```

## Important Notes

- Always use `jq` for JSON parsing. It is available in most environments.
- Properly escape the user's prompt in the JSON payload. Use `jq` to construct the JSON body if the prompt contains special characters:
  ```bash
  JSON_BODY=$(jq -n --arg prompt "$USER_PROMPT" '{"input": {"prompt": $prompt, "aspect_ratio": "1:1", "resolution": "2K", "output_format": "jpg"}}')
  ```
- Do not hardcode the API token — always reference `$REPLICATE_API_TOKEN`.
- If the user wants to save the image locally, download it with `curl -o output.png "$IMAGE_URL"`.
- For multiple images, run the workflow multiple times or adjust model-specific parameters if supported.
