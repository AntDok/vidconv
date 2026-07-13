# Implementation notes

Working notes for whoever maintains `vidconv` next (probably me, having forgotten
all of this). `README.md` covers usage; this covers *why*, and what bit me.

## Why not an off-the-shelf tool

The request was for something off-the-shelf. I couldn't find an honest match, and
that conclusion is worth recording so nobody re-runs the search:

- **Tdarr / Unmanic / FileFlows** — the mature options in this niche. All are
  server + **web UI** apps. They'd technically satisfy "no X11/Wayland" but you'd
  manage your queue in a browser, which is not a shell tool. All want Docker.
- **`ab-av1`, `ffmpeg-normalize`** — CLI-native but single-purpose, not batch
  pipeline managers.
- **Assorted GitHub transcode scripts** — functionally the same clunky loop as the
  `convert.sh` we're replacing.

There is no well-maintained, config-driven, CLI/TUI batch transcoder. Combined
with the project rule of *no package installs* (`CLAUDE.md`), that ruled out
everything anyway. Hence: one file, Python stdlib only.

## Why Python stdlib specifically

The environment gives us Python 3.12, which means `tomllib` (config parsing) and
`curses` (TUI) are **already there**. Zero dependencies, zero install, satisfies
the no-install rule, and it's far easier to keep correct than the equivalent bash
(quoting, `IFS`, filenames with spaces — the old scripts got several of these
wrong). Requires 3.11+ strictly, for `tomllib`.

## The bug that shaped the design — read this one

**A truncated Matroska file lies about its duration, and ffmpeg won't tell you.**

I sabotaged an output by truncating it to half its bytes. My first verification
pass *approved it for deletion*. Three separate checks all failed to catch it:

1. `dst.exists()` — passes, the file is there.
2. `ffprobe` duration vs source duration — passes. Matroska stores duration in the
   **segment header**, so a file chopped in half still claims the full runtime.
3. Full decode, check exit code — **passes**. ffmpeg treats an early EOF as a clean
   end-of-stream, not an error. Exit 0. Even with `-xerror`.

The only thing that catches it is measuring how far the decode *actually got*:

```
ffmpeg -v error -i OUT -f null -progress pipe:1 -nostats -
```

then parsing the last `out_time_us` and comparing to the source duration. That's
what `decode_duration()` does. It correctly reports:

```
output is short: decoded 1.3s of an expected 3.0s (truncated or corrupt)
```

**Do not "simplify" this back to a returncode check.** It looks redundant and it
is not. This is the exact failure mode the old `remove.sh` was blind to — a crash
or full disk mid-encode meant a silently deleted source. Deleting the user's only
copy of something is the one unrecoverable action this tool can take, so the
verification is deliberately paranoid and deliberately the slow path.

`verify = "probe"` exists as an escape hatch for people who want speed and accept
the risk. `clean` refuses to drop below `probe` even if `verify = "none"`, because
deleting on *zero* evidence is indefensible.

## Two checks, not one (and never two decodes)

Originally there was a single `verify` setting used both after each encode and
again before deleting — which meant a `convert --delete auto` **decoded every
output twice**. Measured: 4 decode passes for 2 files. Now:

- `[convert] verify` (default `probe`) — the post-encode check. The one that
  costs wall-clock time. Cheap by default.
- `[delete] verify` (default `decode`) — required before a source is deleted.
  Safety, not speed.

`VERIFY_RANK` orders them, so a check already passed at a rank ≥ what deletion
needs is trusted rather than re-run. And if a run deletes, `cmd_convert`
escalates the post-encode check up to the deletion requirement. Net effect:
**exactly one decode pass, and never zero when a source is about to die.**
`--verify none --delete auto` still decodes. That escalation is the invariant to
preserve if you touch this.

## ffmpeg exit 0 does not mean it did anything

`--redo` used to be a **silent no-op that reported success**. With `-n` (the
default, `overwrite = false`), ffmpeg sees the existing output, prints "already
exists", and **exits 0**. So `convert_one` concluded success, verified the *stale*
output, passed, and reported "2 converted". With `--delete auto` it then deleted
the source having re-encoded nothing.

Two fixes, keep both:

1. `--redo` now implies `-y`. Re-encoding is impossible under `-n`, so any code
   path meaning to re-encode must force overwrite.
2. `convert_one` checks the output's mtime advanced past the moment ffmpeg
   started. This is the general guard: exit 0 with no new bytes written is
   treated as failure regardless of cause. Don't remove it because the `-y` fix
   "already handles it" — that fix covers the one case we found, this covers the
   ones we didn't.

## Other gotchas hit during testing

- **curses crashes on the bottom-right cell.** Writing a character to the last
  column of the last row wraps the cursor off-screen and raises
  `_curses.error: addnwstr() returned ERR`. Every write goes through `put()`, which
  caps at `w-1` and swallows `curses.error`. This is why line widths look
  off-by-one; it's load-bearing.
- **x265 ignores ffmpeg's `-loglevel`.** It writes its own banner and per-frame
  stats straight to stderr. `-x265-params log-level=error` in the profile args is
  what actually silences it. Hence that flag in the x265 profiles.
- **Partial outputs are deleted on ffmpeg failure.** Otherwise the next run's
  "output already exists, skip" logic treats the corpse as a finished job, and
  `clean` sees a plausible-looking file. Failure must leave no trace.

## Design decisions worth keeping

- **Output paths are derived, not tracked.** `output_path_for()` maps source →
  output deterministically, so there's no state file / database to corrupt or get
  out of sync. "Already converted?" is just `dst.exists()`. This is why `strip_dirs`
  exists: dropping the `unconverted` component reproduces the old script's `../`
  behaviour while keeping the mapping pure.
- **`-n` not `-y` by default.** Re-running is safe and idempotent; it skips
  finished work rather than redoing it. `--redo` is the explicit override.
- **Deletion is a separate concern from conversion.** `convert --delete` and the
  standalone `clean` command share `verify_output()`. You can always convert now
  and delete later, which is what `policy = "never"` + `clean` is for.
- **Profiles are just ffmpeg arg strings**, `shlex.split()` on load, spliced
  between input and output. No abstraction over ffmpeg — you can paste any ffmpeg
  invocation you already trust into the config and it works.

## How I tested it

No test framework (no installs), so: generate real video with
`ffmpeg -f lavfi -i testsrc + sine`, build a fake source tree including a filename
with spaces, and exercise the real binary. Worth redoing after changes:

- convert → check output is actually `hevc` via ffprobe, sources survive
- re-run → must skip, not re-encode
- sabotage outputs three ways (**truncate**, empty, delete) → `clean --dry-run`
  must refuse all three
- `--delete auto` → sources gone, outputs intact
- bogus encoder in a profile → exit 1, no partial file, source preserved
- exit codes: 0 ok / 1 failure / 2 config error
- TUI: drive it through a `pty.fork()` and feed keys, since a curses crash only
  shows up at runtime. This is how the bottom-right-cell bug was found.

## Known gaps / if you extend this

- **Serial only.** One ffmpeg at a time. Fine for x265 (it saturates all cores
  anyway) but wasteful for NVENC. A `--jobs N` worker pool would be the obvious
  addition.
- **No resume within a file.** Interrupting mid-encode discards that file's work.
- **Verification doubles I/O** on the `decode` path — it decodes each output once
  more. Accepted cost; see above.
- **VAAPI profile drops subtitles.** The `hwupload` filter can't carry them
  through. Noted in the config comment.
- **`min_depth`** exists only to mirror the old `find -mindepth 3`. Probably nobody
  needs it.

## Appendix: the original scripts

Deleted 2026-07-12, once `vidconv` replaced them. This directory isn't under
version control, so they're recorded here verbatim — purely so the behaviour being
replaced stays legible. Neither should be run again; `remove.sh` in particular is
the unsafe deletion described above.

`convert.sh`:

```bash
#!/bin/bash
cwd=$(pwd -LP)
IFS=$'\n' 
for path in `find . -mindepth 3 -iname  "*mkv" | grep unconverted`
do 
	cd $(dirname "$path")
	(IFS=$'\n'; for file in `ls -r *mkv`; do time nice -n10 ffmpeg -n -hide_banner -i ${file} -map 0 -vcodec libx265 -crf 24 ../${file}; done)
	cd $cwd	
done
```

`remove.sh`:

```bash
#!/bin/bash
find . -mindepth 3 -ipath  "*unconverted*mkv" -print -delete
```

The settings worth preserving from these — `nice -n10`, `-n`, `-map 0 -c:v libx265
-crf 24`, the `unconverted` → parent-directory output mapping — are all carried
over in `vidconv.toml`.
