# Working in this repository (Base44 sandbox notes)

## What this repository is

A document collection, not an application. There is no application code, no
package manifest, no test suite and no service dependency to install:

- `preprints/` — one directory per manuscript family (750 entries). Each holds a
  committed PDF (`paper.pdf`, `article.pdf` or `final.pdf` — the name varies per
  family), a `README.md` with the citation block, and `build/source/*.tex` with
  the LaTeX sources.
- `lean/` — one large Lean 4 library (`lakefile.lean`, `lean-toolchain`,
  `formalization.yaml`, `docs/*.md` per family) with `ComparatorChallenges/`.
- `reasoning_traces/` — abridged reasoning-summary PDFs.
- `CONTENTS.md` (638,727 bytes) — the manuscript map; `overview.pdf` — the family
  overview.

There is nothing to compile or migrate, so "running" the project in the sandbox
means exposing the checkout read-only over HTTP: `docker-compose.base44.yml`
starts a single `nginx:alpine` service that serves the checkout on host port
3000, which is what the preview shows.

## Why the branch needed this runbook

`main` at `1581d4cbdd` ("Automatically lock new PR conversations") ships no
`docker-compose.base44.yml` and no `.base44/environment.json`, so no process
listened on port 3000 and the preview never left its loading screen. The
collection itself was never the problem — there is no build step to fail. The
runbook files here are the fix; without them a fresh import of this repo cannot
start.

## How the service is wired (and the two traps it avoids)

- Served from `root /srv/collection`, not the image's `/usr/share/nginx/html`:
  that directory ships an `index.html`, which would make `/` render nginx's
  welcome page instead of the collection listing. `/srv/collection` does not
  exist in the image, so `/` returns a directory listing.
- The top-level entries are bind-mounted **individually** (see `volumes:`)
  because the checkout root is mode `0700`: nginx drops to an unprivileged user
  and cannot traverse a mount of the whole tree, while every entry below the root
  is world-readable. Mounting them individually also keeps `.git` and `.github`
  out of the served tree. A **new top-level file therefore needs a new `volumes:`
  entry** to become browsable.

## How to verify the setup

```sh
docker compose -f docker-compose.base44.yml ps                 # collection = healthy
curl -s  http://localhost:3000/ | grep -c 'href='              # 10 root entries
curl -s  http://localhost:3000/preprints/ | grep -c 'href='    # 750 families + ../ + build/
curl -sI http://localhost:3000/overview.pdf | head -1          # 200 application/pdf
curl -s  http://localhost:3000/CONTENTS.md | md5sum            # == md5sum CONTENTS.md
curl -s  http://localhost:3000/.git/config                     # 403 (denied)
```

Directory listings are on purpose (the collection is navigated by folder); `.md`
is served as `text/plain` so `CONTENTS.md` and the per-family READMEs read inline;
dotfiles are denied. Note that a `fetch()` of a text file counts characters, not
bytes (multi-byte math symbols), so compare sizes with `curl | wc -c` or `md5sum`.

The preview's route opener cannot reach a static path, so verify non-root pages by
clicking through the listing or by fetching them.

## Quirks worth knowing

- The checkout is several GB (the Lean sources dominate). Do not copy the tree
  around to inspect it.
- Building the Lean library is separate, very heavy (`cd lean && lake update &&
  lake exe cache get && lake build`) and unrelated to the preview; `lean/README.md`
  warns the whole-library build can fail when `vm.max_map_count` is low.
  Comparator verification needs third-party binaries (`comparator`, `landrun`,
  `lean4export`) on `PATH`; see `lean/ComparatorChallenges/README.md`.
- Manuscript PDFs are committed, so serving them needs no LaTeX toolchain.
- The sandbox flag `BASE44_PREVIEW_MODE` is not used here: static files, no
  host/origin allowlist needed.
