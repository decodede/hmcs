# Changelog — HMCS vs official CloudStream

HMCS is a fork of [recloudstream/cloudstream](https://github.com/recloudstream/cloudstream)
tuned for ultra-low-end Android TV boxes (1 GB RAM, 4 GB storage — Allwinner/Rockchip sticks,
Android Go, old Fire TV).

This file tracks every HMCS release, which upstream CloudStream version it is based on,
and everything the fork changes. If you know official CloudStream, this document is the delta.

## How to read versions

| Field       | Rule                                                 | Example                                          |
| ----------- | ---------------------------------------------------- | ------------------------------------------------ |
| versionName | `<upstream versionName>-hmcs.<fork revision>`        | `4.9.1-hmcs.3` = upstream 4.9.1, fork revision 3 |
| versionCode | fork's own sequence, starts at `100`, +1 per release | independent from upstream's (upstream is `68`)   |
| `Base:`     | upstream commit the release is synced to             | `5d66bbe`                                        |

When upstream publishes a new release, its versionName/versionCode is adopted in the next HMCS
release (fork revision resets). See `gradle/libs.versions.toml`.

## Upstream sync status and history

| HMCS release | Upstream base | Upstream version    | What came with it                                                                                                 |
| ------------ | ------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `v1.0.0`     | `9ff05a8b`    | **4.9.0** (code 68) | Initial fork. Low-memory overhaul, NSFW removed, audio/subtitle language prefs.                                   |
| `v1.1.0`     | `5d66bbe`     | **4.9.1** (code 68) | Upstream 4.9.1 sync: HW+ audio-only renderer, `Fixed launch (#3316)`, BackupUtils rewrite, `@Prerelease` cleanup. |
| `v1.2.0`     | `5d66bbe`     | **4.9.1** (code 68) | Fork-only resilience fixes: bounded auto re-prepare + stall watchdog.                                             |

Upstream release history: https://github.com/recloudstream/cloudstream/releases
At the time of writing, the newest upstream release is **4.9.1** (this is what HMCS tracks).

## How to sync with a new upstream release (recipe)

Everything below is run from the HMCS repo; upstream releases are merged as individual
cherry-picks so fork changes are never reverted blindly:

```bash
git remote add upstream https://github.com/recloudstream/cloudstream.git
git fetch upstream master                       # or fetch --tags
git cherry-pick <new upstream commits...>       # one at a time

# Conflicts to expect:
#   1. app/src/main/java/.../ui/player/CS3IPlayer.kt
#      - renderers factory block (fork's decoder fallback + async MediaCodec queueing vs upstream)
#      - keep BOTH: upstream branch + fork's fallback/async additions (see v1.1.0 notes)
#   2. gradle/libs.versions.toml
#      - upstream bumps versionCode/versionName; HMCS uses <upstream>-hmcs.N and code >= 100
#   3. library/api/jvm/library.api
#      - upstream does NOT regenerate its ABI dump when it adds public API;
#        hand-add the new entries until library:checkKotlinAbi passes again
#      - run `./gradlew library:checkKotlinAbi` via CI; it prints the exact missing lines

git push                                        # CI must go green (library ABI + assemble)
# bump version in gradle/libs.versions.toml, add entry to this file,
# tag vX.Y.Z, push tag — .github/workflows/release.yml publishes the APKs
```

Rules we follow for syncs:

- Fork changes are never reverted to land upstream changes; we merge by hand, keeping both.
- No full history merges: HMCS was created from a snapshot and uses upstream as a cherry-pick
  source, so upstream and fork history stay separate but diff-friendly.
- CI is the only gate (no on-device test rig): `library:checkKotlinAbi` + `assembleStableDebug`
  per push, `assembleStableRelease` per release tag.

## Release notes by version

Format: [Keep a Changelog](https://keepachangelog.com/). Newest first.

## [v1.2.0] - 2026-11-10

Base: `recloudstream/cloudstream` @ `5d66bbe` (upstream 4.9.1). Version `4.9.1-hmcs.3`, code `102`.

### Fixed

- **Bounded auto re-prepare on transient playback errors** (`CS3IPlayer`): IO family
  (network failed/timeout, bad HTTP status, invalid content type, file not found,
  remote error) and parse family (`PARSING_CONTAINER_UNSUPPORTED`/`MALFORMED`,
  `PARSING_MANIFEST_MALFORMED`) are re-prepared automatically with a 4-attempt budget.
  Previously these either spun forever (user had to go back and replay — issue seen with
  flaky m3u8 CDNs) or, for delayed-start live links, never recovered.
- **Stall watchdog** (`CS3IPlayer`): every 2.5 s poll; ~8 s in `STATE_BUFFERING` with no
  position progress forces a re-prepare within the same 4-attempt budget.
  Directly targets "live link gets stuck, VLC plays it fine" (media3 never re-prepares
  itself; Nuvio and similar players retry).
- **Live rejoin bounded**: `ERROR_CODE_BEHIND_LIVE_WINDOW` and
  `HlsPlaylistTracker.PlaylistStuckException` now use the 4-attempt budget
  (previously unbounded prepare loops). After the budget is exhausted the error surfaces
  to the mirror failover path as before.
- Behavior note: upstream's unbounded `IO_NETWORK_CONNECTION_FAILED → prepare()` special
  case is subsumed by the bounded retriable class; 4 failures now surface to mirror
  failover instead of looping forever.

### Not changed

- Decoder errors are intentionally NOT retried (they do not self-heal).
- No `LoadErrorHandlingPolicy` subclass shipped: upstream default retry semantics retained;
  recovery happens at the app layer.
- `#3094`-class transient `3003` on delayed-start live links: covered by the retriable class.
- `#3218`-class silent stalls with frozen position: covered by the stall watchdog.

## [v1.1.0] - 2026-11-10

Base: `recloudstream/cloudstream` @ `5d66bbe` (v4.9.1 / code 68 + fork code 101).
Fork versionName `4.9.1-hmcs.2`, versionCode `101`.

Upstream sync (cherry-picked, no reverts of fork changes):

- `468f783` Added HW+ (#3312): audio-only renderer factory (`FixedNextRenderersFactoryWithoutVideo`)
  and new "HW+" / "HW+SW audio-sw/hw" software-decoding preference values (enum values 3 and 4).
- `c92e0cf` Bump 4.9.1 + remove `@Prerelease` annotations.
- `5d66bbe` Fixed launch (#3316): `subsPathPicker` launch via shared `launchSafe`,
  plus assorted cleanup (`BackupUtils` rewrite, unused imports).

HMCS deltas over upstream additionally get current `master` history via upstream sync branch;
all fork changes from v1.0.0 remain intact (async MediaCodec queueing and renderers-factory
fallback now also cover the upstream `isAudioOnly` path).

### Fixed

- Merge-time fix: `FixedNextRenderersFactoryWithoutVideo` also gets decoder fallback + async
  MediaCodec queueing (matches the other renderer factories).

## [v1.0.0] - 2026-11-09

Base: `recloudstream/cloudstream` @ `9ff05a8b` (v4.9.0 / code 68).

### Added

- **DeviceProfile** (`utils/DeviceProfile.kt`): LOW/MID/HIGH tier detection from RAM,
  `isLowRamDevice`, 64-bit support, zRAM swap size, Android Go and kernel age, plus a
  broken-GPU chip list. Drives player buffers, Coil and the HTTP client.
- **Preferred Audio & Subtitle Sources**: new settings screen section
  (Settings → Player → Preferred Audio & Subtitle Sources) with primary/secondary audio
  and subtitle language pickers (full SubtitleHelper language list).
- **Episode continuity**: explicit source/audio/subtitle choices are remembered per series
  (stored via the app's key store, survives restarts) and reused for later episodes.
- **Dub-aware source auto-selection**: preferred source → primary audio language match →
  "dub"/"dubbed" heuristic → secondary audio language → stream default.
- **Fast start**: playback begins as soon as the preferred source link arrives (or any
  usable link at auto-skip priority), instead of waiting for all extractors.
- **Hierarchical audio track selection** replacing the single-language check:
  remembered track → primary language → secondary → stream default.
- Subtitle auto-select rewritten: explicit "None" strictly disables subtitles (no silent
  fallback to stream defaults); remembered series choice → primary → secondary → stream.
- Link resolution now runs on `Dispatchers.IO` with a 60 s overall timeout
  (`PlayerGeneratorViewModel.loadLinks`).
- GitHub Actions: `ci.yml` (library ABI check + `assembleStableDebug` on every push) and
  `release.yml` (`assembleStableRelease` + APK upload to Releases on tag `v*`).
- This changelog; simple README; GPL-3.0 license kept.

### Changed

- **ExoPlayer buffers scale with tier** (target/max/back): potato ≤1024 MB → 6 MB/15 s/2 s;
  LOW → 12 MB/25 s/5 s; MID → 48 MB/45 s/15 s; HIGH → 128 MB/60 s/30 s.
  Byte thresholds are prioritized (`setPrioritizeTimeOverSizeThresholds(false)`).
- ExoPlayer: decoder fallback enabled everywhere (HW→SW or SW→HW on init failure),
  **async MediaCodec queueing** (unties decode handshakes from the playback thread),
  seamless-only frame rate matching, `setLiveTargetOffsetMs` on the main media source factory.
- Disk video cache bypassed on ultra-potato devices (prevents SQLite locking and flash
  thrash on slow eMMC); otherwise the existing `video_buffer_disk_key` setting applies.
- Coil: memory cache 5% (potato) / 10% / 20% / 25% heap by tier (was flat 10%);
  crossfade disabled on LOW (was 200 ms always); RGB_565 bitmaps on LOW (halves poster
  memory; ARGB_8888 otherwise); disk cache capped at 128 MB on LOW (was 512 MB + 4% storage);
  software `BitmapFactoryDecoder` used on LOW in addition to broken-hardware devices.
- OkHttp: HTTP/2, tier-scaled dispatcher (16–96 max requests) and connection pool,
  10–15 s timeouts, disk cache 15–100 MB by tier (was fixed 50 MB).
- TV player + trailer `PlayerView` use `SurfaceView` (zero-copy hardware overlay).
- `LinearListLayout` sets `initialPrefetchItemCount` (2 potato / 4 default / 6 high tier).
- Process-wide memory-pressure callback evacuates idle HTTP sockets
  (`ComponentCallbacks2` in `CloudStreamApp`) under `TRIM_MEMORY_UI_HIDDEN` and above.
- Native libs packaged **uncompressed** (mmap from APK) — removes install-time extraction
  doubling disk usage on 4 GB storage boxes (upstream used legacy compressed packaging).
- `hardwareAccelerated=true` on the application (budget TV firmwares sometimes disable it).
- Launcher label **"HMCS"** in the stable flavor; applicationId suffix `.hmcs`
  (`com.lagradost.cloudstream3.hmcs`) so the fork co-installs with upstream CloudStream.
- Stable release version bumped: versionName `4.9.0-hmcs.1`, versionCode `100`.
- Release APKs are ABI-split (`arm64-v8a`, `armeabi-v7a`) plus a `universal` APK.

### Removed

- **NSFW/18+ surfaces**: home "NSFW" genre chip (and its layout chip), NSFW entry in
  setup/settings "preferred media" multi-select, the "18+" filter chip in Extensions,
  the "18+" badge on extension cards, the `nsfw_marker` views in repository lists,
  the "Enable NSFW on supported Extensions" toggle, the "NsfwOnly" plugin auto-download
  option. Adult plugins are never listed or discoverable. `TvType.NSFW` itself remains
  (extension API compatibility).
- Upstream workflows that required the upstream org's secrets (`prerelease`,
  `build_to_archive`, `dokka`, `update_locales`, `pull_request`, `instrumented-tests`) —
  they failed 100% outside that org; replaced by `ci.yml`/`release.yml`.

### Fixed

- `PlayerGeneratorViewModel.loadLinksPrev()` incremented the episode index
  ("Previous" button skipped forward one episode).
- Home rows no longer grow unbounded with duplicated items when expanding
  (`distinctBy` result was discarded).

### Deliberately NOT changed / NOT done

- **No full Jetpack Compose rewrite.** Upstream's UI is ~5% Compose; a blind migration of
  ~150 XML layouts cannot be device-verified here and would destabilize exactly the devices
  this fork targets. Compose pockets (settings, subtitles, account) stay; a staged migration
  can happen later with on-device profiling.
- **Cut:** R8/minify (unverifiable here; reflection-heavy plugin/Gson paths risk release-only crashes).
- **Kept:** `largeHeap="true"` (ExoPlayer buffering resets under default heap caps, per upstream comment).
- Preview seekbar bitmap handling left as upstream (recycling bitmaps still drawn caused draw-time crashes).
- Home/search/library still load full lists per page (no Paging re-architecture).
- **Not ported from abir614/cloudstream** (by design): Rust native core, NSFW AI classifier,
  telemetry/crash reporting to a private endpoint, SafeDns, `StreamSecurityUseCase`,
  permissive-SSL trust managers, plugin sandboxing/fingerprinting, update security gate,
  DoH resolver changes, main-page disk cache, zero-stall mirror failover, resume-playback toggle.
  These add memory/CPU cost, break working stream links, or add no user value here.
- **No on-device testing.** Verification is CI-only: `library:checkKotlinAbi` +
  `assembleStableDebug` on every push, `assembleStableRelease` for releases.
  Runtime behaviour on real hardware is UNVERIFIED.

### Known issues

- Playback errors `2001/2002/2003/2004` and `ERROR_CODE_PARSING_CONTAINER_UNSUPPORTED`
  are ~always source/network problems (expired signed links, hotlink/Referer/UA blocking,
  geo-IP, wrong box clock, HTML challenge pages). The app cannot fix a dead source; see
  README troubleshooting. An external player (mpv/VLC) bypasses this path entirely.
- Files >3 GB on a 1 GB RAM box may still OOM in the in-app player regardless of settings —
  force 720p/1080p or use an external player.
- versionCode must be increased manually for each new stable release
  (`gradle/libs.versions.toml`); stable releases are not auto-incremented.
