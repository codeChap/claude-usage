# Claude Code Usage in i3bar

Display your Claude Code usage limits (weekly, session, Sonnet) in your i3 status bar, in orange.

![Example output: `CC W:25% S:21% 5h:13% R:112h03m`]

## What it shows

- **W:25%** — Weekly all-models limit usage
- **S:21%** — Weekly Sonnet-only limit usage
- **5h:13%** — Current 5-hour session limit usage
- **R:112h03m** — Time until weekly reset

Refreshes every 60 seconds via the Anthropic API.

## Prerequisites

- i3wm with i3bar
- Claude Code CLI installed and authenticated (`claude auth`)
- Python 3
- curl

## Setup

### 1. Create `~/bin/claude-usage`

This script fetches your usage from the Anthropic API using your Claude Code OAuth token.

```python
#!/usr/bin/env python3
"""Fetch Claude Code usage limits for i3bar display."""

import json
import subprocess
import sys
from datetime import datetime, timezone
from pathlib import Path

CREDS_FILE = Path.home() / ".claude" / ".credentials.json"
API_URL = "https://api.anthropic.com/api/oauth/usage"


def get_token():
    with open(CREDS_FILE) as f:
        creds = json.load(f)
    return creds["claudeAiOauth"]["accessToken"]


def refresh_token():
    """Run claude auth refresh to get a new token if expired."""
    subprocess.run(
        ["claude", "--print", "--max-budget-usd", "0", "hi"],
        capture_output=True, timeout=15
    )


def fetch_usage(token):
    result = subprocess.run(
        [
            "curl", "-s",
            "-H", f"Authorization: Bearer {token}",
            "-H", "Content-Type: application/json",
            "-H", "User-Agent: claude-code/2.1.63",
            "-H", "anthropic-beta: oauth-2025-04-20",
            API_URL,
        ],
        capture_output=True, text=True, timeout=10
    )
    return json.loads(result.stdout)


def time_until(iso_str):
    reset = datetime.fromisoformat(iso_str)
    now = datetime.now(timezone.utc)
    diff = reset - now
    total_secs = int(diff.total_seconds())
    if total_secs <= 0:
        return "now"
    hours = total_secs // 3600
    mins = (total_secs % 3600) // 60
    if hours > 0:
        return f"{hours}h{mins:02d}m"
    return f"{mins}m"


def main():
    try:
        token = get_token()
        data = fetch_usage(token)

        if "error" in data:
            # Token might be expired, try refresh
            refresh_token()
            token = get_token()
            data = fetch_usage(token)

        session = data.get("five_hour", {})
        weekly = data.get("seven_day", {})
        sonnet = data.get("seven_day_sonnet", {})

        session_pct = int(session.get("utilization", 0))
        weekly_pct = int(weekly.get("utilization", 0))

        parts = [f"W:{weekly_pct}%"]

        if sonnet:
            sonnet_pct = int(sonnet.get("utilization", 0))
            parts.append(f"S:{sonnet_pct}%")

        parts.append(f"5h:{session_pct}%")

        reset = time_until(weekly.get("resets_at", ""))
        parts.append(f"R:{reset}")

        print("CC " + " ".join(parts))

    except Exception as e:
        print(f"CC ?")


if __name__ == "__main__":
    main()
```

Make it executable:

```bash
chmod +x ~/bin/claude-usage
```

Test it:

```bash
~/bin/claude-usage
# Output: CC W:25% S:21% 5h:13% R:112h03m
```

### 2. Create `~/bin/i3status-wrapper`

This wraps i3status output, prepending the Claude usage as an orange-colored JSON block using the i3bar protocol.

```python
#!/usr/bin/env python3
"""Wrap i3status output with Claude Code usage in orange using i3bar JSON protocol."""

import json
import os
import subprocess
import sys
import time

REFRESH_INTERVAL = 60


def get_claude_usage():
    try:
        result = subprocess.run(
            [os.path.expanduser("~/bin/claude-usage")],
            capture_output=True, text=True, timeout=15
        )
        return result.stdout.strip() or "CC ?"
    except Exception:
        return "CC ?"


def main():
    proc = subprocess.Popen(["i3status"], stdout=subprocess.PIPE, text=True)

    claude_usage = get_claude_usage()
    last_update = time.time()

    # Read and pass through the version header and opening array bracket
    header = proc.stdout.readline().strip()
    print('{"version":1}')
    sys.stdout.flush()

    # Skip the opening [
    proc.stdout.readline()
    print("[")
    sys.stdout.flush()

    first = True
    for line in proc.stdout:
        line = line.strip()
        if not line:
            continue

        # Strip leading comma for subsequent lines
        if line.startswith(","):
            line = line[1:]

        try:
            blocks = json.loads(line)
        except json.JSONDecodeError:
            continue

        # Refresh claude usage periodically
        now = time.time()
        if now - last_update >= REFRESH_INTERVAL:
            claude_usage = get_claude_usage()
            last_update = now

        # Prepend claude usage block in orange
        claude_block = {
            "full_text": claude_usage,
            "color": "#FF8C00",
            "separator": True,
            "separator_block_width": 15
        }

        all_blocks = [claude_block] + blocks

        prefix = "," if not first else ""
        first = False
        print(prefix + json.dumps(all_blocks), flush=True)


if __name__ == "__main__":
    main()
```

Make it executable:

```bash
chmod +x ~/bin/i3status-wrapper
```

### 3. Configure i3status for JSON output

Create or edit `~/.config/i3status/config` and make sure the `general` block includes:

```
general {
        output_format = "i3bar"
        colors = true
        interval = 5
}
```

### 4. Update i3 config

In `~/.config/i3/config`, change the bar section to use the wrapper:

```
bar {
        status_command ~/bin/i3status-wrapper
}
```

### 5. Reload i3

Press `$mod+Shift+r` or run:

```bash
i3-msg reload
```

## API details

The usage data comes from the Anthropic OAuth API at `https://api.anthropic.com/api/oauth/usage`. It uses the OAuth token stored by Claude Code in `~/.claude/.credentials.json` with the `anthropic-beta: oauth-2025-04-20` header.

The API returns:

| Field | Description |
|---|---|
| `five_hour.utilization` | Current session usage % (resets every 5 hours) |
| `seven_day.utilization` | Weekly all-models usage % |
| `seven_day_sonnet.utilization` | Weekly Sonnet-only usage % |
| `seven_day.resets_at` | ISO timestamp of next weekly reset |

## Customization

- **Color**: Change `#FF8C00` in the wrapper to any hex color
- **Refresh rate**: Change `REFRESH_INTERVAL = 60` (seconds) in the wrapper
- **Position**: Move the `claude_block` insertion in `all_blocks` to append instead of prepend
