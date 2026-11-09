# HMCS

Lightweight fork of [recloudstream/cloudstream](https://github.com/recloudstream/cloudstream)
tuned for ultra-low-end Android TV boxes (1 GB RAM, 4 GB storage — Allwinner/Rockchip sticks,
Android Go, old Fire TV sticks). Same features, far smaller appetite: adaptive buffers,
tier-scaled caches, software-decode fallback, and a previous-episode bugfix.

## Install

Grab the latest APK from [Releases](https://github.com/decodede/hmcs/releases):

- `app-arm64-v8a-*.apk` — most 2016+ TVs and boxes
- `app-armeabi-v7a-*.apk` — old 32-bit boxes
- `app-universal-*.apk` — not sure? take this one

Sideload with a file manager or a phone. Builds are signed with a **public debug keystore**
(or your own via `SIGNING_KEY_*` secrets). The package id is `com.lagradost.cloudstream3.hmcs`,
so it installs **alongside** an upstream CloudStream install instead of colliding with it.

First run: Settings → Extensions → add a repository → pick providers → play.

## What HMCS changes vs upstream

- **Built to survive 1 GB RAM**: device-tiered ExoPlayer buffers (6 MB target on potato
  boxes), RGB_565 posters, software-decode + decoder fallback, tier-scaled Coil/OkHttp limits,
  pinned `.so` mmap packaging (no extraction doubling).
- **Preferred Audio & Subtitle Sources** in Settings → Player: primary/secondary audio and
  subtitle languages; per-series memory of your source/audio/subtitle picks; automatic
  dub-vs-original source preference; player fast-starts when your preferred source arrives.
- **No NSFW/18+ anywhere**: toggles, badges and filters removed; adult plugins never listed.
- **No security gate breaking stream links** (SafeDns/permissive-SSL/sandboxing from forks are
  deliberately not ported). Plain upstream link handling.
- Bugfix: Previous-episode button no longer skips forward.

Full list + rationale: [CHANGELOG.md](CHANGELOG.md)

## Building

Needs JDK 17 and the Android SDK. `./gradlew assembleStableRelease`, APKs land in
`app/build/outputs/apk/stable/release/`. Releases are built and published automatically:
push a tag `v*` or run the `Release` workflow manually.

## Compose?

The Views UI stays. Upstream is ~5% Compose; a blind rewrite cannot be device-verified and
would risk exactly the crashes this fork targets. Settings screens stay Compose as upstream has them.

## Troubleshooting: "No Links Found" / 2001, 2002, 2003, 2004

These codes are ~90% source/network problems, not app bugs. In order:

1. Settings → General → DNS over HTTPS → Cloudflare or Google, then restart the app.
2. Check the box's date and time. Wrong clock = every HTTPS request fails.
3. Clear cache: Settings → Apps → HMCS → Storage → Clear cache.
4. Update extensions; reinstall the one that fails; check the provider's own repo for status.
5. ISP blocking the host? Use a VPN.
6. URL works in a browser but not in any app: CDN requires Referer/UA headers the extension
   doesn't send, or returns an HTML challenge. Pick another source/mirror.
7. Very large (3+ GB) files on 1 GB RAM: force a lower quality, or use an external player
   (mpv/VLC) — see Settings → Player → External player. mpv handles buffering on the C side
   and won't OOM the app's Java heap.

License: GPL-3.0 (same as upstream). Fork credit: upstream CloudStream team,
plus [abir614/cloudstream](https://github.com/abir614/cloudstream) whose device-profile and
buffer work we adapted (without his NSFW AI or security gate).
