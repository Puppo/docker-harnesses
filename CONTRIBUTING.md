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

4. **Add `Dockerfile.<name>`.** Keep it slim and based on `node:lts-slim` unless the tool requires something else:

   ```dockerfile
   FROM node:lts-slim

   RUN npm i -g <your-tool>@latest

   COPY --chmod=755 <name>/* /usr/local/bin/

   ENTRYPOINT ["entrypoint"]
   ```

   - `COPY --chmod=755 <name>/*` stages both `entrypoint` and `init` into `/usr/local/bin/`.
   - `ENTRYPOINT ["entrypoint"]` (no path) so it resolves from `$PATH`.

5. **Add the workflow** at `.github/workflows/<name>.yml`. Use the existing workflows as a template — the shape is:

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

         - name: Build and publish image
           uses: ilteoood/docker_buildx@master
           with:
             tag: ${{ github.event.inputs.tag || 'latest' }}
             platform: linux/amd64,linux/arm64
             imageName: <dockerhub-namespace>/<name>
             dockerFile: Dockerfile.<name>
             label: <tool>_version=${{ steps.latest_<tool>.outputs.tag }}
             publish: true
             dockerUser: <dockerhub-namespace>
             dockerPassword: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
   ```

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

The Docker Hub username is supplied as a literal in the workflow (`dockerUser: <dockerhub-namespace>`) — it does not need to be a secret.

## Manual runs

Any user with write access can trigger a workflow from the GitHub UI:

**Actions → <Workflow name> → Run workflow**

The optional `tag` input lets you publish a non-`latest` tag (e.g. `nightly`, `2026-07-04`, or a version).

## Multi-arch

All images build for both `linux/amd64` and `linux/arm64` via the `ilteoood/docker_buildx` action. No additional setup is needed beyond specifying `platform` in the workflow.

## Coding style

- Shell scripts: `set -e`, prefer `bash`, keep them small enough to read at a glance.
- Dockerfiles: one `RUN` per logical concern, explicit package versions where stability matters, `COPY --chmod` for entrypoints.
- Workflows: name steps so the Actions log is readable, fail loudly if an `npm view` returns empty.

## Questions

Open an issue if anything is unclear. PRs welcome.