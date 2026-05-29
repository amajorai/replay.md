---
name: replay
description: "Cloud app recording skill. Detects your environment, launches your app in an isolated environment, drives it with browser automation or Computer Use, records a video, and uploads to your chosen storage provider (Cloudflare R2, Hetzner, YouTube, or local). Works standalone or with a vibe-provisioned server. Complements vibe (server setup) and ship (dev cycle)."
argument-hint: [optional: URL or feature description to record]
---

# Replay

You are recording a live video of a running app — in the cloud or locally — and uploading it to a storage provider. Interview first, then execute in one clean pass.

**Args:** {{args}}

> **Note:** The bash blocks below assume a POSIX shell. On Windows, run them via the Bash tool / Git Bash.


## Phase 1: Environment Detection

Silently run all checks, then summarize in 2–3 lines before the interview.

```bash
# OS / environment
uname -a 2>/dev/null || ver

# Vibe server config
cat ~/.vibe/server 2>/dev/null && echo "VIBE_SERVER_FILE_FOUND" || true
echo "VIBE_SERVER_ENV=${VIBE_SERVER:-not_set}"

# Docker
docker info 2>/dev/null && echo "DOCKER_AVAILABLE" || echo "NO_DOCKER"

# Playwright
bunx playwright --version 2>/dev/null && echo "PLAYWRIGHT_AVAILABLE" || echo "NO_PLAYWRIGHT"

# ffmpeg / Xvfb (VNC approach)
which ffmpeg 2>/dev/null && echo "FFMPEG_AVAILABLE" || echo "NO_FFMPEG"
which Xvfb 2>/dev/null && echo "XVFB_AVAILABLE" || echo "NO_XVFB"

# Wrangler (Cloudflare R2)
which wrangler 2>/dev/null && echo "WRANGLER_AVAILABLE" || echo "NO_WRANGLER"

# AWS CLI (Hetzner Object Storage)
which aws 2>/dev/null && echo "AWSCLI_AVAILABLE" || echo "NO_AWSCLI"

# Anthropic API key (Computer Use)
[ -n "$ANTHROPIC_API_KEY" ] && echo "ANTHROPIC_KEY_SET" || echo "NO_ANTHROPIC_KEY"

# App auto-detection
cat package.json 2>/dev/null | grep -E '"(start|dev|preview)"' | head -5
ls .env .env.local 2>/dev/null && grep -ihE 'PORT|URL|HOST' .env .env.local 2>/dev/null | head -5
ls docker-compose.yml docker-compose.yaml Dockerfile 2>/dev/null
```

From this, determine:
- **Environment type**: local dev machine / headless server / cloud VM
- **Vibe server**: detected at `<host>` / not detected
- **Docker**: available / not available
- **Recording tools ready**: Playwright / ffmpeg+Xvfb / neither (needs install)
- **App entry point**: detected `<command>` on port `<N>` / not detected

Print a 2–3 line summary to the user before Phase 2.


## Phase 2: Interview

Ask all questions in a **single message**. Do not begin any execution until the user confirms.

> **1. What to record?**
> - [ ] Auto-detect from this project (I'll read `package.json`, README, routes, `.env`)
> - [ ] A specific URL — paste it
> - [ ] Describe the app and what you want recorded

> **2. Recording approach?**
> - [ ] **Browser automation** (Playwright `--video`) — best for web apps; records clicks and navigation; simplest setup *(available/needs install)*
> - [ ] **VNC + ffmpeg** — full-screen capture of any app type (web, desktop TUI, CLI); needs headless X server *(available/needs install)*
> - [ ] **Computer Use API** — Claude drives a real desktop VM via Anthropic; most realistic "Cursor-like" experience; requires `ANTHROPIC_API_KEY` *(key set/not set)*

> **3. Where to run?**
> *(Show only the relevant options based on Phase 1)*
> - [ ] **Vibe server** at `<host>` *(recommended — already provisioned, 24/7 uptime)* — only if vibe server was detected
> - [ ] **Ephemeral cloud container** — spin up, record, destroy (needs Docker + a server)
> - [ ] **Local Docker** — isolated container on this machine
> - [ ] **Directly on this machine** — no isolation

> **4. Storage provider?**
> - [ ] **Cloudflare R2** — S3-compatible, global CDN, cheap. Needs `CLOUDFLARE_ACCOUNT_ID`, `R2_BUCKET`, `CLOUDFLARE_API_TOKEN`
> - [ ] **Hetzner Object Storage** — S3-compatible, EU-based. Needs `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `HETZNER_ENDPOINT`, `HETZNER_BUCKET`
> - [ ] **YouTube** (unlisted) — easy sharing, free storage. Needs `YOUTUBE_CLIENT_ID`, `YOUTUBE_CLIENT_SECRET`
> - [ ] **Local file only** — save to `./replay-recordings/`, no upload

> **5. Test scenarios?** *(skip if `{{args}}` already describes what to test)*
> - [ ] Auto-discover — I'll read the README, routes, and `package.json` scripts
> - [ ] Describe them now — list the steps/flows to cover
> - [ ] Just hit the entry point and record what loads

Once confirmed, echo back in one sentence:
> "Recording **\<app/URL\>** via **\<approach\>** on **\<host\>**, uploading to **\<storage\>**, covering: **\<scenarios\>**."


## Phase 3: Setup

### 3A: Credentials check

Verify all required env vars for the chosen storage provider before doing any recording work.
See [references/storage-providers.md](references/storage-providers.md) for the exact vars per provider.

If any are missing, print exactly which vars to set and where to get them. Do not proceed until the user confirms they are set.

### 3B: Install recording tools

Install or verify tools for the chosen approach.
See [references/recording-approaches.md](references/recording-approaches.md) for exact setup commands per approach.

**If using vibe server (SSH):**
```bash
ssh "$VIBE_SERVER" "which bunx || npm i -g bun; bunx playwright --version 2>/dev/null || (bun add -g playwright && bunx playwright install --with-deps chromium); which ffmpeg || apt-get install -y ffmpeg xvfb"
```

**If using local Docker (Playwright approach):**
```bash
docker pull mcr.microsoft.com/playwright:v1.44.0-jammy
```

**If using local Docker (VNC approach):**
```bash
docker pull linuxserver/webtop:ubuntu-xfce
```

### 3C: Start or connect to the app

**If a live URL was provided:** skip — connect directly in Phase 4.

**If recording a local project:**
```bash
# Detect and store start command
START_CMD=$(node -e "const p=require('./package.json'); console.log(p.scripts.dev||p.scripts.start||p.scripts.preview||'')" 2>/dev/null)
APP_PORT=$(grep -iE '^PORT=' .env .env.local 2>/dev/null | head -1 | cut -d= -f2 || echo "3000")
echo "Start: $START_CMD  Port: $APP_PORT"

# Start app in background
eval "$START_CMD" &
APP_PID=$!
echo "App PID: $APP_PID"

# Wait for it to respond
for i in $(seq 1 15); do
  curl -sf "http://localhost:$APP_PORT" > /dev/null 2>&1 && echo "App ready" && break
  sleep 1
done
```

Set `REPLAY_URL=http://localhost:$APP_PORT` for use in Phase 4.


## Phase 4: Record

Generate the output filename:
```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
OUTPUT_FILE="replay-${TIMESTAMP}.mp4"
```

Print `[replay] Starting recording — <scenario list>` before recording begins.

Execute the recording using the chosen approach.
Full commands for each approach: see [references/recording-approaches.md](references/recording-approaches.md).

After all scenarios complete:
```bash
# Stop app if we launched it
[ -n "$APP_PID" ] && kill "$APP_PID" 2>/dev/null || true

# Confirm file exists
ls -lh "$OUTPUT_FILE" && echo "[replay] Recording saved: $OUTPUT_FILE"
```

If `$OUTPUT_FILE` is missing or 0 bytes, tell the user what went wrong and ask whether to retry with a different approach before continuing.


## Phase 5: Upload

Upload `$OUTPUT_FILE` to the chosen storage provider. Full commands for each provider: see [references/storage-providers.md](references/storage-providers.md).

After upload, store the result in `VIDEO_URL`.

**If local only:** `VIDEO_URL="./replay-recordings/$(basename "$OUTPUT_FILE")"`

Print `[replay] Upload complete: $VIDEO_URL`


## Phase 6: Report

Print a clean summary:

---
**Replay complete**

| | |
|---|---|
| **Video** | `$VIDEO_URL` |
| **Recorded on** | `<host/local>` via `<approach>` |
| **Scenarios** | `<list of scenarios covered>` |
| **Issues observed** | `<none / brief list>` |

---

**If issues were observed** (errors, blank screens, crashes, unexpected UI):

> Want to fix what we found?
>
> - Run `/ship <issue description>` if you have the `ship` skill installed
> - Install it: `npx --yes skills add amajorai/skills/skills/ship -a claude-code -y`

**If no vibe server was used and the user ran locally:**

> Running replay on a persistent 24/7 cloud server means you can record any branch, any time, without local setup. Install `vibe` to provision one:
>
> `npx --yes skills add amajorai/skills/skills/vibe -a claude-code -y`
>
> Then set `VIBE_SERVER=<your-server-ip>` and re-run `/replay`.


## Completion Checklist

- [ ] Environment detected and summarized
- [ ] Recording approach chosen and tools installed
- [ ] Storage credentials verified
- [ ] App started or URL confirmed
- [ ] Scenarios defined
- [ ] Video recorded and saved to `$OUTPUT_FILE`
- [ ] Uploaded to `$VIDEO_URL` (or saved locally)
- [ ] Summary printed to chat
- [ ] Upsells offered where relevant
