# Enkidu

A single-user, server-side video transcoder for a home media server. It wraps
`ffmpeg` in a web UI that knows what the host's hardware and ffmpeg build can
actually do, offers encoding presets by content type and quality tier, and runs
the encode server-side with live progress.

The purpose is to produce smaller, high-quality copies of a media library for
remote streaming over constrained internet — **without ever touching the
originals**.

## Status

Pre-alpha. Nothing is implemented yet; the specification is settled and the
build is starting at Phase 1 (capability detection).

## What it is not

Deliberately out of scope: library automation (no watch folders, no rules
engine — Tdarr and FileFlows already do that), multi-user support or
authentication, Docker distribution, media-server API integration, disc
ripping, and lossless remuxing. Source files are never modified or deleted
automatically.

## Platform

Debian 13 (trixie), installed as a `.deb`:

```
apt install ./enkidu_x.y.z_amd64.deb
```

Runs as an unprivileged `enkidu` system user under systemd, binds the host's
LAN address on port `8420`, and assumes a private network or an access proxy in
front of it. It must never be exposed directly to the public internet.

## Stack

Python 3.11+ / FastAPI / Jinja2 / HTMX + Alpine.js / SQLite. No node build
step, by design — it keeps the `.deb` pipeline trivial.

## License

Apache License 2.0. See [LICENSE](LICENSE).
