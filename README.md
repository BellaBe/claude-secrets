# Claude Channel Secrets

GPG-encrypted credential storage for Claude Code MCP channels. Keeps tokens out of flat files where Claude's tools can read them.

## Why

MCP channel servers need API tokens. Storing them in `.env` files under `~/.claude/channels/` means Claude Code can read them via file tools. This moves credentials into `pass` (GPG-encrypted `~/.password-store/`), which Claude cannot access.

## Install

```bash
# Prerequisites (one-time)
sudo apt install pass gnupg
gpg --gen-key
pass init <gpg-key-id>          # gpg --list-keys to find it

# Place the directory
cp -r claude-secrets ~/claude-secrets
chmod +x ~/claude-secrets/secret.sh

# Optional alias
echo 'alias claude-secret="~/claude-secrets/secret.sh"' >> ~/.bashrc
source ~/.bashrc
```

## Usage

```
secret.sh set <channel> <key> <value>
secret.sh get <channel> <key>
secret.sh list [channel]
secret.sh rm  <channel> <key>
secret.sh migrate <channel> <ENV_KEY> <pass_key>
```

### Store

```bash
cd ~/claude-secrets

./secret.sh set telegram bot-token "123456789:AAH..."
./secret.sh set telegram admin-id "987654321"
./secret.sh set slack bot-token "xoxb-..."
./secret.sh set slack signing-secret "abc123..."
./secret.sh set discord bot-token "MTk..."
```

### Retrieve

```bash
./secret.sh get telegram bot-token
```

### List

```bash
./secret.sh list              # all channels
./secret.sh list telegram     # one channel
```

### Delete

```bash
./secret.sh rm slack signing-secret
```

### Migrate from .env

Reads a key from `~/.claude/channels/<channel>/.env`, stores it in `pass`, strips it from the file:

```bash
./secret.sh migrate telegram TELEGRAM_BOT_TOKEN bot-token
./secret.sh migrate telegram TELEGRAM_ADMIN_USER_ID admin-id
```

Override the channels directory if yours is elsewhere:

```bash
CLAUDE_CHANNELS_DIR=~/my-channels ./secret.sh migrate telegram TELEGRAM_BOT_TOKEN bot-token
```

## Storage layout

```
~/.password-store/
  claude-telegram/
    bot-token.gpg
    admin-id.gpg
  claude-slack/
    bot-token.gpg
    signing-secret.gpg
```

## Using in channel start scripts

Load secrets into env vars before launching the server:

```bash
export TELEGRAM_BOT_TOKEN="${TELEGRAM_BOT_TOKEN:-$(pass show claude-telegram/bot-token 2>/dev/null)}"
export TELEGRAM_ADMIN_USER_ID="${TELEGRAM_ADMIN_USER_ID:-$(pass show claude-telegram/admin-id 2>/dev/null)}"
exec bun run server.ts
```

The `${VAR:-$(...)}` pattern respects env vars already set by Docker, CI, or manual overrides.

## Security model

| Layer | Claude can access? |
|---|---|
| `~/.claude/channels/*/.env` | **Yes** — flat file |
| `~/.password-store/*.gpg` | **No** — GPG-encrypted |
| `process.env` at runtime | **No** — in-memory only |

## Directory

```
~/claude-secrets/
  secret.sh     # this tool
  README.md     # this file
```

Separate from `~/.claude/` — that's Claude's directory, this is yours.