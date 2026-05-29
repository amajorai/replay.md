# Recording Approaches

Reference for Phase 3B (setup) and Phase 4 (record) of the `replay` skill.

---

## Option A: Browser Automation (Playwright)

Best for web apps. Records video of every page interaction. Headless — no display server needed.

### Setup

```bash
# Install Playwright + Chromium (local or on vibe server)
bun add -D playwright
bunx playwright install --with-deps chromium
```

### Record

Write a temporary script and run it:

```bash
cat > /tmp/replay-playwright.mjs << 'SCRIPT'
import { chromium } from 'playwright';

const url    = process.env.REPLAY_URL    || 'http://localhost:3000';
const output = process.env.REPLAY_OUTPUT || '/tmp/replay.webm';
const rawScenarios = process.env.REPLAY_SCENARIOS;

// scenarios: JSON array of steps, e.g.:
// [{"type":"goto","value":"http://localhost:3000/login"},
//  {"type":"fill","selector":"#email","value":"test@test.com"},
//  {"type":"click","selector":"button[type=submit]"},
//  {"type":"wait","ms":2000}]
const scenarios = rawScenarios ? JSON.parse(rawScenarios) : [];

(async () => {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({
    recordVideo: {
      dir: '/tmp/replay-videos/',
      size: { width: 1280, height: 720 }
    }
  });
  const page = await context.newPage();

  await page.goto(url);
  await page.waitForLoadState('networkidle').catch(() => {});
  console.log('[replay] Page loaded:', url);

  for (const step of scenarios) {
    console.log('[replay] Step:', step.type, step.selector || step.value || '');
    if (step.type === 'goto')       await page.goto(step.value);
    if (step.type === 'click')      await page.click(step.selector);
    if (step.type === 'fill')       await page.fill(step.selector, step.value);
    if (step.type === 'wait')       await page.waitForTimeout(step.ms || 1000);
    if (step.type === 'screenshot') await page.screenshot({ path: step.path });
    if (step.type === 'scroll')     await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
  }

  // Always wait a moment at the end so the last state is visible
  await page.waitForTimeout(2000);

  const videoPath = await page.video()?.path();
  await context.close();
  await browser.close();

  if (videoPath) {
    const { execSync } = await import('child_process');
    // Convert webm → mp4 if ffmpeg available, else keep webm
    try {
      execSync(`ffmpeg -i "${videoPath}" -c:v libx264 "${output}" -y 2>/dev/null`);
      execSync(`rm -f "${videoPath}"`);
    } catch {
      execSync(`cp "${videoPath}" "${output}"`);
    }
    console.log('[replay] Saved:', output);
  } else {
    console.error('[replay] No video captured');
    process.exit(1);
  }
})();
SCRIPT

REPLAY_URL="$REPLAY_URL" \
REPLAY_OUTPUT="$OUTPUT_FILE" \
REPLAY_SCENARIOS='[]' \
  node /tmp/replay-playwright.mjs
```

### On vibe server (SSH)

```bash
# Copy script and run remotely, then pull the file back
scp /tmp/replay-playwright.mjs "$VIBE_HOST:/tmp/"
ssh "$VIBE_HOST" "REPLAY_URL=$REPLAY_URL REPLAY_OUTPUT=/tmp/$OUTPUT_FILE node /tmp/replay-playwright.mjs"
scp "$VIBE_HOST:/tmp/$OUTPUT_FILE" "./$OUTPUT_FILE"
ssh "$VIBE_HOST" "rm /tmp/$OUTPUT_FILE"
```

---

## Option B: VNC + ffmpeg

Full-screen capture. Works for any app type: web, desktop, TUI, CLI. Requires a Linux environment with Xvfb.

### Setup

```bash
# Debian/Ubuntu
sudo apt-get update -qq
sudo apt-get install -y xvfb x11-utils ffmpeg chromium-browser xdotool

# Start virtual display
export DISPLAY=:99
Xvfb :99 -screen 0 1280x720x24 -ac +extension GLX +render -noreset &
XVFB_PID=$!
sleep 1
echo "[replay] Virtual display started (PID $XVFB_PID)"
```

### Record

```bash
# Start screen capture
ffmpeg -f x11grab -video_size 1280x720 -r 24 -i :99.0+0,0 \
  -vcodec libx264 -preset ultrafast -pix_fmt yuv420p \
  "$OUTPUT_FILE" -y &
FFMPEG_PID=$!
echo "[replay] ffmpeg recording started (PID $FFMPEG_PID)"

# Open browser
DISPLAY=:99 chromium-browser --no-sandbox --disable-gpu \
  --window-size=1280,720 --start-maximized \
  "$REPLAY_URL" &
BROWSER_PID=$!
sleep 4  # let the page render

# --- Run xdotool interactions (optional, build from scenarios) ---
# Example interactions:
# DISPLAY=:99 xdotool mousemove 640 360 click 1       # click center
# DISPLAY=:99 xdotool type --clearmodifiers "hello"   # type text
# DISPLAY=:99 xdotool key Return                       # press Enter
# DISPLAY=:99 xdotool key ctrl+l                       # address bar focus
# sleep 2                                              # wait between steps

# Record for N seconds (adjust to scenario length)
sleep 15

# Stop recording cleanly
kill "$FFMPEG_PID" 2>/dev/null
wait "$FFMPEG_PID" 2>/dev/null

# Cleanup
kill "$BROWSER_PID" "$XVFB_PID" 2>/dev/null
echo "[replay] VNC recording complete"
```

### Inside Docker (no local Xvfb needed)

```bash
docker run --rm \
  -e REPLAY_URL="$REPLAY_URL" \
  -v "$(pwd):/output" \
  --shm-size=2g \
  mcr.microsoft.com/playwright:v1.44.0-jammy \
  bash -c "
    apt-get install -y xvfb ffmpeg > /dev/null 2>&1
    Xvfb :99 -screen 0 1280x720x24 &
    export DISPLAY=:99
    ffmpeg -f x11grab -video_size 1280x720 -r 24 -i :99 \
      -vcodec libx264 -preset ultrafast /output/$OUTPUT_FILE -y &
    chromium-browser --no-sandbox --disable-gpu \$REPLAY_URL &
    sleep 15
    kill %2
    wait %2
  "
```

---

## Option C: Computer Use API

Claude drives a real desktop VM. Most realistic — Claude clicks, types, scrolls, and navigates like a real user. Requires `ANTHROPIC_API_KEY` and Docker.

### Setup

```bash
# Pull the Anthropic computer-use demo image
docker pull ghcr.io/anthropics/anthropic-quickstarts:computer-use-demo-latest

# Start the container (desktop accessible via VNC on :5900, web UI on :6080)
docker run -d \
  --name replay-cu \
  -e ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  -p 5900:5900 -p 8501:8501 -p 6080:6080 \
  --shm-size=2g \
  ghcr.io/anthropics/anthropic-quickstarts:computer-use-demo-latest

# Give it a moment to boot
sleep 5
echo "[replay] Computer Use container running"
```

### Record + drive with Claude

```bash
# Start recording the container's display via VNC
ffmpeg -f x11grab -video_size 1280x768 -r 15 -i :0 \
  -vcodec libx264 -preset ultrafast -pix_fmt yuv420p \
  "$OUTPUT_FILE" -y &
FFMPEG_PID=$!

# Send instructions to Claude via the Computer Use API
SCENARIO_TEXT="$1"  # pass scenario description as argument

curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: computer-use-2024-10-22" \
  -H "content-type: application/json" \
  -d "{
    \"model\": \"claude-opus-4-7\",
    \"max_tokens\": 4096,
    \"tools\": [{
      \"type\": \"computer_20241022\",
      \"name\": \"computer\",
      \"display_width_px\": 1280,
      \"display_height_px\": 768,
      \"display_number\": 1
    }],
    \"messages\": [{
      \"role\": \"user\",
      \"content\": \"Open a browser, navigate to $REPLAY_URL, and perform the following: $SCENARIO_TEXT. Move slowly and pause 1 second between each action so the recording is clear.\"
    }]
  }"

# Let Claude finish, then stop recording
sleep 5
kill "$FFMPEG_PID" 2>/dev/null
wait "$FFMPEG_PID" 2>/dev/null

# Teardown
docker rm -f replay-cu 2>/dev/null
echo "[replay] Computer Use recording complete"
```
