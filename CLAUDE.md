# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What this is

`ister-app`'s fork of `media-kit/libmpv-win32-video-cmake`, itself a fork of
`shinchiro/mpv-winbuild-cmake`: it cross-compiles mpv and libmpv for Windows (x86_64, x86_64-v3,
i686, aarch64) from Linux with a self-built LLVM/MinGW toolchain.
`ister-app/media-kit`'s `libs/windows/media_kit_libs_windows_video` downloads the `mpv-dev-*.7z`
of a release by URL + md5. Only x86_64 and aarch64 are ever used; the i686 archive is dead weight.

Merge from `shinchiro/mpv-winbuild-cmake` when things look stale — this fork drifted a year behind
and that is how a `GREATER_EQUAL` version check on meson silently broke every build.

## Workflows (both `workflow_dispatch` only)

1. **llvm toolchain** — builds LLVM/clang and the four cross toolchains into Actions caches. Hours.
   A fresh fork has no caches, so this must run first.
2. **mpv clang** — builds mpv against those caches. Inputs: `command` (leave empty),
   `github_release` (**true** to publish), `mpv_tarball` (**false**, or it ignores the pinned
   `GIT_TAG` and builds the latest mpv tarball instead).

Versions are pinned per package in `packages/*.cmake` (`GIT_TAG`).

## Things that cost time to find out

- **The ffmpeg configuration is an allowlist** (`--disable-demuxers`, `--disable-decoders`,
  `--disable-parsers`, `--disable-protocols`). Inherited from media-kit's *audio* build, it had no
  `hls` and no `mpegts` demuxer, so the player's HLS streams could not be read at all: the master
  playlist probes as "No format found", mpv falls back to its own playlist parser and the `.ts`
  segments then fail too. Video *decoders* are fine — they come in as dependencies of
  `--enable-hwaccels`, which is why `--vd=help` lists h264/hevc/vp9 despite the audio-only
  `--enable-decoder=` list. Anything the player needs must be listed explicitly.
- **`--enable-small` strips ffmpeg's long names**, so grepping a binary for
  "Apple HTTP Live Streaming" proves nothing. Test behaviour instead (see below).
- **ffmpeg 9 removed libpostproc**; `--disable-postproc` is an unknown option now.
- **The release step used to lie.** It POSTed to `media-kit/libmpv-win32-**audio**-cmake` with
  `curl -s` and `continue-on-error`, so the job went green while uploading nothing. It now targets
  `$GITHUB_REPOSITORY`, reuses an existing release for the day, replaces same-named assets and uses
  `--fail-with-body`.

## Testing without a Windows machine

Run the built `mpv.exe`/`mpv.com` under Proton's wine against a real stream — this is how the
missing demuxers were found and the fix confirmed:

```bash
export WINEPREFIX=/tmp/wp WINEDEBUG=-all
P=~/.steam/steam/steamapps/common/"Proton - Experimental"/files
"$P/bin/wine" mpv.com --no-config --vo=null --ao=null --untimed --frames=150 \
  --msg-level=all=v --demuxer-lavf-o=allowed_extensions=ALL,extension_picky=0 "<hls url>"
```

A working build reports `Detected file format: hls (libavformat)` followed by `VO: [null] …`.
A stream URL with a valid token can be lifted from an Android device's logcat while the player
runs with `MPVLogLevel.v`.
