# Custom fork synchronization

The `custom-sync.yml` workflow merges `liketrek/TREK` **main** into
`codex/custom-beian-footer`. It runs daily at 04:20 Asia/Shanghai, on custom
branch pushes, and manually from Actions. It never resets the custom branch,
increments upstream versions, or deploys a server.

## Initial setup

1. Commit and push your custom changes and this workflow to the custom branch.
   Review the existing package-lock.json changes before including them.
2. Also add the workflow file to your fork's default branch (`main`). GitHub
   scheduled workflows run only from the default branch. Do not copy the
   application customizations to main; keep those on the custom branch.
3. Enable Actions in the fork. In Settings → Actions → General, allow GitHub
   Actions to create pull requests, and ensure repository policy permits the
   workflow's contents, packages and pull-request write permissions.
4. Run **Sync upstream and publish custom image** manually once and inspect its
   result. No personal access token is needed: it uses the repository GITHUB_TOKEN.

## Results and failures

On a clean merge, the workflow runs workspace typechecks/tests, plugin SDK
checks, an AMD64 Docker build and a temporary-container health check. Only then
does it push the merged custom branch and publish the image. Concurrent remote
changes cause a normal push rejection; they are never overwritten. A failed run
can be retried manually. GitHub Actions notifications report failed runs when
enabled in your account.

Conflicts create a PR from a `sync/upstream-<sha>` branch into the custom branch.
Resolve the conflicts and merge that PR. That push triggers validation again.
If PR creation is disallowed, the run fails with the GitHub error; enable the
setting above and rerun. Failed validation does not update the custom branch or
the image. A publication failure after the branch update can be rerun manually.

Bot pushes using GITHUB_TOKEN do not trigger another push workflow; validation
and image publication are deliberately in this same workflow.

## Version and deployment

The root, client, server and shared package versions must agree. Their upstream
version becomes APP_VERSION at build time. Image tags are:

- `ghcr.io/mantra50/trek:<version>-custom-<commit>` (traceable build)
- `ghcr.io/mantra50/trek:custom-latest` (latest validated custom build)

Use the image in your existing Compose file. Remove any fixed APP_VERSION
environment override so the image version takes effect. Keep your existing
encryption key, data/uploads mounts, and other runtime configuration.

```yaml
services:
  app:
    image: ghcr.io/mantra50/trek:custom-latest
```

New GHCR packages may be private. For anonymous server pulls, set the package's
visibility to public; otherwise authenticate the server with a read:packages
credential. Repository-owned GITHUB_TOKEN must have access if the package already
exists and belongs to another repository.

Back up data, uploads and encryption credentials before upgrading. Then run:

```sh
docker compose pull app
docker compose up -d app
```

Deployments remain manual. For rollback, use a known commit image tag and, if
database migrations require it, restore the matching backup. GitHub may disable
scheduled workflows after extended repository inactivity; check Actions if daily
sync stops. This workflow builds linux/amd64 only.
