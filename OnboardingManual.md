# OpenClaw Onboarding Manual

> **Purpose**: This document details the OpenClaw onboarding process, both interactive and non-interactive, so that any AI or developer working on automating the onboarding via GitHub Actions can understand every step, prompt, and config outcome.

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Onboarding Modes](#onboarding-modes)
4. [Interactive Onboarding Flow (Step by Step)](#interactive-onboarding-flow)
5. [Non-Interactive Onboarding](#non-interactive-onboarding)
6. [Config File Structure](#config-file-structure)
7. [CLIProxyAPI Integration](#cliproxyapi-integration)
8. [Encryption Scheme (from Strix/Qwen.yml)](#encryption-scheme)
9. [OpenClaw.yml Workflow Automation](#openclawml-workflow-automation)
10. [Key Source Files Reference](#key-source-files-reference)

---

## Overview

OpenClaw is an AI agent platform that connects to various chat channels (Discord, Telegram, Slack, etc.) and uses LLM providers to respond. The onboarding wizard configures:

1. **LLM Provider** (Anthropic, OpenAI, Gemini, custom API, etc.)
2. **Gateway** (local WebSocket server for the agent)
3. **Chat Channels** (Discord, Telegram, etc.)
4. **Skills** (plugins like clawhub, goplaces, etc.)
5. **Hooks** (automation on agent commands)
6. **Daemon Service** (systemd/launchd service to keep gateway running)

## Installation

```bash
npm install -g openclaw@latest
```

**Requirements**: Node.js >= 22.12.0

After installation, the CLI is available as `openclaw`.

## Onboarding Modes

### 1. Interactive Mode (default)
```bash
openclaw onboard --install-daemon
```

### 2. Non-Interactive Mode
```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice custom-api-key \
  --custom-base-url "http://127.0.0.1:8317/v1" \
  --custom-api-key "dummykey" \
  --custom-model-id "qwen3-coder-plus" \
  --custom-compatibility openai \
  --install-daemon \
  --skip-channels \
  --skip-skills
```

### 3. Flow Options
- `--flow quickstart` - Quick setup with defaults (loopback, token auth, auto daemon install)
- `--flow advanced` (or `--flow manual`) - Full manual configuration

---

## Interactive Onboarding Flow

Below is the exact sequence of prompts you will encounter during interactive onboarding. This was studied from the source code in `src/wizard/onboarding.ts` and related files.

### Step 1: Security Warning & Risk Acknowledgement
```
Security warning -- please read.
...
I understand this is powerful and inherently risky. Continue? [y/N]
```
- **Answer**: Yes (or use `--accept-risk` flag)

### Step 2: Onboarding Mode Selection
```
Onboarding mode
> QuickStart  (Configure details later via openclaw configure.)
  Manual      (Configure port, network, Tailscale, and auth options.)
```
- **QuickStart** skips gateway configuration details and goes straight to provider + channels
- **Manual** lets you configure gateway port, bind, auth, tailscale

### Step 3: Existing Config Detection (if config exists)
```
Existing config detected
Config handling
> Use existing values
  Update values
  Reset
```

### Step 4: LLM Provider Selection (Auth Choice)
The prompt groups providers into categories. The key ones:
- **Anthropic** (OAuth, API key, setup-token)
- **OpenAI** (API key, Codex OAuth)
- **Google** (Gemini API key, Gemini CLI, Antigravity)
- **Custom** (`custom-api-key`) - This is what we use for CLIProxyAPI

When choosing **Custom Provider** (`custom-api-key`):

#### Step 4a: API Base URL
```
API Base URL
> http://127.0.0.1:11434/v1
```
- Enter the CLIProxyAPI endpoint (e.g., `http://127.0.0.1:8317/v1`)

#### Step 4b: API Key
```
API Key (leave blank if not required)
> sk-...
```
- Enter any dummy key (e.g., `dummykey`) since CLIProxyAPI handles auth

#### Step 4c: Endpoint Compatibility
```
Endpoint compatibility
> OpenAI-compatible      (Uses /chat/completions)
  Anthropic-compatible   (Uses /messages)
  Unknown (detect automatically)  (Probes OpenAI then Anthropic endpoints)
```
- Choose **Unknown (detect automatically)** - this probes the endpoint
- If detection fails, it asks you to retry or change URL/model

#### Step 4d: Model ID
```
Model ID
> e.g. llama3, claude-3-7-sonnet
```
- Enter `qwen3-coder-plus`

#### Step 4e: Endpoint ID
```
Endpoint ID
> custom-127-0-0-1-8317
```
- Auto-derived from URL, can be customized

#### Step 4f: Model Alias (optional)
```
Model alias (optional)
> e.g. local, ollama
```
- Can leave empty

### Step 5: Gateway Configuration (Manual mode only)
In QuickStart mode, defaults are:
- Port: 18789
- Bind: Loopback (127.0.0.1)
- Auth: Token (auto-generated)
- Tailscale: Off

### Step 6: Channel Selection

#### QuickStart Mode:
```
Select channel (QuickStart)
> Discord        (needs token / configured)
  Telegram       (needs token)
  Slack          (needs token)
  WhatsApp       (needs setup)
  Signal         (needs setup)
  iMessage       (macOS only)
  Skip for now
```

#### When Discord is selected:
```
Discord bot token
1) Discord Developer Portal -> Applications -> New Application
2) Bot -> Add Bot -> Reset Token -> copy token
3) OAuth2 -> URL Generator -> scope 'bot' -> invite to your server
...

Enter Discord bot token
> [your bot token]
```

Then:
```
Configure Discord channels access?
> Yes / No
```

If Yes:
```
Discord channels access
> Allowlist (recommended)
  Open (allow all channels)
  Disabled (block all channels)
```

### Step 7: Skills Configuration
```
Skills status
Eligible: X
Missing requirements: Y
Unsupported on this OS: Z
Blocked by allowlist: W

Configure skills now? (recommended)
> Yes / No
```

If Yes:
```
Install missing skill dependencies
> [ ] Skip for now
  [x] clawhub
  [ ] other-skill...
```

Then:
```
Preferred node manager for skill installs
> npm
  pnpm
  bun
```

After installation, for each skill with missing env vars:
```
Set GOOGLE_PLACES_API_KEY for goplaces? [y/N]
Set GEMINI_API_KEY for nano-banana-pro? [y/N]
Set NOTION_API_KEY for notion? [y/N]
Set OPENAI_API_KEY for openai-image-gen? [y/N]
Set OPENAI_API_KEY for openai-whisper-api? [y/N]
Set ELEVENLABS_API_KEY for sag? [y/N]
```

### Step 8: Hooks Configuration
```
Hooks
Hooks let you automate actions when agent commands are issued.
Example: Save session context to memory when you issue /new or /reset.

Enable hooks?
> [ ] Skip for now
  [ ] hook-name-1
  [ ] hook-name-2
```

### Step 9: Gateway Service Installation
```
Install Gateway service (recommended) [Y/n]
```
In QuickStart: auto-installed

### Step 10: Health Check
Automatic health check of the gateway.

### Step 11: Control UI / TUI
```
How do you want to hatch your bot?
> Hatch in TUI (recommended)
  Open the Web UI
  Do this later
```

### Step 12: Completion
```
Onboarding complete. Use the dashboard link above to control OpenClaw.
```

---

## Non-Interactive Onboarding

The non-interactive mode (`--non-interactive --accept-risk`) supports:

### Supported auth choices:
- `apiKey` (Anthropic API key)
- `token` (Anthropic setup token)
- `gemini-api-key`
- `openai-api-key`
- `openrouter-api-key`
- `custom-api-key` (requires `--custom-base-url` and `--custom-model-id`)
- Many others (see `src/commands/onboard-types.ts`)

### Full non-interactive example:
```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice custom-api-key \
  --custom-base-url "http://127.0.0.1:8317/v1" \
  --custom-api-key "dummykey" \
  --custom-model-id "qwen3-coder-plus" \
  --custom-compatibility openai \
  --install-daemon \
  --skip-channels \
  --skip-skills \
  --skip-health \
  --skip-ui
```

### What non-interactive does NOT handle:
- Channel configuration (Discord, Telegram, etc.)
- Skills installation
- Hooks setup

These must be configured by directly editing `~/.openclaw/openclaw.json` after non-interactive onboarding.

---

## Config File Structure

The config lives at `~/.openclaw/openclaw.json`. Key sections:

```json
{
  "models": {
    "mode": "merge",
    "default": "custom-127-0-0-1-8317/qwen3-coder-plus",
    "providers": {
      "custom-127-0-0-1-8317": {
        "baseUrl": "http://127.0.0.1:8317/v1",
        "api": "openai-completions",
        "apiKey": "dummykey",
        "models": [
          {
            "id": "qwen3-coder-plus",
            "name": "qwen3-coder-plus (Custom Provider)",
            "contextWindow": 4096,
            "maxTokens": 4096,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "reasoning": false
          }
        ]
      }
    }
  },
  "channels": {
    "discord": {
      "enabled": true,
      "token": "YOUR_DISCORD_BOT_TOKEN",
      "groupPolicy": "open",
      "guilds": {}
    }
  },
  "skills": {
    "install": {
      "nodeManager": "npm"
    },
    "entries": {}
  },
  "hooks": {
    "internal": {
      "enabled": false,
      "entries": {}
    }
  },
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "models": {}
    }
  },
  "gateway": {
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "auto-generated-token"
    }
  },
  "_wizard": {
    "command": "onboard",
    "mode": "local"
  }
}
```

### Discord Configuration Details:
- `channels.discord.enabled`: true/false
- `channels.discord.token`: Bot token string
- `channels.discord.groupPolicy`: "open" | "allowlist" | "disabled"
  - "open" = allow all channels
  - "allowlist" = only specified guilds/channels
- `channels.discord.guilds`: Guild/channel allowlist entries

---

## CLIProxyAPI Integration

### Building from Source
```bash
git clone --depth 1 https://github.com/router-for-me/CLIProxyAPI.git /tmp/CLIProxyAPI
cd /tmp/CLIProxyAPI
go build -o cli-proxy-api ./cmd/server
sudo mv cli-proxy-api /usr/local/bin/cliproxyapi
```

### Config for CLIProxyAPI (`config.yaml`)
```yaml
port: 8317
auth-dir: "/path/to/auth/dir"
debug: true
logging-to-file: true
api-keys:
  - "dummykey"
routing:
  strategy: "round-robin"
```

### Starting the proxy server
```bash
cliproxyapi -config /path/to/config.yaml &
```

It will listen on `http://127.0.0.1:8317` and provide:
- `POST /v1/chat/completions` (OpenAI-compatible)
- `GET /v1/models` (model listing)

### Qwen Token Files
Qwen auth tokens are stored as JSON files in the auth-dir:
- Pattern: `qwen-*.json`
- Created by `cliproxyapi -qwen-login` or by decrypting pre-collected tokens

### Round-Robin
CLIProxyAPI automatically round-robins between all Qwen token files found in the auth directory.

---

## Encryption Scheme

The Strix/Qwen.yml workflow uses the following encryption for token collection:

### Encryption (done by Qwen.yml workflow):
1. Qwen tokens are collected as JSON files (`qwen-*.json`)
2. Files are tar'd together: `tar -czf qwen-tokens.tar.gz qwen-*.json`
3. Encrypted with AES-256-CBC: 
   ```bash
   openssl enc -aes-256-cbc -salt -pbkdf2 \
     -in qwen-tokens.tar.gz \
     -out qwen-tokens.enc \
     -pass pass:'<user_password>'
   ```
4. Base64 encoded: `base64 -w 0 qwen-tokens.enc`
5. Stored as the `QWEN_TOKENS` GitHub secret (one long base64 string)

### Decryption (done by our OpenClaw.yml workflow):
```bash
# 1. Decode base64
echo "$ENCRYPTED_TOKENS" | base64 -d > qwen-tokens.enc

# 2. Decrypt with AES-256-CBC
openssl enc -aes-256-cbc -d -salt -pbkdf2 \
  -in qwen-tokens.enc \
  -out qwen-tokens.tar.gz \
  -pass pass:"$DECRYPTION_PASSWORD"

# 3. Extract token files
tar -xzf qwen-tokens.tar.gz
```

The decrypted `qwen-*.json` files go into CLIProxyAPI's auth directory.

---

## OpenClaw.yml Workflow Automation

### Workflow Inputs (workflow_dispatch):
| Input | Description | Required |
|-------|-------------|----------|
| `encrypted_tokens` | Base64-encoded AES-256-CBC encrypted Qwen tokens | Yes |
| `decryption_password` | Password used to encrypt the tokens | Yes |
| `discord_bot_token` | Discord bot token for channel setup | Yes |

### Automation Strategy:
1. **Install Node.js >= 22** (required by OpenClaw)
2. **Install Go** (required to build CLIProxyAPI from source)
3. **Install OpenClaw** globally via npm
4. **Build CLIProxyAPI** from source
5. **Decrypt tokens** and place in CLIProxyAPI auth directory
6. **Create CLIProxyAPI config** with round-robin strategy
7. **Start CLIProxyAPI** in background
8. **Run OpenClaw non-interactive onboarding** with custom-api-key pointing to CLIProxyAPI
9. **Patch config** for Discord, skills, and hooks (since non-interactive skips these)
10. **Install daemon** and start the gateway

### Key Decisions:
- **Model name**: `qwen3-coder-plus` (confirmed from CLIProxyAPI config.example.yaml)
- **API endpoint**: `http://127.0.0.1:8317/v1` (CLIProxyAPI default port)
- **Compatibility**: `openai` (CLIProxyAPI provides OpenAI-compatible endpoints)
- **API key**: `dummykey` (CLIProxyAPI uses its own auth; the key just needs to match its config)

---

## Key Source Files Reference

| File | Purpose |
|------|---------|
| `src/wizard/onboarding.ts` | Main interactive onboarding wizard |
| `src/commands/onboard.ts` | Entry point (routes to interactive or non-interactive) |
| `src/commands/onboard-non-interactive.ts` | Non-interactive dispatcher |
| `src/commands/onboard-non-interactive/local.ts` | Non-interactive local onboarding |
| `src/commands/onboard-non-interactive/local/auth-choice.ts` | Non-interactive auth choice handler |
| `src/commands/onboard-custom.ts` | Custom API provider configuration |
| `src/commands/onboard-channels.ts` | Channel setup wizard |
| `src/commands/onboard-skills.ts` | Skills setup wizard |
| `src/commands/onboard-hooks.ts` | Hooks setup wizard |
| `src/channels/plugins/onboarding/discord.ts` | Discord-specific onboarding |
| `src/channels/plugins/onboarding/channel-access.ts` | Channel access policy prompts |
| `src/wizard/onboarding.finalize.ts` | Finalization (daemon, health, TUI) |
| `src/cli/program/register.onboard.ts` | CLI flag definitions |
| `src/commands/onboard-types.ts` | Type definitions for all onboard options |
| `src/config/config.ts` | Config file read/write (~/.openclaw/openclaw.json) |

---

## Notes for Next AI Working on This

1. **Non-interactive mode is the key**. Use `--non-interactive --accept-risk` with `--auth-choice custom-api-key` to skip all prompts for the core LLM setup.

2. **Channels/Skills/Hooks must be patched manually** after non-interactive onboarding by editing `~/.openclaw/openclaw.json` with `jq` or similar.

3. **CLIProxyAPI must be running BEFORE onboarding** if you want endpoint detection to work. Otherwise use `--custom-compatibility openai` to skip detection.

4. **The encryption is standard AES-256-CBC with PBKDF2** via openssl. The password is user-provided.

5. **The Qwen model name is `qwen3-coder-plus`** (confirmed from CLIProxyAPI config.example.yaml oauth-model-alias section).

6. **Discord "Open (allow all channels)"** maps to `groupPolicy: "open"` in config JSON.

7. **Skills "clawhub"** is the main installable skill dependency. Install via `openclaw` post-onboarding or patch config.

8. **The gateway port default is 18789**. CLIProxyAPI uses port 8317.

9. **Config file location**: `~/.openclaw/openclaw.json`

10. **Workspace location**: `~/.openclaw/workspace/`
