# Webhook alerts for Home Assistant

[![Validate](https://github.com/SudoManaged/ha-webhook-alerts/actions/workflows/validate.yml/badge.svg)](https://github.com/SudoManaged/ha-webhook-alerts/actions/workflows/validate.yml)
![Home Assistant 2024.11+](https://img.shields.io/badge/Home%20Assistant-2024.11%2B-41BDF5)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Let any script, cron job or device send an alert to your phone and to Home Assistant with one HTTP POST.

A free, open-source Home Assistant blueprint by SudoManaged. It runs entirely on your own Home Assistant: no
add-ons, no custom integrations, no cloud services of its own, no accounts. It started as an automation that runs every day
on our own Home Assistant, and was then generalized and tested for release.

## Contents

- [Requirements and tested versions](#requirements-and-tested-versions)
- [Installation](#installation)
- [What it is for](#what-it-is-for)
- [Use cases](#use-cases)
- [How it works](#how-it-works)
- [Message format](#message-format)
- [Settings](#settings)
- [Sending alerts: examples](#sending-alerts-examples)
- [Choosing a webhook ID](#choosing-a-webhook-id)
- [Test your setup](#test-your-setup)
- [Which senders count as local](#which-senders-count-as-local)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Updating and removing](#updating-and-removing)
- [Security](#security)
- [Reporting problems and contributing](#reporting-problems-and-contributing)
- [More free Home Assistant tools from SudoManaged](#more-free-home-assistant-tools-from-sudomanaged)
- [Changelog and license](#changelog-and-license)

## Requirements and tested versions

- **Home Assistant 2024.11 or newer.** The blueprint declares this, and Home Assistant refuses to import it from
  the UI on an older release.
- The **logbook** integration (part of Home Assistant's default configuration). Without it the alert stops at its
  first step.
- For phone notifications: the **Home Assistant app** on your phone, or any other notify action.
- It passes Home Assistant's configuration check on **2024.11.0**, **2026.3.3** and **2026.11.0**, and was run
  live on all three with 20 kinds of request: alert, info, no level, a body that is not JSON, invalid JSON, an
  empty body, a JSON list, the texts "None", "..." and "[1, 2]", a numeric source, the values `0` and `false`, a
  long source, a body over 256 KB, accented text, a GET request, an unknown ID, the "app options" off, and `info`
  as the default level.
- Every change to this repository is checked automatically against Home Assistant 2024.11.0 and the current
  release (see [`.github/workflows/validate.yml`](.github/workflows/validate.yml)).

## Installation

[![Import the Webhook alerts blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FSudoManaged%2Fha-webhook-alerts%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fsudomanaged%2Fwebhook_alerts.yaml)

**With the button:** click **Import blueprint** above. It opens your own Home Assistant (through
my.home-assistant.io) with the blueprint ready to import. Then go to **Settings, Automations & scenes,
Blueprints**, open the blueprint and click **Create automation**.

**By hand:** copy [`blueprints/automation/sudomanaged/webhook_alerts.yaml`](blueprints/automation/sudomanaged/webhook_alerts.yaml)
to `/config/blueprints/automation/sudomanaged/webhook_alerts.yaml` in your Home Assistant configuration folder,
then reload automations (**Developer tools, YAML, Automations**) or restart Home Assistant. The blueprint then
appears under **Settings, Automations & scenes, Blueprints**.

**In YAML** (`automations.yaml`), if you prefer:

```yaml
- id: webhook_alerts
  alias: Webhook alerts
  use_blueprint:
    path: sudomanaged/webhook_alerts.yaml
    input:
      webhook_id: YOUR_LONG_RANDOM_ID
      notify_action: notify.mobile_app_my_phone
```

## What it is for

Backups, sync jobs, update scripts, a Raspberry Pi in the garage: lots of things can tell when something went
wrong, but have no easy way to tell *you*. This blueprint gives them one: a single HTTP POST to a Home Assistant
webhook, and the alert lands on your phone (through the Home Assistant app or another notify action) and in Home
Assistant's logbook. Routine "all is well" messages can be sent too, and stay quietly in the logbook.

## Use cases

- Backup jobs (rsync, restic, Veeam, Time Machine scripts) reporting failure, and an `info` heartbeat on success.
- Scheduled scripts on servers or Windows PCs: update runs, certificate renewals, disk space checks.
- Devices that can send an HTTP POST with a JSON body: NAS boxes, routers, 3D printers, Raspberry Pis, ESP32
  boards.
- Other systems' webhooks pointed at Home Assistant, if you can set the JSON body they send.

## How it works

1. A script sends a POST with a small JSON body to `http://YOUR_HA:8123/api/webhook/YOUR_WEBHOOK_ID`.
2. Every message is written to the logbook. The entry shows the source, then `<level>: <message>`, for example
   **Nightly backup** alert: Backup failed: disk full.
3. If the level is anything other than `info`:
   - a notification is created inside Home Assistant (the bell), if you turned that on;
   - the alert is sent to your notify action.
4. Each source gets its own tag (`webhook-alert-<source>`). With the "Home Assistant app options" on, a new alert
   from the same source replaces the previous one on the phone instead of stacking up. Inside Home Assistant
   there is one notification per source.

Messages are handled one at a time, in order (`queued` mode): one is handled while up to 19 more wait. While the
queue is full, further messages are dropped and Home Assistant logs a warning.

## Message format

POST a JSON object with `Content-Type: application/json`. All fields are optional:

| Field | Meaning |
|---|---|
| `source` | Who is reporting, for example `Nightly backup`. Shown as the notification title. Longer than 60 characters is cut to 60. Default: `Webhook alert`. |
| `level` | `info` goes to the logbook only. Anything else (`alert`, `error`, `warning`...) also notifies. Not case-sensitive. Default: the blueprint's "Level when a message has none". |
| `message` | The text of the alert. Longer than 900 characters is cut to 900, ending in `...`. Default: `(no message)`. |

What happens with other bodies:

| The request | What happens |
|---|---|
| JSON header, but the body is not valid JSON | Home Assistant drops it before the automation runs: no alert, no trace, and the log says "Error processing webhook". |
| JSON header and an empty body | Depends on the Home Assistant release: 2024.11 drops it like invalid JSON; newer releases treat it as no fields. |
| No JSON header (form data or plain text) | The automation runs as if no fields were sent: a message "(no message)" from "Webhook alert" at the default level, so a misconfigured sender still gets your attention (unless the default level is `info`). |
| JSON that is not an object (for example a list) | The same as no fields. |
| Field names other than `source`, `level` and `message` | Ignored. |
| Numbers or `true`/`false` instead of text | Used as text (`"source": 123` shows as "123", `false` as "False"). A text that is only a number may be shown reformatted (`1.50` as 1.5). |
| The text `None` or `...` | Shown as "(none)" or ". . .": Home Assistant would otherwise turn them into other values. |
| A very large body | Fine: only the three fields are read, and the source and message are cut to length. |

## Settings

| Setting | Default | Explanation |
|---|---|---|
| **Webhook ID** | (required) | The secret part of the URL. Use a long random value; see [Choosing a webhook ID](#choosing-a-webhook-id). |
| **Local network only** | On | Accept requests only from private network addresses; see [Which senders count as local](#which-senders-count-as-local). Turn it off only if your senders reach Home Assistant from outside your network. |
| **Notify action** | `notify.notify` | Where alerts go. For one phone use `notify.mobile_app_<your_phone>` (find the exact name under **Developer tools, Actions**, search `notify.`). With the Home Assistant app installed, `notify.notify` sends to every phone that has the app. Notify groups defined in YAML work too. Notify *entities* (used with `notify.send_message`, including notify groups made in the UI) are not supported. |
| **Home Assistant app options** | On | Sends the app's options with each alert: a tag per source, so a new alert replaces the previous one, and high priority with `ttl: 0`, so Android delivers it at once even when the phone is idle (iOS ignores these two). Turn it off if your notify action is not the Home Assistant app and rejects the extra data. |
| **Also show alerts inside Home Assistant** | On | Creates a Home Assistant notification (the bell) for each alert, one per source. |
| **Level when a message has none** | `alert` | Use `info` if your senders mostly send routine messages and only mark problems explicitly. |

## Sending alerts: examples

Replace `homeassistant.local:8123` with your Home Assistant's address and `YOUR_WEBHOOK_ID` with your ID.

**curl (Linux and macOS)**

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"source":"Nightly backup","level":"alert","message":"Backup failed: disk full"}' \
  http://homeassistant.local:8123/api/webhook/YOUR_WEBHOOK_ID
```

On Windows, use the PowerShell example below: in the Windows command prompt and in Windows PowerShell 5.1 the
quoting above does not work.

**A shell script that reports its own result**

Build the JSON with [jq](https://jqlang.org/) so quotes, backslashes and line breaks in a message cannot break it
(a broken body is dropped without an alert, see [Message format](#message-format)):

```bash
#!/bin/bash
HA_ALERT=http://homeassistant.local:8123/api/webhook/YOUR_WEBHOOK_ID
alert() {  # alert <level> <message>
  jq -nc --arg s "Photo sync" --arg l "$1" --arg m "$2" '{source: $s, level: $l, message: $m}' |
    curl -s -m 10 -X POST -H "Content-Type: application/json" --data-binary @- "$HA_ALERT" >/dev/null
}
if rsync -a /photos/ /mnt/backup/photos/; then
  alert info "Sync finished"
else
  alert alert "rsync failed with exit code $?"
fi
```

**PowerShell (Windows scheduled tasks)**

```powershell
$body = @{ source = 'Veeam'; level = 'alert'; message = 'Last backup did not complete' } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri 'http://homeassistant.local:8123/api/webhook/YOUR_WEBHOOK_ID' -Method Post `
  -ContentType 'application/json; charset=utf-8' -Body ([Text.Encoding]::UTF8.GetBytes($body)) -TimeoutSec 10
```

Sending the body as UTF-8 bytes matters on Windows PowerShell 5.1: otherwise accented letters and symbols such as
`°` are sent in a different encoding, the body is not valid JSON for Home Assistant, and the alert is dropped.

**Python**

```python
import requests  # pip install requests

requests.post(
    "http://homeassistant.local:8123/api/webhook/YOUR_WEBHOOK_ID",
    json={"source": "Garage Pi", "level": "alert", "message": "Freezer temperature above -10 C"},
    timeout=10,
)
```

Keep the webhook URL in a file or environment variable your script reads, not in the script itself, especially
if the script is shared or kept in version control.

## Choosing a webhook ID

The webhook ID works like a password: anyone who can reach your Home Assistant and knows it can send you alerts.
Generate a long random one, for example:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
# or
openssl rand -hex 32
```

To change it later, edit the automation, paste a new ID and update your senders.

## Test your setup

After creating the automation, send a test from a computer on your network:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"source":"Test","level":"alert","message":"Webhook alerts works"}' \
  http://homeassistant.local:8123/api/webhook/YOUR_WEBHOOK_ID
```

You should get a phone notification titled "Test", a Home Assistant notification (if turned on) and a logbook
entry **Test** alert: Webhook alerts works. Send the same with `"level":"info"` and only the logbook entry should
appear.

A `200` answer does not prove anything (see [Troubleshooting](#troubleshooting)); the logbook entry does.

## Which senders count as local

With "Local network only" on, Home Assistant accepts a request only when it comes straight from one of these
addresses:

- loopback: `127.0.0.0/8`, `::1`
- private IPv4: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- private IPv6: `fd00::/8`
- link-local: `169.254.0.0/16`, `fe80::/10`

Everything else is refused, including:

- requests through Home Assistant Cloud (Nabu Casa) remote access, always;
- Tailscale IPv4 addresses (`100.64.0.0/10`), which are not in the private ranges (Tailscale's IPv6 addresses,
  `fd7a:115c:a1e0::/48`, are inside `fd00::/8` and do count as local);
- devices on your own network that reach Home Assistant over a public IPv6 address, or through a tunnel or proxy
  (see below).

Refused requests still get a `200` answer; the log says "Received remote request for local webhook". If one of
your senders is in this list, either have it use Home Assistant's private address, or turn "Local network only"
off and rely on the long random webhook ID.

**Behind a reverse proxy** (nginx, Caddy, Traefik and so on), Home Assistant sees the proxy's address:

- If the proxy sends `X-Forwarded-For` (most do), you must set `use_x_forwarded_for: true` and `trusted_proxies`
  in the `http:` section, or every request through the proxy is refused with a `400` error. With them set, Home
  Assistant uses the real sender's address.
- If the proxy does not send `X-Forwarded-For`, every request looks like it comes from the proxy, which is usually
  local, so **requests from the internet through the proxy count as local**. Configure the proxy to send the
  header, or keep this webhook off the proxy.

## Troubleshooting

Home Assistant answers `200` to a POST even when the ID is unknown, the sender is not local or the body is
invalid, so the answer tells you little. Check these instead:

- **No logbook entry and no trace** (open the automation, then **Traces**):
  - the webhook ID in the URL does not match the automation;
  - the sender was refused as not local: **Settings, System, Logs** shows "Received remote request for local
    webhook" (see [Which senders count as local](#which-senders-count-as-local));
  - the body was not valid JSON: the log shows "Error processing webhook". Check quoting, and on Windows the
    encoding (see the PowerShell example).
- **A `405` answer.** The request was not a POST (a browser opening the URL sends GET).
- **A `400` answer.** You are behind a reverse proxy that sends `X-Forwarded-For` without `use_x_forwarded_for`
  and `trusted_proxies` set (see [Which senders count as local](#which-senders-count-as-local)).
- **The alert arrives as "(no message)" from "Webhook alert".** The `Content-Type: application/json` header was
  missing, the body was not a JSON object, or the fields are named differently (for example `msg` instead of
  `message`).
- **No phone notification, but a logbook entry.** Open the automation's trace: it shows the exact error, for
  example a notify action that does not exist. Check the name under **Developer tools, Actions**.
- **Alerts pile up on the phone instead of replacing each other.** Turn on "Home Assistant app options", and make
  sure the notify action reaches the Home Assistant app.

## Limitations

- POST with a JSON body only: devices that can only open a URL (GET) or send form fields cannot set the source or
  message.
- Notify entities (`notify.send_message`, and notify groups made in the UI) are not supported; use a
  `notify.mobile_app_...` action, `notify.notify` or a notify group defined in YAML.
- High priority and `ttl` only affect Android; iOS delivers notifications its own way.
- The Home Assistant app allows at most 500 push notifications per phone per day (reset at midnight UTC; see the
  [app's documentation](https://companion.home-assistant.io/docs/notifications/notification-details/)). A sender
  in a loop can use them up.
- At most 19 messages wait in the queue behind the one being handled; more are dropped while it is full.
- A message that is exactly one of the Home Assistant app's command words (for example `clear_notification` or
  `request_location_update`) is carried out by the app instead of shown. With "Home Assistant app options" on,
  `clear_notification` clears that source's previous alert.
- One-way: the sender does not learn whether you saw the alert.
- Plain HTTP on your local network unless your Home Assistant itself uses HTTPS.

## Updating and removing

- **Updating:** in **Settings, Automations & scenes, Blueprints**, open the blueprint's menu and choose
  **Re-import blueprint** (or replace the file by hand and reload automations). Your automations keep their
  settings.
- **Removing:** delete the automations that use the blueprint, then delete the blueprint. Nothing else is
  installed.

## Security

- The blueprint is **local-only by default**. The webhook ID works like a password: use a long random one and do
  not publish it.
- If Home Assistant is behind a reverse proxy, read [Which senders count as local](#which-senders-count-as-local):
  a proxy that does not send `X-Forwarded-For` makes internet requests look local.
- Nothing is sent outside your Home Assistant except the notifications you configure.
- To report a security problem privately, use **Report a vulnerability** on this repository's
  [Security tab](../../security). Please do not open a public issue for security problems. See
  [SECURITY.md](SECURITY.md).

## Reporting problems and contributing

- **Something does not work:** open an [issue](../../issues) with your Home Assistant version, what you expected
  and what happened. An automation trace helps a lot; remove anything private from it first.
- **Improvements:** pull requests are welcome. Keep the blueprint self-contained (no custom integrations),
  local-only by default, and documented in this README.

## More free Home Assistant tools from SudoManaged

- [Bluetooth self-heal](https://github.com/SudoManaged/ha-bluetooth-self-heal)
- [Yealink desk phone call log](https://github.com/SudoManaged/ha-yealink-call-log)
- [Dehumidifier kit](https://github.com/SudoManaged/ha-dehumidifier-kit)
- [Internet failover alerts](https://github.com/SudoManaged/ha-internet-failover)
- [Daily schedule from several calendars](https://github.com/SudoManaged/ha-daily-schedule)

## Changelog and license

See [CHANGELOG.md](CHANGELOG.md). Released under the [MIT License](LICENSE). Copyright (c) 2026 SudoManaged.
