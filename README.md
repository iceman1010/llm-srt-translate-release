# llm-srt-translate

A PHP CLI tool that translates subtitle files (SRT, VTT, ASS, and more) into any language using LLMs — via **any OpenAI-compatible API**. Built for a [LiteLLM](https://docs.litellm.ai/) proxy setup: all providers, keys, and model aliases are managed centrally on the proxy; this tool just talks to one OpenAI-standard endpoint.

This is the successor to [glm-srt-translate](https://github.com/iceman1010/glm-srt-translate) (discontinued — GLM models cannot be forced into a strict output contract, which makes batch subtitle translation unreliable; see that repo's README for the post-mortem).

## Features

- **Any OpenAI-compatible endpoint** — point it at a LiteLLM proxy, OpenAI, or any compatible gateway
- **Any model alias the proxy exposes** — models with tuned settings live in `llm-models.json`; everything else works via passthrough defaults
- **Tiered structured output** — `json_schema` strict enforcement when the upstream supports it, `json_object` fallback, plain-text markers as last resort
- **Index-keyed splicing** — translations are matched by an index field, never by position, preventing text/timestamp drift
- **Smart batching** — sends subtitles in large batches; batch size auto-fits the model's context window
- **Parallel batch mode** — `--parallel=N` runs N batches concurrently with adaptive pacing (backs off on 429s, eases up on clean windows)
- **Checkpoint/resume** — parallel mode auto-saves checkpoints per batch; `--resume` recovers with zero wasted tokens
- **Automatic retry for partial batches** — missing subtitles from incomplete responses are retried immediately
- **Rate limit handling** — exponential backoff, honors `Retry-After` headers
- **RTL support** — automatic BiDi Unicode wrapping for Arabic, Hebrew, Persian, etc.
- **HTML tag preservation** — `<i>`, `<b>` tags survive translation
- **Subtitle format support** — SRT, VTT, ASS, and any format supported by [mantas-done/subtitles](https://github.com/mantas-done/subtitles)
- **PHAR build** — single-file executable, self-update via GitHub releases

## Requirements

- PHP >= 8.1
- ext-curl, ext-json, ext-mbstring
- An API key and base URL for an OpenAI-compatible endpoint (e.g. a LiteLLM proxy)

## Installation

### Download PHAR (recommended)

Grab the latest release from [GitHub Releases](https://github.com/iceman1010/llm-srt-translate/releases):

```bash
chmod +x llm-srt-translate.phar
sudo mv llm-srt-translate.phar /usr/local/bin/llm-srt-translate
```

### From Source

```bash
git clone https://github.com/iceman1010/llm-srt-translate.git
cd llm-srt-translate
composer install
```

## Setup

### Quick Setup (interactive)

```bash
php translate.php --setup-api
```

Prompts for your API key and base URL, saved to `~/.llm-srt-translate/.env`.

### Manual Setup

Create a `.env` file in the project directory (see `.env.example`):

```
LLM_API_KEY=your_api_key_here
LLM_API_BASE=https://your-litellm-proxy:4000/v1/chat/completions
```

## Usage

```bash
# Translate to German with the default model
php translate.php -i movie.srt -l de

# Explicit model alias (anything your proxy exposes)
php translate.php -i movie.srt -l German -m kimi-k2.6

# Parallel translation
php translate.php -i movie.srt -l de -m nemotron-ultra-550b --parallel=5

# See what the proxy offers
php translate.php --list-proxy-models
```

Languages can be given as full name (`German`), ISO 639-1 (`de`), or ISO 639-2 (`deu`). Output is written next to the input file (e.g. `movie.german.srt`).

### Options

| Option | Short | Description |
|--------|-------|-------------|
| `--input=<file>` | `-i` | Input subtitle file (required) |
| `--language=<lang>` | `-l` | Target language (required) |
| `--output=<file>` | `-o` | Output file path (default: auto-generated) |
| `--model=<alias>` | `-m` | Model alias exposed by the proxy (default: from `llm-models.json`) |
| `--batch-size=<n>` | `-b` | Override batch size |
| `--temperature=<f>` | `-t` | Sampling temperature 0.0–1.0 (default: 0.6) |
| `--max-tokens=<n>` | `-M` | Override max output tokens |
| `--description=<text>` | `-d` | Additional context for the translator |
| `--think` | `-r` | Enable reasoning mode on reasoning models |
| `--retry=<n>` | `-R` | Retry count for missing entries (default: 1) |
| `--format=<fmt>` | `-f` | Response format: `auto`, `json`, or `simple` (default: `auto` — picks the best tier the model supports) |
| `--delay=<secs>` | `-D` | Delay between batches in sequential mode (default: 60) |
| `--log=<file>` | `-L` | Log full request/response for debugging |
| `--parallel[=N]` | | Parallel batch mode: N concurrent requests (default: 3) |
| `--resume` | | Resume from checkpoints (requires `--parallel`) |
| `--restart` | | Start from beginning, ignore saved progress/checkpoints |
| `--log-issues` | | Dump batch diagnostics when entries are skipped/missing/duplicate |
| `--debug` | `-v` | Show system prompt and first user message |
| `--list-models` | | List models with local metadata |
| `--list-proxy-models` | | List model aliases the proxy currently exposes |
| `--setup-api` | | Configure key + base URL interactively |
| `--update[=version]` | | Self-update (PHAR only) |
| `--version` | | Show version |

## Response Formats

`--format=auto` (the default) picks the strongest output contract the model supports, in this order:

1. **`json_schema`** — token-level schema enforcement (`response_format: json_schema, strict`). The API guarantees one `{"index", "text"}` object per subtitle cue. Set `structured_output: "json_schema"` in `llm-models.json` for models/providers that support it.
2. **`json_object`** — valid JSON guaranteed, schema enforced via prompt. Works on most OpenAI-compatible providers but cannot prevent cue merging (the failure mode that killed the GLM-only predecessor).
3. **`simple`** — plain-text `[N]:` markers. Last resort for providers without JSON mode.

Translations are always spliced back by the returned `index` field — never by position.

## Model Registry

`llm-models.json` holds tuned settings for known models (batch size, context window, structured-output capability, notes). **Any model alias not listed there still works** — it gets sensible passthrough defaults. Run `--list-models` for the local registry, `--list-proxy-models` for what the proxy actually exposes.

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Rate limit (HTTP 429) | Exponential backoff (30s → 300s cap), honors `Retry-After`; parallel mode adapts concurrency. Not counted toward abort limit. |
| Quota/budget exhausted (429) | Fatal abort — top up or switch key. |
| Partial batch response | Missing subtitles retried immediately in a follow-up request. |
| Timeout | Retry after 10s. Counts toward 3-strike abort. |
| Server error (5xx) | Retry after 60s. Counts toward 3-strike abort. |
| Auth error (401/403) | Abort immediately. |
| 3 consecutive errors | Save progress and abort; resume with the same command. |

## Building from Source

```bash
composer install
./install_local.sh
```

Produces `llm-srt-translate.phar` in the project root.

## Architecture

```
llm-srt-translate/
├── bin/translate          # CLI entry point
├── src/
│   ├── LlmClient.php      # Generic OpenAI-compatible chat completions client
│   ├── PromptBuilder.php  # Translation prompts + strict JSON schema
│   └── Translator.php     # Orchestrator (batching, parallel, checkpoints, retry)
├── llm-models.json        # Model metadata + passthrough defaults
├── VERSION                # Current version (read by PHAR)
├── box.json               # PHAR build configuration
└── translate.php          # Convenience wrapper → bin/translate
```

## Credits

- Forked from [glm-srt-translate](https://github.com/iceman1010/glm-srt-translate), which was ported from [cf-llm-srt-translator](https://github.com/iceman1010/cf-llm-srt-translator)
- Subtitle parsing by [mantas-done/subtitles](https://github.com/mantas-done/subtitles)

## License

MIT
