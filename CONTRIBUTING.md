# Contributing

Thanks for your interest in extending `docker-harnesses` with a new image. The repo follows a small, repeatable layout — once you've seen one image, you've seen them all.

## Repository layout

Each image has three files:

```
.
├── <name>/                # e.g. ai-harness/
│   ├── entrypoint         # bash script: runs init, then exec <tool>
│   └── init               # default no-op bash script (overridable at runtime)
├── Dockerfile.<name>      # e.g. Dockerfile.ai-harness
└── .github/workflows/<name>.yml
```

## Adding a new image

1. **Create the folder** at the repo root: `<name>/`.

2. **Add `entrypoint`.** Runs `bash /usr/local/bin/init`, then `exec`s the tool. Example (`ai-harness/entrypoint`):

   ```bash
   #!/bin/bash

   set -e

   # Run user-customizable init script
   bash /usr/local/bin/init

   # Start Paseo CLI
   echo "Starting Paseo CLI..."
   exec paseo
   ```

3. **Add `init`.** Default no-op — the user is expected to mount over this at container start:

   ```bash
   #!/bin/bash

   echo "No init file configured, skipping..."
   ```

4. **Add `Dockerfile.<name>`.** Keep it slim and based on `node:lts-slim` unless the tool requires something else. Install each tool in its own stage under its own npm prefix, then copy it into the final image as a separate layer:

   ```dockerfile
   # syntax=docker/dockerfile:1

   FROM node:lts-slim AS <tool>
   ARG <TOOL>_VERSION=latest
   RUN npm i -g --no-fund --no-audit --prefix /opt/<tool> <package>@${<TOOL>_VERSION} \
    && rm -rf /root/.npm /tmp/*

   FROM node:lts-slim

   COPY --from=<tool> /opt/<tool> /opt/<tool>

   ENV PATH="/opt/<tool>/bin:${PATH}"

   COPY --chmod=755 <name>/* /usr/local/bin/

   ENTRYPOINT ["entrypoint"]
   ```

   - One stage per tool, one `COPY --from` per tool. A tool whose version did not change between nightly builds is restored from the build cache as an identical layer, so clients only download and extract the tools that changed. Put the most frequently updated tool last: a changed layer invalidates every layer after it on the client, never the ones before it.
   - `ARG <TOOL>_VERSION=latest` keeps `docker build` working locally; CI pins the version via `--build-arg` so the cache key is stable.
   - Clean up inside the same `RUN` that installs (`rm -rf /root/.npm`, unused per-platform binaries), otherwise the waste is baked into the layer.
   - `COPY --chmod=755 <name>/*` stages both `entrypoint` and `init` into `/usr/local/bin/`.
   - `ENTRYPOINT ["entrypoint"]` (no path) so it resolves from `$PATH`.

5. **Add the workflow** at `.github/workflows/<name>.yml`. Use the existing workflows as a template. The real workflow has two jobs: `resolve`, which fetches the latest version of every tool and skips scheduled runs when the published image already has them, and `docker_image`, which smoke tests each platform locally before publishing. The essential shape is:

   ```yaml
   name: <Name>
   on:
     push:
       branches: [main]
     schedule:
       - cron: '0 3 * * *'
     workflow_dispatch:
       inputs:
         tag:
           description: 'Image tag to build and publish'
           required: false
           default: 'latest'
           type: string
   permissions:
     contents: read
   jobs:
     docker_image:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v7

         # One step per bundled tool — fetch the latest version
         - name: Get latest <tool> version
           id: latest_<tool>
           run: |
             TAG=$(npm view <package> version)
             [ -n "$TAG" ] || { echo "Failed to fetch latest version" >&2; exit 1; }
             echo "tag=$TAG" >> $GITHUB_OUTPUT

         - uses: docker/setup-qemu-action@v4
         - uses: docker/setup-buildx-action@v4
         - uses: docker/login-action@v4
           with:
             username: <dockerhub-namespace>
             password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

         - name: Build and publish image
           uses: docker/build-push-action@v7
           with:
             context: .
             file: Dockerfile.<name>
             platforms: linux/amd64,linux/arm64
             build-args: |
               <TOOL>_VERSION=${{ steps.latest_<tool>.outputs.tag }}
             labels: |
               <tool>_version=${{ steps.latest_<tool>.outputs.tag }}
             cache-from: type=registry,ref=<dockerhub-namespace>/<name>:buildcache
             cache-to: type=registry,ref=<dockerhub-namespace>/<name>:buildcache,mode=max
             outputs: type=image,name=<dockerhub-namespace>/<name>:${{ github.event.inputs.tag || 'latest' }},push=true,oci-mediatypes=true,compression=zstd,force-compression=true
   ```

   - `build-args` feed the resolved versions into the Dockerfile stages, so the registry cache (`<name>:buildcache`) is hit for every tool that did not change.
   - `compression=zstd` makes layers decompress several times faster than gzip on low-power clients such as a Raspberry Pi. Pulling zstd layers needs Docker Engine 23 or newer on the client.
   - Before the publish step, build each platform with `load: true` and run every bundled CLI (`<tool> --version`) in the loaded image. A failing upstream package then fails the workflow instead of shipping.
   - Label the image with every tool version plus the base image digest. The `resolve` job compares those labels with the freshly resolved versions on scheduled runs and skips the build when they match. Copy that job from `ai-harness.yml`.

## The init-script convention

`/usr/local/bin/init` is the runtime-customisation seam. The entrypoint runs `bash /usr/local/bin/init` before launching the tool, so users can mount their own script:

```bash
docker run --rm -it \
  -v "$PWD/my-init.sh:/usr/local/bin/init:ro" \
  <dockerhub-namespace>/<image>:latest
```

Keep the default `init` minimal (a no-op is fine) — never bake environment-specific setup into the image. If you find yourself wanting to, add an opt-in flag instead.

## Required secrets

For the workflow to push, the repo needs a single secret:

| Secret | Value |
|---|---|
| `DOCKER_HUB_ACCESS_TOKEN` | A Docker Hub **access token** (not the user's password). Generate one at https://hub.docker.com/settings/security. Access tokens are scoped, revocable independently, and Docker's recommended credential for CI. |

The Docker Hub username is supplied as a literal in the workflow (`username: <dockerhub-namespace>` in the login step) — it does not need to be a secret.

## Manual runs

Any user with write access can trigger a workflow from the GitHub UI:

**Actions → <Workflow name> → Run workflow**

The optional `tag` input lets you publish a non-`latest` tag (e.g. `nightly`, `2026-07-04`, or a version).

## Multi-arch

All images build for both `linux/amd64` and `linux/arm64` via `docker/build-push-action` on top of `docker/setup-qemu-action` and `docker/setup-buildx-action`. No additional setup is needed beyond specifying `platforms` in the workflow.

## Coding style

- Shell scripts: `set -e`, prefer `bash`, keep them small enough to read at a glance.
- Dockerfiles: one stage per tool, one `RUN` per logical concern, clean up in the same `RUN` that creates the waste, `COPY --chmod` for entrypoints.
- Workflows: name steps so the Actions log is readable, fail loudly if an `npm view` returns empty, smoke test before publishing.
- Action versions are bumped by Dependabot (`.github/dependabot.yml`); add new actions with a major version tag (`@v4`) so it can track them.

## Questions

Open an issue if anything is unclear. PRs welcome.