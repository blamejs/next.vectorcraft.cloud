# next.vectorcraft.cloud

https://next.vectorcraft.cloud/ serves a web build of the `main` branch of https://github.com/storytold/vectorcraft.

- `src/index.js` is the Cloudflare Worker `next-vectorcraft-cloud`. It serves every build in the R2 bucket
  `web-app-next` under its own path, `/b/<id>/`, and redirects `/` (and any other path) to the current build's path.
  A page and the files it loads therefore always come from one build. It sends each `.wasm` file Brotli-compressed to
  browsers that accept `br`, and adds an `X-Build` header naming the build. The bucket layout is described at the top
  of the file.
- `.github/workflows/publish.yml` runs at 00:23 and 12:23 UTC. It builds only when the head of `main` differs from the
  served build, or the served build has not passed its check, or when it is run by hand with `force` set. It has
  three jobs:
  - `compare` reads the head of `main` and the bucket's `STATE`.
  - `build` checks out VectorCraft at that commit and builds the web zip with `packaging/web/package.sh`, using the
    craft-fonts commit pinned in VectorCraft's `release.yml`. This job has no Cloudflare secrets. It uploads the zip
    as the workflow artifact `web-zip`.
  - `publish` downloads the zip and uploads it with `scripts/publish.sh` as a new build with the id
    `<first 12 characters of the commit>.<UTC time>`. It then checks the site with `scripts/check.sh` and records the
    result with `scripts/mark-checked.sh`. When the check fails, the last build that passed its check is served again
    (`scripts/rollback.sh`). A commit that failed its check is not built again until `main` moves to another
    commit or the workflow is run by hand with `force` set. `scripts/cleanup.sh` keeps the last 3 builds and
    deletes older ones. When the run's commit is not newer on `main` than the published build, the job does not
    publish it.
- Run the workflow by hand from the Actions tab. With `force` set, it builds and publishes the head of `main` even
  when that commit is already current or failed its check before.
- `scripts/current.sh web-app-next` prints `STATE`: the served build (`current=`), the build served before it
  (`previous=`, kept by cleanup), the last build that passed its check (`checked=`) and the last build that failed
  it (`failed=`).
- The scripts read `STATE` and write it back whole. Do not run them by hand while the workflow is running.

## Setup

The repository needs two secrets:

- `CLOUDFLARE_API_TOKEN`: an API token with the account permission Workers R2 Storage: Edit.
- `CLOUDFLARE_ACCOUNT_ID`: the account ID.

The Worker is deployed by hand: `npm ci`, `npx wrangler login`, then `npx wrangler deploy`. The bucket was created
with `npx wrangler r2 bucket create web-app-next`.

## Checks

- `npm test` runs the Worker's tests.
- `scripts/check.sh https://next.vectorcraft.cloud/ <id>` checks the live site, where `<id>` is the `current=` value
  that `scripts/current.sh web-app-next` prints.
