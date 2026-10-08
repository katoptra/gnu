# gnu

The pipeline in this repository copies GNU's release tree, `ftp.gnu.org/gnu`, from a
secondary mirror into Cloudflare R2. The mirror serves the tree at `https://gnu.katoptra.org/`.
`README.md` identifies the upstream, and it tells how to use the mirror and how to fork it.
The README of [katoptra/lib](https://github.com/katoptra/lib) is the manual for the parts
that all mirrors share. This file gives the rules that each change must obey.

This repository does not start runs. An external scheduler dispatches `sync.yml` at 03:42
and 15:42 UTC. A run does `reconcile` when the interval since the last reconcile is 24 h or
more. This can be the run at 03:42 or the run at 15:42. `Taskfile.yml` contains only the
root vars and the two includes, and each verb is lib's.

## Constraints

- **Where a change goes.** This repository has no verbs. A change to the movement of bytes,
  to the pages or to the checks of `smoke` goes to lib's rsync engine. There, each rsync
  mirror gets the change. Excludes: none.
- **Root vars.** Root vars contain only the values of this mirror, and they do not give an
  engine default again
  ([lib README, Rules a mirror keeps](https://github.com/katoptra/lib#rules-a-mirror-keeps)).
  Do not set a lib var again. In an engine verb, the engine uses a root var and not a
  `KEY=value` from the command line. Thus, do not make `MAX_BATCHES`, `BATCH_GB` or
  `RECONCILE` root vars.
- **`SOURCE` must be a secondary mirror, not `ftp.gnu.org`.** GNU recommends this to each
  mirror. Before you change `SOURCE`, get a listing of the new secondary. Compare its paths
  and sizes with the listing of the primary.
- **Paths are GNU's, at the root of the bucket, with the dot-files.** Only the mirror uses
  the prefix `.state/` and the file name `gnu.katoptra.org.directory.index.html`. The GNU
  tree does not have these two names.
- **GNU's mirror guidelines are applicable to each page that this host serves.** Text must
  be "as short as possible, and strictly explanatory". Images and logos are not permitted.
  The only permitted link is a link for bug reports.
- **`PAGE_FOOT` is the only text that the mirror adds.** The pages that exist get a change
  to `PAGE_FOOT` only when the engine makes all the pages again. For this, use
  `aws s3 rm s3://gnu/.state/indexed.txt.xz`. The next run then makes all the pages again,
  with approximately 2,600 PutObjects.
- **The bug-report link is `mailto:gnu@katoptra.org`.** That address must receive e-mail.
  The mail records of the zone are not in this repository. They are in the same place as
  the zone rules.
- **Storage.** For each change that adds storage, calculate the new storage. Compare it with
  the 275 GB baseline and the 350 GB ceiling.
- **Writing.** Use ASD-STE100 and the rules in the
  [Writing section of the org CONTRIBUTING](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md#writing).
  Read that section before you write.

## Must knows

- **Symlinks become objects.** `icecat`, `windows` and `libc` are symlinks upstream. With
  `-L`, rsync gets the trees that they point to. Thus, the bucket contains these 31 GB two
  times, and clients get the same files at those URLs as from all other GNU mirrors. An
  exclude for these trees decreases the cost by $0.47 a month, but then clients get no file
  at those URLs.
- **rsync exits 23 for each listing.** Approximately 150 symlinks that point to no file cause
  it, and most of them are `back-RSN.README` files. The engine accepts exit 23. For a
  directory that it cannot read, rsync also exits 23. Then the engine deletes the files of
  that directory ([lib, The rsync engine](https://github.com/katoptra/lib#the-rsync-engine)).
  `LIST_FLOOR: 40000` stops a run if the listing decreases by more than approximately 3,800
  files.
- **The zone rules are not in this repository.** README step 4 gives them. The canary finds
  a missing rule: `smoke` reads `aspell/dict/0index.html` as `libwww-perl`.
- **No `Content-Encoding`.** GNU recommends no `Content-Encoding` header. On each run,
  `smoke` reads a `.tar.gz` and makes sure that it has no `Content-Encoding`.
- **Freshness is upstream's.** `mirror-updated-timestamp.txt` contains the epoch time of
  ftp.gnu.org, which writes it each hour. The secondary copies the file, and the mirror
  copies it from the secondary. The mirror does not write a timestamp.
- **A run failure is the only alert.** healthchecks.io: cron `42 3,15 * * *` UTC, 3 h grace.

## Verifying a change

- `task check` makes a render of each command of the pipeline in the image. Then it compares
  the render with `render.txt`. `task render-update` accepts a change. On each pull request,
  the check workflow does the same `task check`, with no secrets.
- `task run -- task list` gets a listing of the secondary, with no credentials. It is the
  one check of upstream that you can do without the vault. `.run/upstream.txt` then has
  approximately 43,850 lines, and `mirror-updated-timestamp.txt` and `gnu-keyring.gpg` are
  two of them.
- `task plan` does the read-only part of a run against the bucket of the mirror: `clock`,
  `list`, `state`, `diff` and `split`. It uses the secrets from the vault.
- The verbs of the engine, the pages and the read-back checks are lib's. To do a test of
  them, use `cd ../lib/examples/rsync && task run -- task offline`.
- Freshness: compare `curl -s https://gnu.katoptra.org/mirror-updated-timestamp.txt` with
  `date +%s`.
