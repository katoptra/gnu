<p align="center">
  <a href="https://github.com/katoptra">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://katoptra.org/brand/katoptra-mark-dark-224.png">
      <img src="https://katoptra.org/brand/katoptra-mark-224.png" alt="Katoptra" width="112">
    </picture>
  </a>
</p>

<h1 align="center">gnu</h1>

<p align="center">A twice-daily mirror of GNU's release tree, ftp.gnu.org/gnu.</p>

<p align="center">
  <a href="https://github.com/katoptra/gnu/actions/workflows/sync.yml"><img src="https://github.com/katoptra/gnu/actions/workflows/sync.yml/badge.svg" alt="sync"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/katoptra/gnu" alt="license"></a>
  <a href="https://github.com/katoptra/gnu/actions/workflows/sync.yml"><img src="https://healthchecks.io/b/2/5b1c7694-c3aa-4db7-9813-0b898669564d.svg" alt="mirror"></a>
</p>

A mirror of `https://ftp.gnu.org/gnu/` on Cloudflare R2, served at
`https://gnu.katoptra.org/`. Every path the GNU tree serves is served here at the root,
symlinked trees included: about 44,000 files and 275 GB, with a page for every directory.
Twice a day it lists a GNU secondary mirror, moves only what changed, and redraws the pages
of the directories it touched.

## How to use

Swap the host in any GNU download URL. `https://ftp.gnu.org/gnu/emacs/` becomes:

```sh
curl -s https://gnu.katoptra.org/emacs/
```

Every release sits beside its detached signature, and the GNU keyring is at the root.
Verify a tarball with:

```sh
curl -sO https://gnu.katoptra.org/gnu-keyring.gpg
gpg --verify --keyring ./gnu-keyring.gpg hello-<version>.tar.gz.sig hello-<version>.tar.gz
```

Is it fresh? ftp.gnu.org writes the time as epoch seconds to
`mirror-updated-timestamp.txt` every hour, and the mirror copies it like any other file:

```sh
echo $(( $(date +%s) - $(curl -s https://gnu.katoptra.org/mirror-updated-timestamp.txt) )) seconds behind ftp.gnu.org
```

## How it works

Twice a day a GitHub Actions job runs this pipeline inside the toolbox image from
[katoptra/lib](https://github.com/katoptra/lib). Every box is a verb of lib's rsync engine;
this mirror adds none of its own.

```mermaid
flowchart LR
  clock --> list --> state --> rebuild --> diff --> split --> batches
  subgraph b["batches: the first MAX_BATCHES of the delta, each committed before the next"]
    direction LR
    fetch --> publish --> checkpoint
  end
  batches --> b --> delete --> reconcile --> index --> smoke --> report --> ping
```

What this mirror sets, in [`Taskfile.yml`](Taskfile.yml):

- **Its identity**: `SOURCE`, which is a secondary mirror (`mirror.csclub.uwaterloo.ca`)
  because GNU asks mirrors to spare `ftp.gnu.org`; `HOST`, `BUCKET`, a 350 GB `CEILING_GB`
  past which a run refuses to start, and a `LIST_FLOOR` under which a listing is taken as
  truncated rather than as a deletion list.
- **Directory pages.** `INDEX` turns on the engine's pages: one per directory, drawn from
  the state and served for `/dir/` and `/dir` alike. `PAGE_FOOT` is the line every page
  closes on: what the mirror is, how often it updates, and where to report a problem.
- **The canary.** `CANARY` is `aspell/dict/0index.html`, whose plain `http://` links any
  of Cloudflare's HTML rewriters would alter. After every run the engine reads it through
  the domain as a Perl client and compares it byte for byte with the bucket's copy.

Everything else, from the list diff and the batching to the state file and the daily
reconcile, is documented once in [lib's README](https://github.com/katoptra/lib#the-rsync-engine).

## Want your own?

### 1. Fork it

Fork [katoptra/gnu](https://github.com/katoptra/gnu). `HOST`, `BUCKET` and the address in
`PAGE_FOOT` are yours to change. `SOURCE` may be any secondary on
[GNU's mirror list](https://www.gnu.org/prep/ftp.html) that carries the full tree; compare
its listing with the primary's once before the first run.

### 2. Storage

| What | Why |
|---|---|
| An R2 bucket, or any S3-compatible bucket | The tree, its pages and their state: about 275 GB, $4.12 a month on R2 |
| An API token with Object Read & Write, scoped to that bucket | The three `AWS_*` values in step 3 |
| A custom domain on the bucket, which is `HOST` | What clients and the read-back checks fetch from |

R2 has no symlinks, so the engine stores each path a symlink makes reachable as a real
object. That doubles `/icecat/`, `/windows/` and `/libc/`, 31 GB between them, so that they
answer as on every other GNU mirror. `aws.config` keeps every file under 4 GiB a single
PutObject; the largest is 1.51 GB. [lib, Storage](https://github.com/katoptra/lib#storage).

### 3. Secrets

Four values, in one vault item named `gnu`:

| Section | Field | What it is | Reaches the run as |
|---|---|---|---|
| `r2` | `access_key_id` | The token from step 2 | `AWS_ACCESS_KEY_ID` |
| `r2` | `secret_access_key` | Its secret | `AWS_SECRET_ACCESS_KEY` |
| `r2` | `endpoint` | `https://<account-id>.r2.cloudflarestorage.com` | `AWS_ENDPOINT_URL` |
| `healthcheck` | `url` | Optional: a healthchecks.io ping URL | `HEALTHCHECK_URL` |

Put your vault's UUID into the four references in [`op.env`](op.env), make a service
account that can read that vault, and store its token as the `OP_SERVICE_ACCOUNT_TOKEN`
secret. [lib, Secrets](https://github.com/katoptra/lib#secrets) has the details.

### 4. The zone

Three rules, each scoped to the mirror's hostname:

| Rule | What it does |
|---|---|
| Configuration Rule | Email Obfuscation, Rocket Loader, Automatic HTTPS Rewrites and Browser Integrity Check off. The first three rewrite HTML in flight. The fourth answers 403 to Perl and Python clients. The canary fails if any comes back on |
| Cache Rule | Bypass. An edge copy would serve stale pages and a stale timestamp |
| Transform Rule | A path ending in `/` rewrites to `concat(http.request.uri.path, http.host, ".directory.index.html")`, the root included |

### 5. Prove it, run it, schedule it

On a laptop with go-task, the 1Password CLI and Docker or Apple `container`:

```sh
task check                # render every command of the pipeline inside the image; diff against render.txt
task run -- task list     # list the secondary, no credentials; read .run/upstream.txt
```

Then Actions, sync, Run workflow. The first run finds an empty bucket, takes the whole
tree as the delta and works it four batches at a time, queueing the next run itself until
nothing is left: about 275 GB over roughly eighteen chained runs. Every run after that
moves the delta, usually a handful of files.

Nothing in this repository schedules a run. Add a `schedule:` trigger to
`.github/workflows/sync.yml` with a time of your own, or dispatch it from outside as this
mirror is. Whatever the time, a run also sweeps the bucket against the state once the last
sweep is 24 h old.

## Operating it

`task` alone prints the menu. A run takes its flags after the double dash, and the same
flags go in the workflow's `vars` input:

```sh
task sync                                   # one run, the same thing Actions runs
task sync -- RECONCILE=true                 # rebuild the state from the bucket and sweep orphans now
gh workflow run sync.yml -f vars='RECONCILE=true'
```

Every run appends one table to its job page: the delta and what landed, the state,
storage against the ceiling, and the directory pages drawn. A failed run is the only
alert: healthchecks.io emails when a slot passes without a ping, which also catches a
scheduler that stopped.

- **`split` refused the tree.** Upstream passed 350 GB. The mirror stays stale until
  `CEILING_GB` is raised, which also raises the bill.
- **The listing was refused.** Under `LIST_FLOOR` lines is a truncated listing, never a
  deletion list. Re-run; if it persists, the secondary is the place to look, and another
  from GNU's list can replace it.
- **The canary failed.** The domain served bytes the bucket does not hold, or refused a
  Perl client: a zone rule is missing or was widened. The bucket is intact.
- **A `.tar.gz` came back with a `Content-Encoding`.** Clients would unpack it in flight.
  Look at the AWS CLI version in lib's lock.
- **The run did not start.** Check the scheduler, then `gh workflow view sync.yml` for a
  manual disable, and dispatch by hand meanwhile.

## Reference

Every listing exits 23. About 150 symlinks in the tree point at nothing, and rsync skips
each with a line on stderr and lists the rest. GNU's mirror page says errors of this class
are to be ignored, and the engine passes 23. The same code covers a directory rsync cannot
read, whose files the engine then treats as gone and deletes until the secondary serves
them again. A loss large enough to take the listing under `LIST_FLOOR` stops the run
instead.

The tree's root carries dot-files, `.header.shtml` and `.message`, mirrored like any file.
`.state/` is the one prefix the mirror reserves, and GNU has none.

Pull requests are welcome.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
