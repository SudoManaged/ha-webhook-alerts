# Changelog

All notable changes. Versions follow [Semantic Versioning](https://semver.org/).

## 1.0.0 (2026-11-05)

First public release.

- One local-only webhook for alerts from scripts and devices: a JSON POST with `source`, `level` and `message`.
- `info` messages go to the logbook only; other levels also notify, through any notify action, and can create a
  Home Assistant notification.
- Home Assistant app options (on by default): one tag per source so a new alert replaces the previous one, and
  high priority with `ttl: 0` for prompt delivery on Android.
- Bodies without a JSON header, or JSON that is not an object, still produce a message "(no message)" at the
  default level; only the three fields are read, so very large bodies are fine.
- Tested live on Home Assistant 2024.11.0, 2026.3.3 and 2026.11.0.
