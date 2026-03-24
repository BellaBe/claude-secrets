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

**Always quote values** — tokens contain colons and special characters that the shell will split or truncate without quotes:

```bash
cd ~/claude-secrets

# ✓ Correct — quoted
./secret.sh set telegram bot-token "123456789:AAHfiqksKZ8..."
./secret.sh set telegram admin-id "987654321"

# ✗ Wrong — token gets truncated at the colon
./secret.sh set telegram bot-token 123456789:AAHfiqksKZ8...
```

More examples:

```bash
./secret.sh set slack bot-token "xoxb-..."
./secret.sh set slack signing-secret "abc123..."
./secret.sh set discord bot-token "MTk..."
```

### Verify after storing

Always verify the token works after storing:

```bash
# Telegram
curl "https://api.telegram.org/bot$(pass show claude-telegram/bot-token)/getMe"

# Discord
curl -H "Authorization: Bot $(pass show claude-discord/bot-token)" https://discord.com/api/v10/users/@me
```

If you get a 404 or 401, the token is wrong — re-store it with quotes.

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

### GPG passphrase and subprocesses

Claude Code spawns the start script as a subprocess with no TTY. If GPG needs your passphrase and the cache is empty, it can't prompt you — the start script fails silently.

**Prime the cache before starting Claude Code:**

```bash
pass show claude-telegram/bot-token > /dev/null && claude --dangerously-load-development-channels server:telegram
```

Or create a one-liner alias:

```bash
alias claude-tg='pass show claude-telegram/bot-token > /dev/null && claude --dangerously-load-development-channels server:telegram'
```

### Lock down GPG cache timeout

By default GPG remembers your passphrase for a long time. Shorten it so the decryption window closes before Claude's session is fully running:

```bash
echo "default-cache-ttl 60
max-cache-ttl 300" > ~/.gnupg/gpg-agent.conf

gpg-connect-agent reloadagent /bye
```

Passphrase expires 60 seconds after you type it. The start script decrypts at launch, the cache clears, and Claude can't decrypt anything even if it tries.

## Security model

| Layer | Claude can access? |
|---|---|
| `~/.claude/channels/*/.env` | **Yes** — flat file |
| `~/.password-store/*.gpg` | **No** — GPG-encrypted |
| `process.env` at runtime | **No** — in-memory only |
| `/proc/<pid>/environ` | **Technically** — but requires finding the PID and parsing binary format |

The token never hits disk in plain text. Flow: encrypted file → pipe → shell memory → server memory.

## Troubleshooting

**"Secrets not found" when Claude Code starts**
GPG cache expired. Prime it: `pass show claude-telegram/bot-token > /dev/null` then reconnect.

**Token stored but API returns 404**
Token was truncated — you likely stored it without quotes. Re-store with quotes around the value.

**`pass insert` succeeds but `pass show` is empty**
Old version of the script without `-e` flag. Update `secret.sh` and re-store.

**Migration ran but no secrets stored**
Same `-e` flag issue. Use `secret.sh set` to store manually.

## Directory

```
~/claude-secrets/
  secret.sh     # this tool
  README.md     # this file
```

Separate from `~/.claude/` — that's Claude's directory, this is yours.