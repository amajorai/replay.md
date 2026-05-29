# Storage Providers

Reference for Phase 3A (credentials check) and Phase 5 (upload) of the `replay` skill.

---

## Cloudflare R2

S3-compatible object storage with global CDN. Cheapest egress. Already available if you used the `vibe` skill with Wrangler.

### Required credentials

| Variable | Where to get it |
|---|---|
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare dashboard → right sidebar under your domain |
| `CLOUDFLARE_API_TOKEN` | Cloudflare dashboard → My Profile → API Tokens → Create Token (use the R2 template) |
| `R2_BUCKET` | Name of your R2 bucket (will be created if it doesn't exist) |

### Credentials check

```bash
[ -n "$CLOUDFLARE_ACCOUNT_ID" ] && echo "ACCOUNT_ID_SET" || echo "MISSING: CLOUDFLARE_ACCOUNT_ID"
[ -n "$CLOUDFLARE_API_TOKEN" ]  && echo "API_TOKEN_SET"  || echo "MISSING: CLOUDFLARE_API_TOKEN"
[ -n "$R2_BUCKET" ]             && echo "BUCKET_SET"     || echo "MISSING: R2_BUCKET"
which wrangler 2>/dev/null      && echo "WRANGLER_OK"    || echo "MISSING: wrangler (run: bun add -g wrangler)"
```

### Setup bucket

```bash
wrangler r2 bucket create "$R2_BUCKET" 2>/dev/null || echo "Bucket already exists — continuing"
```

### Upload

```bash
OBJECT_KEY="replay-recordings/$(basename "$OUTPUT_FILE")"
wrangler r2 object put "$R2_BUCKET/$OBJECT_KEY" --file "$OUTPUT_FILE"

# If the bucket has public access enabled via a custom domain or r2.dev:
VIDEO_URL="https://pub-<your-hash>.r2.dev/$OBJECT_KEY"

# Otherwise generate a pre-signed URL (1-hour expiry) — replace <account-id> with $CLOUDFLARE_ACCOUNT_ID:
VIDEO_URL=$(curl -s -X POST \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$R2_BUCKET/objects/$OBJECT_KEY/presign?expiresIn=3600" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | grep -o '"url":"[^"]*"' | cut -d'"' -f4)

echo "[replay] R2 URL: $VIDEO_URL"
```

---

## Hetzner Object Storage

S3-compatible, EU-based, low-cost. Good fit if your vibe server is on Hetzner.

### Required credentials

| Variable | Where to get it |
|---|---|
| `AWS_ACCESS_KEY_ID` | Hetzner Cloud console → Object Storage → your bucket → Access Keys → Create |
| `AWS_SECRET_ACCESS_KEY` | Same creation flow as above |
| `HETZNER_ENDPOINT` | e.g. `https://fsn1.your-objectstorage.com` (shown in Hetzner bucket settings) |
| `HETZNER_BUCKET` | Name of your Hetzner bucket |

### Credentials check

```bash
[ -n "$AWS_ACCESS_KEY_ID" ]     && echo "ACCESS_KEY_SET"  || echo "MISSING: AWS_ACCESS_KEY_ID"
[ -n "$AWS_SECRET_ACCESS_KEY" ] && echo "SECRET_KEY_SET"  || echo "MISSING: AWS_SECRET_ACCESS_KEY"
[ -n "$HETZNER_ENDPOINT" ]      && echo "ENDPOINT_SET"    || echo "MISSING: HETZNER_ENDPOINT"
[ -n "$HETZNER_BUCKET" ]        && echo "BUCKET_SET"      || echo "MISSING: HETZNER_BUCKET"
which aws 2>/dev/null           && echo "AWSCLI_OK"       || echo "MISSING: aws cli (run: bun x aws-cli or apt-get install awscli)"
```

### Setup

```bash
# Point AWS CLI at Hetzner endpoint
aws configure set default.s3.endpoint_url "$HETZNER_ENDPOINT"
```

### Upload

```bash
OBJECT_KEY="replay-recordings/$(basename "$OUTPUT_FILE")"
aws s3 cp "$OUTPUT_FILE" "s3://$HETZNER_BUCKET/$OBJECT_KEY" \
  --endpoint-url "$HETZNER_ENDPOINT"

# Pre-signed URL (1-hour expiry):
VIDEO_URL=$(aws s3 presign "s3://$HETZNER_BUCKET/$OBJECT_KEY" \
  --endpoint-url "$HETZNER_ENDPOINT" \
  --expires-in 3600)

echo "[replay] Hetzner URL: $VIDEO_URL"
```

---

## YouTube (Unlisted)

Free, easy to share, no storage cost. Video is unlisted — only people with the link can watch.

### Required credentials

| Variable | Where to get it |
|---|---|
| `YOUTUBE_CLIENT_ID` | [Google Cloud Console](https://console.cloud.google.com) → APIs & Services → Credentials → Create OAuth 2.0 Client ID (Desktop app) |
| `YOUTUBE_CLIENT_SECRET` | Same credential entry |

Enable the **YouTube Data API v3** in your Google Cloud project before uploading.

### Credentials check

```bash
[ -n "$YOUTUBE_CLIENT_ID" ]     && echo "CLIENT_ID_SET"     || echo "MISSING: YOUTUBE_CLIENT_ID"
[ -n "$YOUTUBE_CLIENT_SECRET" ] && echo "CLIENT_SECRET_SET" || echo "MISSING: YOUTUBE_CLIENT_SECRET"
python3 --version 2>/dev/null   && echo "PYTHON_OK"         || echo "MISSING: python3"
```

### Setup

```bash
pip install --quiet --upgrade \
  google-auth \
  google-auth-oauthlib \
  google-auth-httplib2 \
  google-api-python-client
```

### Upload

Write and run the upload script:

```bash
cat > /tmp/replay-youtube.py << 'PYEOF'
import os, sys
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google_auth_oauthlib.flow import InstalledAppFlow

SCOPES = ['https://www.googleapis.com/auth/youtube.upload']
CLIENT_CONFIG = {
    "installed": {
        "client_id":     os.environ['YOUTUBE_CLIENT_ID'],
        "client_secret": os.environ['YOUTUBE_CLIENT_SECRET'],
        "redirect_uris": ["urn:ietf:wg:oauth:2.0:oob", "http://localhost"],
        "auth_uri":  "https://accounts.google.com/o/oauth2/auth",
        "token_uri": "https://oauth2.googleapis.com/token"
    }
}

file_path = sys.argv[1]
title     = sys.argv[2] if len(sys.argv) > 2 else "Replay Recording"

flow = InstalledAppFlow.from_client_config(CLIENT_CONFIG, SCOPES)
creds = flow.run_local_server(port=0)
youtube = build('youtube', 'v3', credentials=creds)

body = {
    'snippet': {
        'title':       title,
        'description': 'Recorded by /replay (Claude Code)',
        'tags':        ['replay', 'claude-code']
    },
    'status': {'privacyStatus': 'unlisted'}
}
request = youtube.videos().insert(
    part='snippet,status',
    body=body,
    media_body=MediaFileUpload(file_path, chunksize=-1, resumable=True)
)
response = request.execute()
print(f"https://youtu.be/{response['id']}")
PYEOF

VIDEO_URL=$(python3 /tmp/replay-youtube.py "$OUTPUT_FILE" "Replay $(date '+%Y-%m-%d %H:%M')")
echo "[replay] YouTube URL: $VIDEO_URL"
```

---

## Local File Only

No upload, no credentials. Saves to `./replay-recordings/` in the current project.

### Upload (save locally)

```bash
mkdir -p ./replay-recordings
cp "$OUTPUT_FILE" "./replay-recordings/$(basename "$OUTPUT_FILE")"
VIDEO_URL="./replay-recordings/$(basename "$OUTPUT_FILE")"
echo "[replay] Saved locally: $VIDEO_URL"
ls -lh "$VIDEO_URL"
```
