# ffmpeg4apple

Fork of [FFmpeg/FFmpeg](https://github.com/FFmpeg/FFmpeg) that automatically publishes
static Apple Silicon (arm64) macOS builds of `ffmpeg`, `ffprobe` and `ffplay` for every
upstream release tag from `n8.1` onward.

## How it works

Two workflows (this directory), which live only on `master` — upstream tags don't
contain them, so tag pushes can never trigger a workflow directly; the sync workflow
dispatches builds explicitly instead.

### `sync.yml` — every 6 hours (and on demand)

1. Fetches `master` and all `n*` tags from upstream and pushes them to this repo
   (merging upstream `master` into ours; a conflict is warned about, not fatal).
2. For every tag matching `nX.Y[.Z]` with version >= 8.1 that has **no GitHub release
   yet** and no build already queued/running, dispatches `build.yml` with that tag.

### `build.yml` — workflow_dispatch with a `tag` input

1. `preflight` (ubuntu): validates the tag format/floor and exits quietly if the
   release already exists.
2. `build` (`macos-26`, arm64): checks out the tag, clones
   [build-script](https://git.martin-riedl.de/ffmpeg/build-script) (`main` branch),
   and patches one line of `script/build-ffmpeg.sh` so it consumes a tarball produced
   from this checkout via `git archive` (with a `VERSION` file injected, mirroring
   official release tarballs) instead of downloading from ffmpeg.org.
3. Runs the full static build (x264, x265 multibit, aom, svt-av1, rav1e, dav1d, vvenc,
   vpx, opus, lame, srt, libass, vmaf, ...) including the script's post-build tests.
4. Publishes a GitHub release tagged `X.Y.Z` (no `n` prefix, pinned to the source
   tag's commit) with `ffmpeg-X.Y.Z-macos-arm64`, `ffprobe-...`, `ffplay-...`, a
   `.tar.gz` bundle (preserves the executable bit) and `SHA256SUMS.txt`.

Only prerequisite installed on the runner is `cargo-c` (for rav1e); the build script
compiles its own cmake/nasm/ninja/pkg-config and all libraries statically.

## Operational notes

- A full build takes a few hours on the hosted runner; job timeout is 6 h.
- If this repo was created with GitHub's fork button, enable workflows once in the
  Actions tab (scheduled workflows are off by default on forks).
- Releases are keyed by version: to rebuild one, delete its release (and the `X.Y.Z`
  tag it created) and re-run `Build` (or wait for the next sync).
- Building a specific tag manually: Actions → Build → Run workflow → enter e.g. `n8.1.2`.
