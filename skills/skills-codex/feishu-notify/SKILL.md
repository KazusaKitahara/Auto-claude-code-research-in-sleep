---
name: "feishu-notify"
description: "Send an explicitly requested Feishu/Lark message or a status notification covered by existing user authorization. Supports configured webhook and interactive bridge routes."
---

# Feishu/Lark Notification

Apply [ARIS task scope and run limits](../shared-references/effort-contract.md#task-scope-and-run-limits) when interpreting defaults, checkpoints, and downstream calls.

Send the authorized notification described by **$ARGUMENTS**. Other ARIS skills may call this utility only when the user has authorized their notifications to the configured destination. A config file or a caller's suggestion alone is not messaging authorization.

## Configuration

Read `${CODEX_HOME:-~/.codex}/feishu.json` inside the sending process. Do not print the config, webhook URL, tokens, or authorization headers to the transcript. A missing file or `mode: off` disables automatic notifications. For an explicit send request, explain a missing route instead of claiming delivery.

```json
{
  "mode": "push",
  "webhook_url": "https://open.feishu.cn/open-apis/bot/v2/hook/YOUR_WEBHOOK_ID",
  "interactive": {
    "bridge_url": "http://localhost:5000",
    "timeout_seconds": 300
  }
}
```

- `off`: no notifications.
- `push`: send a status card and return after checking delivery.
- `interactive`: use the configured Feishu bridge to request and receive a decision.

Inspect only redacted configuration facts if setup needs diagnosis, such as mode and whether the required fields are present. Preserve existing user configuration.

## Push workflow

1. Resolve the destination and notification scope from existing user authorization.
2. Prepare the card as JSON data in `.aris/feishu-message.json` using the host's file-writing tool. Include only the requested title and body; omit credentials and unrelated project contents.
3. Send it with the configured route. This example reads secrets inside Python and does not interpolate message text into shell source:

```bash
python3 - <<'PY_SEND'
import json
import os
from pathlib import Path
from urllib.error import URLError
from urllib.request import Request, urlopen

config_path = Path(os.environ.get("CODEX_HOME", str(Path.home() / ".codex"))) / "feishu.json"
if not config_path.exists():
    print("Feishu disabled: config missing")
    raise SystemExit(0)
config = json.loads(config_path.read_text())
if config.get("mode", "off") == "off":
    print("Feishu disabled")
    raise SystemExit(0)
url = config.get("webhook_url")
if not url:
    raise SystemExit("Feishu delivery unavailable: webhook missing")
payload = json.loads(Path(".aris/feishu-message.json").read_text())
request = Request(url, data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
                  headers={"Content-Type": "application/json"}, method="POST")
try:
    with urlopen(request, timeout=30) as response:
        result = json.load(response)
except (URLError, TimeoutError, ValueError):
    raise SystemExit("Feishu delivery unconfirmed: request failed; do not blindly resend")
if result.get("code", result.get("StatusCode")) != 0:
    raise SystemExit("Feishu rejected the notification")
print("Feishu delivered")
PY_SEND
```

A card payload has this shape; generate valid JSON rather than replacing placeholders inside shell commands:

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {"title": {"tag": "plain_text", "content": "Experiment complete"}, "template": "green"},
    "elements": [{"tag": "markdown", "content": "The requested run finished. Results: ..."}]
  }
}
```

An ambiguous network failure is not proof that the message was unsent. Record delivery as unconfirmed instead of automatically duplicating it. A failed status notification does not block the underlying research work.

## Interactive decisions

The optional [Feishu bridge](https://github.com/joewongjc/feishu-claude-code) uses `POST /send` with `{type, title, body, options}` and `GET /poll?timeout=...` for replies. Use its installed API and associate the returned reply with the current request; do not invent capabilities or consume an unrelated decision.

- Send the concrete checkpoint only when messaging and that interactive route are authorized.
- Wait through a cancellable host mechanism, keeping the current decision pending. A reply such as `approve`, `reject`, or custom instructions applies only to the presented scope.
- **Timeout is not approval.** Return `timeout` to the caller. A required user decision remains pending; continue only independent work already authorized.
- If the bridge is unavailable, report that the decision route is unavailable and present the same checkpoint in the current conversation. Do not silently downgrade an approval gate to push-only mode.
- `AUTO_PROCEED=true` may make a workflow selection checkpoint informational when no user decision is required. It cannot override a deliberately enabled interactive approval gate.

## Caller events

| Event | Typical caller | Content |
|---|---|---|
| `experiment_done` | run-experiment, monitor-experiment | Status and requested results |
| `review_scored` | auto-review-loop, auto-paper-improvement-loop | Actual review score, verdict, and key findings |
| `checkpoint` | idea-discovery, research-pipeline | Concrete choice needing the user's decision |
| `error` | Authorized workflow | Concise failure and next action |
| `pipeline_done` | Authorized workflow | Deliverables and unresolved items |
| `custom` | Explicit user invocation | Requested message |

Every caller checks both authorization and configuration before sending. Automatic notifications remain a no-op when disabled. Never report success without a successful transport response.
