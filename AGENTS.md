# Working in this repository (Base44 sandbox notes)

## What this repository is

A document collection, not an application. There is no application code, no
package manifest, no test suite and no service dependency:

- `preprints/` — one directory per manuscript family (749 of them). Each holds a
  committed PDF (`paper.pdf`, `article.pdf` or `final.pdf` — the name varies per
  family), a `README.md` with the citation block, and `build/source/*.tex` with
  the LaTeX sources (plus a per-family build instruction where present).
- `lean/` — one large Lean 4 library (`lakefile.lean`, `lean-toolchain`,
  `formalization.yaml`, `docs/*.md` per family) with `ComparatorChallenges/`.
- `reasoning_traces/` — abridged reasoning-summary PDFs.
- `CONTENTS.md` (638 KB) — the manuscript map; `overview.pdf` — family overview.

Because of that, there is nothing to compile or migrate to see the project:
`docker-compose.base44.yml` runs a single `nginx:alpine` service that serves the
checkout read-only on host port 3000, which is what the preview shows.

## How the service is wired (and the two traps it avoids)

- Served from `root /srv/collection`, not the image's `/usr/share/nginx/html`:
  that directory ships an `index.html`, which would make `/` render nginx's
  welcome page instead of the collection listing. `/srv/collection` does not
  exist in the image, so a request for `/` gets a directory listing.
- The top-level entries are bind-mounted individually (see the `volumes:` list)
  because the checkout's root directory is mode `0700`: nginx drops to an
  unprivileged user and cannot traverse a mount of the whole tree, while every
  entry below that root is world-readable. Mounting them individually also keeps
  `.git`/`.github` out of the served tree. A *new* top-level file therefore needs
  a new `volumes:` entry to be browsable in the preview.

## How to verify the setup

```sh
docker compose -f docker-compose.base44.yml ps             # collection = healthy
curl -s  http://localhost:3000/ | grep -c preprints        # collection listing on /
curl -s  http://localhost:3000/preprints/ | grep -c '/"'   # ~750 families listed
curl -sI http://localhost:3000/overview.pdf | head -1      # 200 application/pdf
curl -s  http://localhost:3000/CONTENTS.md | md5sum        # matches the checkout
```

Directory listings are enabled on purpose (the collection is navigated by
folder); `.md` files are served as `text/plain` so the catalogue and the
per-family READMEs read inline; dotfiles are denied. The preview's route opener
uses SPA navigation, which cannot reach a static path: verify non-root pages by
fetching them or by clicking through in the preview, not with
`preview_screenshot(path=...)` (it opens `/` only).

## Quirks worth knowing

- The checkout is ~3.2 GB (Lean sources dominate). Full clones/copies are slow;
  do not copy the tree around to inspect it.
- Building the Lean library is a separate, very heavy operation
  (`cd lean && lake update && lake exe cache get && lake build`) that needs the
  elan toolchain and hours of compute; it is unrelated to the preview and is not
  part of the sandbox setup. `lean/README.md` warns that compiling the whole
  library can fail when `vm.max_map_count` is low.
- Comparator verification needs third-party binaries (`comparator`, `landrun`,
  `lean4export`) on `PATH`; see `lean/ComparatorChallenges/README.md`.
- Manuscript PDFs are committed, so serving them needs no LaTeX toolchain.
- The sandbox flag `BASE44_PREVIEW_MODE` is not used here: the service serves
  static files and needs no host/origin allowlist entries.
