# llm-srt-translate

Public releases for the llm-srt-translate CLI tool.

This repository only hosts compiled releases. The README is synced from the
source repository on each release.

## Download

Grab `llm-srt-translate.phar` from the [releases page](https://github.com/iceman1010/llm-srt-translate-release/releases).

```bash
chmod +x llm-srt-translate.phar
sudo mv llm-srt-translate.phar /usr/local/bin/llm-srt-translate
```

## Requirements

- PHP 8.1 or later
- An OpenAI-compatible API endpoint (e.g. a LiteLLM proxy)

Configure with `llm-srt-translate --setup-api`.
