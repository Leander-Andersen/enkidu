# Server-Side Transcoder — Project Specification

**Package name:** `enkidu`
**Target platform:** Debian 13 (trixie) LXC on Proxmox VE
**Distribution:** `.deb`, installed via `apt install ./enkidu_x.y.z_amd64.deb`

---

## 1. Purpose

A single-user web application that runs on a media server and produces smaller, high-quality transcoded copies of video files using ffmpeg. It exists to replace hand-written ffmpeg command lines with a UI that:

- knows what the host's hardware and ffmpeg build can actually do,
- offers sensible encoding presets by content type and quality tier,
- exposes full encoder parameters with inline documentation when the user wants them,
- runs the encode server-side and reports progress.

The primary use case is generating lower-bitrate copies of an anime/film library for remote streaming over constrained internet, without altering the original files.

## 2. Non-goals

Explicitly out of scope. Do not build these.

- Library automation: no watch folders, no rules engine, no flow builder. This is a manual, per-job tool. (Tdarr, Unmanic and FileFlows already occupy that space.)
- Multi-user support, authentication, RBAC, or user sessions.
- Docker or container-based distribution.
- Automatically modifying, replacing or deleting source files. Deletion is only ever a manual, post-verification action initiated by the user (see §5.9).
- Media-server API integration. No Emby/Jellyfin library rescan triggering.
- Disc ripping or decryption (MakeMKV's job).
- Lossless remuxing or container surgery (mkvtoolnix's job).

## 3. Deployment and packaging

### 3.1 Package build

Build the `.deb` with [nfpm](https://nfpm.goreleaser.com/) (declarative YAML, single Go binary, no Ruby toolchain). Do not hand-write a `debian/` directory with debhelper.

Build in CI (GitHub Actions) so packages are reproducible.

### 3.2 Package layout

```
/opt/enkidu/
    venv/                  # Python virtualenv, built at package time
    app/                   # application source
    static/                # CSS/JS assets
    data/param-metadata.yaml
/lib/systemd/system/enkidu.service
/etc/enkidu/config.toml   # conffile, not overwritten on upgrade
/var/lib/enkidu/          # SQLite DB, job logs
```

### 3.3 Dependencies

```
Depends:    python3 (>= 3.11), ffmpeg, adduser
Recommends: intel-media-va-driver-non-free, vainfo
Suggests:   libva-drm2, mesa-va-drivers
```

The `Depends`/`Recommends` mechanism is the dependency installer. The application must never invoke `apt` itself.

### 3.4 Maintainer scripts

**postinst:**
1. Create system user `enkidu` with no login shell.
2. Add `enkidu` to the `render` and `video` groups (required for `/dev/dri` access).
3. Create `/var/lib/enkidu/`, owned by `enkidu`.
4. Write a default `config.toml` if absent.
5. `systemctl daemon-reload`, `systemctl enable --now enkidu`.
6. Print the access URL to stdout, e.g. `Enkidu is running at http://<detected-lan-ip>:8420`.

**prerm:** `systemctl stop`, `systemctl disable`.

**postrm (purge only):** remove `/var/lib/enkidu/` and the system user.

### 3.5 Runtime

- Default port: `8420` (configurable in `config.toml`; changing it requires a service restart).
- Default bind address: the host's primary LAN address. **Never default to `0.0.0.0`.**
- Runs as the `enkidu` system user, not root.

### 3.6 Updates

v1 ships bare `.deb` files on GitHub Releases (`apt install ./file.deb`). An APT repository with GPG signing is a v2 concern, not v1.

## 4. Technology stack

| Layer | Choice | Rationale |
|---|---|---|
| Language | Python 3.11+ | Available in Debian 13; matches maintainer familiarity |
| Web framework | FastAPI + Uvicorn | Async, good SSE support |
| Templating | Jinja2 | Server-rendered |
| Interactivity | HTMX + Alpine.js | **No node build step.** Keeps the `.deb` pipeline trivial — no npm, no bundler, no dist artifacts to ship. |
| Persistence | SQLite (stdlib `sqlite3` or SQLModel) | Single file, no server |
| Config | TOML at `/etc/enkidu/config.toml` | Debian conffile semantics |

Do not introduce a React/Vue frontend. The packaging cost is not justified for a single-user internal tool.

## 5. Subsystems

### 5.1 Capability detection

This is the differentiating feature and should be built first.

**Data sources:**

1. `ffmpeg -hide_banner -encoders` — what the ffmpeg *binary* supports.
2. `vainfo --display drm --device /dev/dri/renderD<N>` — what the *silicon* supports.
3. `/dev/dri/render*` — enumerate available render nodes.
4. `/sys/class/drm/card*/device/vendor` and `.../device` — PCI vendor/device IDs.

**The core logic:** an encoder is *usable* only if it appears in ffmpeg's encoder list **and** vainfo reports the corresponding profile paired with an encode entrypoint (`VAEntrypointEncSlice` or `VAEntrypointEncSliceLP`). A profile with only decode entrypoints means decode-only. `ffmpeg -encoders` listing `av1_vaapi` does **not** mean the installed GPU can encode AV1.

**GPU identification:** map the PCI vendor:device ID to a human-readable name using a bundled lookup table covering common Intel Arc/UHD, AMD, and NVIDIA parts. Fall back to displaying the raw PCI ID when unknown.

#### 5.1.1 Detection backends (see decision 8)

The host is surveyed for whatever GPU is present; no vendor is assumed. A single vainfo probe is not sufficient to do that, because the vendors do not all report capabilities through the same interface. The detector therefore runs a set of backends and merges their results into one matrix:

| Backend | Covers | Probe | Encoder suffix |
|---|---|---|---|
| VAAPI | Intel, AMD | `vainfo` profile/entrypoint pairs, as above | `*_vaapi` |
| QSV | Intel only | ffmpeg init test against the render node; QSV is a distinct ffmpeg path from VAAPI even on the same silicon | `*_qsv` |
| NVENC | NVIDIA only | `nvidia-smi` / NVML for device identity, plus an ffmpeg init test per codec. **NVENC capabilities are not visible to vainfo at all** — an NVIDIA card probed only through VAAPI reads as having no encoders. | `*_nvenc` |

Each backend must degrade quietly. A host with no NVIDIA card, no `nvidia-smi`, and no `/dev/nvidia*` reports "NVENC: not present", not an error. The same holds for a host with no `/dev/dri` at all, which is a valid software-only configuration.

The "in the ffmpeg encoder list AND confirmed by the hardware probe" rule applies identically to every backend. Presence of `av1_nvenc` in `ffmpeg -encoders` proves nothing about the installed card, exactly as with `av1_vaapi`.

#### 5.1.2 Codec scope (see decision 7)

| Codec | Software encoder | Hardware encoders |
|---|---|---|
| AV1 | `libsvtav1` | `av1_vaapi`, `av1_qsv`, `av1_nvenc` |
| HEVC (H.265) | `libx265` | `hevc_vaapi`, `hevc_qsv`, `hevc_nvenc` |
| H.264 | `libx264` | `h264_vaapi`, `h264_qsv`, `h264_nvenc` |
| VP9 | `libvpx-vp9` | `vp9_vaapi`, `vp9_qsv` |

Notes that the detector exists to confirm rather than assume:

- **AV1 hardware encode** requires recent silicon — Intel Arc or Meteor Lake and later, AMD RDNA3 and later, NVIDIA Ada (RTX 40-series) and later. Older Intel UHD and pre-RDNA3 AMD are HEVC/H.264 only.
- **VP9 hardware encode is Intel-only.** AMD VCN and NVIDIA NVENC decode VP9 but do not encode it, so no `vp9_amf`/`vp9_nvenc` row exists. Intel VP9 encode support also varies by generation and is not assumed for any particular part.
- `libaom-av1` and `librav1e` are excluded. SVT-AV1 is the AV1 software encoder; libaom is too slow to be useful at library scale.

**Output:** a capability matrix, one row per codec:

| Codec | Software encoder | Hardware encoder | Status | Fix |
|---|---|---|---|---|
| AV1 | `libsvtav1` — available | `av1_vaapi` — available | Usable | — |
| HEVC | `libx265` — available | `hevc_vaapi` — available | Usable | — |
| AV1 (QSV) | — | `av1_qsv` — not in ffmpeg build | Unavailable | `apt install ...` |

Where a codec is unavailable, display the exact `apt install` command that would fix it. **Display only. Never execute it.**

Cache detection results; provide a manual "re-scan" button.

### 5.2 Preset system

Preset schema:

```
id, name, content_type, tier, encoder, rate_control, quality_value,
pixel_format, filters[], encoder_params{}, audio_handling,
subtitle_handling, container
```

- `content_type`: `anime`, `live_action`, `animation_3d`, `grain_heavy`
- `tier`: `fine`, `good`, `great`

Built-in presets are seeded on first run. User-created presets are stored in SQLite and are editable/deletable. Built-ins are read-only but can be duplicated.

**Important design constraint:** VAAPI hardware encoders expose almost no tuning knobs — no `tune`, no `aq-mode`, no psychovisual RDO. On the hardware path, the difference between content-type presets is therefore expressed almost entirely through (a) the quality value and (b) the pre-encode filter chain (e.g. `deband` for anime gradients, light `hqdn3d` for grainy film). The encoder-parameter half of the preset only becomes meaningful on the software path. The schema must accommodate both without a rewrite.

### 5.3 Encoding settings UI

**Always visible:**
- Preset dropdown (content type × tier)
- Quality control (CRF for software, `global_quality`/ICQ for VAAPI), pre-filled from the preset and editable
- Raw extra-args textarea, appended verbatim (see §7 for the security requirement)

**Visible when a software encoder is selected:** the full parameter list for that encoder.

**Control type must match parameter type.** Do not use a slider for a discrete parameter.

| Parameter type | Control |
|---|---|
| Continuous (CRF, `aq-strength`, `psy-rd`) | Slider with numeric entry |
| Enumerated algorithm (`aq-mode`) | Segmented buttons |
| Ordered enum (`preset`) | Ordered dropdown |
| Integer (`bframes`, `ref`) | Stepper |
| Boolean (`--no-sao`) | Toggle |

**Details panel.** Selecting or hovering a parameter shows: what it does, its valid range, and the recommended value **for the currently selected content type**. Recommendations are context-dependent — the right CRF for clean anime is not the right CRF for grainy 35mm film.

Parameter metadata (name, type, range, description, per-content-type recommendations) lives in `data/param-metadata.yaml`, not in code, so copy can be revised without touching the application.

**Scope note for v1:** ship approximately 8–10 parameters per software encoder with high-quality documentation. Place the remainder behind the raw-args textarea. Backfill over time. Writing good copy for all ~40 x265 parameters is a larger task than the rest of the application.

### 5.4 Job execution

- Construct the ffmpeg argument vector from preset + user overrides.
- Spawn with `-nostats -progress pipe:1`. Parse the resulting `key=value` lines on stdout (`frame`, `fps`, `out_time_ms`, `speed`, `progress=continue|end`). Do not scrape stderr.
- Stream progress to the browser over Server-Sent Events.
- Concurrency limit configurable, **default 1**. (Emby runs on the same host and shares the GPU's media engine; concurrent batch encodes will degrade live transcodes.)
- Cancel: `SIGTERM`, then `SIGKILL` after a timeout.
- Persist job history to SQLite: source path, output path, preset used, full argument vector, exit code, duration, output size.
- Capture full ffmpeg stderr per job to `/var/lib/enkidu/logs/<job-id>.log`, viewable in the UI. This is essential for debugging failed encodes.
- Write output to a temporary filename and rename on success, so a failed or cancelled job never leaves a truncated file that looks complete.

### 5.5 Stream handling defaults

Defaults matter here because the library is anime-heavy.

- `-map 0` — carry all streams by default.
- Video: re-encoded.
- Audio: `-c:a copy` by default.
- Subtitles: `-c:s copy` by default.
- **Attachments: `-map 0:t?` — must be preserved.** MKV font attachments are required for styled ASS subtitles to render correctly. Dropping them causes fallback-font rendering. This is a known failure mode and a hard requirement.

Provide a per-job stream selection panel (list all streams from `ffprobe -show_streams`, allow deselecting) but default to keeping everything.

### 5.6 Filter chain construction

VAAPI requires correct hardware-upload ordering. CPU filters must run before `hwupload`:

```
-vaapi_device /dev/dri/renderD128
-vf 'deband=...,format=nv12|p010,hwupload'
```

10-bit output requires `p010` rather than `nv12`. Getting this ordering wrong is the most common source of cryptic VAAPI failures — build and unit-test the filter-graph constructor as a separate, testable component.

### 5.7 Settings page

- Capability matrix (§5.1), with install commands shown as copyable text.
- Detected GPU identity and its supported encode profiles.
- Media root path (read from), output root path (write to).
- Concurrency limit.
- Bind address and port (with a note that changes require a service restart).
- Link to job history and logs.

### 5.8 Sample encode

The feature that justifies building this rather than using an existing tool. Available on the new-job page before committing to a full encode.

**Behaviour:**

1. User selects a file and a preset/settings combination.
2. Enkidu extracts short segments from the source and encodes them with the exact settings that would be used for the full job.
3. The resulting clip is playable in the browser, so banding and artefacting can be judged directly.
4. An estimated final size is displayed alongside.

**Segment selection.** Do not take a single 60-second clip from one point in the file. Encoder bitrate varies enormously with scene complexity, and a sample taken from a static dialogue scene will wildly under-estimate a file containing action sequences.

Default: three 20-second segments at 25%, 50% and 75% of runtime, concatenated. Make the count and duration configurable.

Use `-ss` before `-i` for fast seeking, and re-encode each segment independently before concatenation so each one gets a clean keyframe start.

**Size estimation.** With constant-quality rate control (CRF / ICQ), bitrate is content-dependent, so extrapolation from samples is an estimate and not a prediction.

- Compute video bitrate from the sample segments only (exclude audio and subtitle streams, which are copied and whose size is known exactly from the source).
- Extrapolate: `estimated_video_bytes = sample_video_bitrate x source_duration`.
- Add the exact byte size of all copied streams and container overhead.
- **Present the result as a range, not a single number** — apply roughly +/-25% on the extrapolated video component. Label it clearly as an estimate.
- Store the estimate with the job, and after a full encode completes, record estimate versus actual in the history view. Over time this shows how reliable the estimator is for a given content type.

Sample encodes are ephemeral: write them to a temp directory and clean up on a schedule or on job submission.

### 5.9 Post-encode verification and source deletion

Source files are never deleted automatically. There is no "delete on success" setting, and one must not be added — a single bad preset applied in batch would destroy library content irrecoverably. Deletion is always a deliberate action taken after the user has looked at the result.

**After a job completes, the history/detail view shows a source-versus-output comparison:**

| | Source | Output |
|---|---|---|
| Size | | |
| Duration | | |
| Video codec / profile / bit depth | | |
| Video bitrate | | |
| Resolution | | |
| Audio streams | | |
| Subtitle streams | | |
| Attachments | | |

Populate both columns from `ffprobe`. The attachment row matters — it is the check that font attachments survived.

**Automatic integrity checks, run before the delete control is enabled:**

1. ffmpeg exit code was 0.
2. Output parses under `ffprobe` without error.
3. Output duration matches source duration within a 1-second tolerance.
4. Output stream count matches expectations from the job's stream-selection settings.
5. Output file size is non-zero and plausible.

If any check fails, display the failure and **disable the delete control entirely**. Do not offer an override.

**Delete flow:**

- A "Delete source file" button, shown only when all checks pass.
- Two-step confirmation displaying the full absolute path of the file about to be deleted.
- Deletion is recorded in job history with a timestamp: what was deleted, what replaced it, which preset produced it.
- Deleting the source does not move or rename the output. The output stays where it was written. If the user wants it in the original location under the original name, that is a separate explicit "move output to source path" action, offered after deletion.

Provide a browser-playable preview of the output (or a short excerpt of it) in this view, so verification does not require opening a separate player.

## 6. Pages

1. **New job** — file browser, preset selection, settings, sample encode, submit.
2. **Queue** — active and pending jobs with live progress.
3. **History** — completed jobs, source-vs-output comparison, integrity check results, source-delete control, estimate-vs-actual, link to log.
4. **Presets** — list, duplicate, edit, delete.
5. **Settings** — as §5.7.

## 7. Security requirements

The application ships without authentication and is assumed to be deployed on a private LAN or behind an access proxy. That assumption places the following requirements on the implementation:

1. **Never bind `0.0.0.0` by default.** Bind the detected LAN address; require an explicit config change to widen.
2. **Never execute a shell.** Build ffmpeg invocations as an argument list and pass them to `subprocess` with `shell=False`. There must be no code path where user input is concatenated into a shell string.
3. **Tokenize the raw-args field with `shlex.split()`** and append the resulting tokens to the argument vector. Reject tokens containing shell metacharacters. This field is the primary injection surface in the application.
4. **Constrain all file selection to the configured media root.** Resolve paths with `Path.resolve()` and verify the result is inside the root before use. Reject anything else — this blocks `../` traversal.
5. **Never invoke `apt`, `dpkg`, or any package manager** from the application.
6. Run as the unprivileged `enkidu` user. Do not require root, and do not configure passwordless sudo.

## 8. Resolved decisions

These were open during specification and are now settled. Recorded here so they are not relitigated during implementation.

| # | Decision | Resolution |
|---|---|---|
| 1 | Output destination | Separate configured output root, mirroring the source directory structure. Originals are never automatically modified, moved or deleted. A manual post-verification delete control is provided (§5.9). |
| 2 | Media-server API integration | Out of scope. No Emby integration, no library rescan triggering, and no post-job hook needs to be built for it. |
| 3 | File browser scope | Configured media root only. |
| 4 | Job queue persistence | Persist to SQLite; resume pending jobs on service restart. |
| 5 | Sample encode | **In v1.** Includes estimated final size (§5.8). |
| 6 | Authentication | None. The application is LAN-only or sits behind an access proxy (Cloudflare Access). It must never be exposed directly to the public internet, and §7 is written on that assumption. |
| 7 | Codec scope for v1 | AV1, HEVC, H.264 and VP9 — software and hardware encoders for each, where the hardware exists. `libaom-av1` and `librav1e` are excluded in favour of SVT-AV1. See §5.1.2. |
| 8 | GPU vendor scope | All three vendors. The application surveys the host and reports what is actually installed rather than assuming a vendor, which requires more than one detection backend. See §5.1.1. |
