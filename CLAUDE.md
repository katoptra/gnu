# gnu

A mirror of GNU's release tree, `ftp.gnu.org/gnu`, on Cloudflare R2, served at
`https://gnu.katoptra.org/`, pulled from a secondary. `README.md` says what it mirrors, how
to use it and how to fork it; [katoptra/lib](https://github.com/katoptra/lib)'s README is
the manual for everything the mirrors share. This file is what a change must not break.

Nothing in this repo starts a run: an external scheduler dispatches `sync.yml` at 03:42
and 15:42 UTC; `reconcile` runs once the last one is 24 h old, whichever run that is.
`Taskfile.yml` holds root vars and the two includes and nothing else; every verb is lib's.

## Constraints

- No verbs of its own. A change to how bytes move, how pages are drawn or what `smoke`
  reads goes to lib's engine, where ctan and nongnu get it too. The one exclude is
  `report-engine`, on the toolbox include. Never redefine a lib var. Inside an engine verb a
  root var shadows a command-line `KEY=value`, so `MAX_BATCHES`, `BATCH_GB` and `RECONCILE`
  stay out of the root vars.
- `SOURCE` is a secondary, never `ftp.gnu.org`: GNU asks it of every mirror. Changing it is
  a listing compared with the primary's by path and size first.
- Paths are GNU's, at the bucket root, dot-files included. `.state/` is the reserved prefix
  and `gnu.katoptra.org.directory.index.html` the reserved file name; GNU has neither.
- GNU's mirror guidelines govern every page this host serves: text short and strictly
  explanatory, no images or logos, no link but a bug-reporting one. `PAGE_FOOT` is the only
  text the mirror adds. A change to it reaches existing pages only through a full redraw:
  `aws s3 rm s3://gnu/.state/indexed.txt.xz`, about 2,600 PutObjects on the next run.
- The bug-reporting link is `mailto:gnu@katoptra.org`, and that address must take mail.
  The zone's mail records live outside this repository, beside its rules.
- Recompute any change that adds storage against the 275 GB baseline and the 350 GB ceiling.

## Must knows

- **Symlinks become objects.** `icecat`, `windows` and `libc` are symlinks upstream; `-L`
  stores what they reach, 31 GB twice over, so those URLs work as on every GNU mirror.
  Excluding them would save $0.47 a month and break them.
- **Every listing exits 23**, from about 150 dangling `back-RSN.README` links and others.
  The engine passes it. The same code means an unreadable directory, whose files the engine
  then deletes as gone; `LIST_FLOOR: 40000` stops a loss of more than about 3,800.
- **The zone rules live outside this repository**; README step 4 lists them. The canary,
  `aspell/dict/0index.html` read as `libwww-perl`, is what notices one missing.
- **No `Content-Encoding`.** GNU asks for none; `smoke` asserts it on a `.tar.gz` every run.
- **Freshness is upstream's.** `mirror-updated-timestamp.txt` is ftp.gnu.org's hourly epoch,
  carried through the secondary; the mirror writes no timestamp of its own.
- **A failed run is the only alert.** healthchecks.io cron `42 3,15 * * *` UTC, 3 h grace.

## Verifying a change

- `task check` renders every command of the pipeline inside the image and diffs it against
  `render.txt`; `task render-update` accepts a change.
- `task run -- task list` lists the secondary with no credentials: about 43,850 lines in
  `.run/upstream.txt`, `mirror-updated-timestamp.txt` and `gnu-keyring.gpg` among them.
- The engine's verbs, the pages and the read-back checks are lib's:
  `cd ../lib/examples/rsync && task run -- task offline`.
- Is the mirror fresh? `curl -s https://gnu.katoptra.org/mirror-updated-timestamp.txt`,
  against `date +%s`.
