# Replicate Images Plugin for Claude Code

Generate images from text prompts using the [Replicate API](https://replicate.com), directly from Claude Code conversations.

## Installation

```bash
claude plugin marketplace add robzolkos/replicate-images
claude plugin install replicate-images
```

## Prerequisites

Set your Replicate API token as an environment variable:

```bash
export REPLICATE_API_TOKEN="r8_..."
```

Get a token at [replicate.com/account/api-tokens](https://replicate.com/account/api-tokens).

## Usage

Just ask Claude to generate an image in natural language:

- "Generate an image of a mountain sunset"
- "Create a picture of a cat wearing a top hat"
- "Make me a cover image for a fizzy card of an aquarium"

The skill triggers automatically and returns a Replicate-hosted URL.

### Models

The default model is [Nano Banana 2](https://replicate.com/google/nano-banana-2) (Google Gemini 3.1 Flash). You can request a different model by name:

| Model | Notes |
|-------|-------|
| Nano Banana 2 | Fast, high quality (default) |
| Nano Banana Pro | Highest quality, slower |
| Flux Schnell | Fast alternative |
| Flux Dev | Higher quality, slower |

### Aspect Ratios

The plugin picks an aspect ratio based on context:

| Request | Aspect Ratio |
|---------|-------------|
| Default | 1:1 |
| "landscape" | 16:9 |
| "portrait" | 9:16 |
| "wide" | 21:9 |
| Fizzy card cover | 4:1 |

## License

MIT
