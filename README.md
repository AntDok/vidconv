# vidconv

Batch video transcoding on top of ffmpeg. Replaces `convert.sh` + `remove.sh`.

Single file, Python stdlib only, no install step. Needs Python 3.11+ (for
`tomllib`) and `ffmpeg`/`ffprobe` on PATH. No GUI, no X11, no Wayland.

## Usage

```sh
./vidconv list                  # what would be converted, and where it lands
./vidconv convert               # convert everything not yet converted
./vidconv convert --select      # pick files interactively (TUI)
./vidconv convert --dry-run     # print the ffmpeg commands, do nothing
./vidconv convert -p nvenc      # use a different encode profile
./vidconv convert -m 'Season 2' # only paths matching a regex
./vidconv convert --redo        # re-convert even if the output exists
./vidconv clean                 # delete sources whose outputs verify
./vidconv profiles              # list configured profiles
```

Everything is driven by `vidconv.toml` (source dir, output dir, scan patterns,
encode profiles, deletion policy). It is read from `./vidconv.toml`, then the
script's own directory, then `$XDG_CONFIG_HOME/vidconv/config.toml`; override
with `--config`.

In the TUI: `space` toggle, `a` all, `n` none, `i` invert, `p` cycle profile,
`enter` run, `q` cancel.

## Deleting originals

`delete.policy` in the config controls this, and `--delete` overrides it per run:

- `ask` (default) — prompt before deleting each source
- `auto` — delete without prompting
- `never` — keep sources; run `vidconv clean` later

**A source is only ever deleted after its output verifies.** `delete.verify`
picks how strict that is; `decode` (the default) fully decodes the output and
compares how far the decode actually got against the source duration.

That last part matters more than it sounds. Matroska stores duration in the
segment header, so a file truncated by a crash or a full disk still *claims* its
full duration, and ffmpeg treats the early EOF as a clean end of stream rather
than an error. Checking the container's metadata — or even just "did ffmpeg exit
0" — will happily bless a half-written file. Only the decoded timestamp catches
it. Set `verify = "probe"` to skip the decode pass if you want speed and accept
that risk; `clean` will still never delete on no evidence at all.

## Notes on the old scripts

`remove.sh` deleted every mkv under an `unconverted` path with no check that a
converted output existed, let alone that it was intact. `vidconv clean` is the
direct replacement and refuses to delete anything it cannot verify.

`convert.sh` re-ran its inner `ls *mkv` loop once per file `find` returned, so a
directory with N files was walked N times. ffmpeg's `-n` made that harmless but
wasteful. Output paths now come from `strip_dirs` in the config, which reproduces
the old `../` behaviour by dropping the `unconverted` component from the path.
