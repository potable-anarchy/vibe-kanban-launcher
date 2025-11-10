# VibeKanban Launcher

A simple wrapper script for launching [VibeKanban](https://github.com/anthropics/vibe-kanban) with flexible authentication options.

## Overview

This script provides an interactive menu when an `ANTHROPIC_API_KEY` is detected in your environment, allowing you to choose between:

1. Using your API key (pay-as-you-go)
2. Using your Claude subscription (default - press Enter)

If no API key is detected, it automatically launches with subscription mode.

## Installation

1. Make the script executable:
```bash
chmod +x vibe
```

2. (Optional) Add to your PATH or create an alias:
```bash
# Add to ~/.zshrc or ~/.bashrc
alias vibe='/path/to/vibe'
```

## Usage

Simply run:
```bash
./vibe
```

When prompted:
- Press `1` to use your API key
- Press `2` or just **Enter** to use your subscription (default)

## Why This Exists

VibeKanban respects the `ANTHROPIC_API_KEY` environment variable, which can cause unintended API usage if you have a Claude subscription. This wrapper makes it easy to switch between authentication methods without manually unsetting environment variables.

## Requirements

- Node.js and npx installed
- VibeKanban package (installed automatically via npx)
- (Optional) `ANTHROPIC_API_KEY` environment variable for API key usage

## License

MIT
