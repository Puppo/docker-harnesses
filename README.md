<div align="center">

# docker-harnesses

*Reusable Docker images that bundle CLI-based AI tools into ready-to-run containers.*

[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![Docker Hub](https://img.shields.io/badge/Docker%20Hub-delpuppoluca-2496ED?style=flat-square&logo=docker&logoColor=white)](https://hub.docker.com/u/delpuppoluca)
[![CI](https://img.shields.io/github/actions/workflow/status/Puppo/docker-harnesses/ai-harness.yml?style=flat-square&label=ai-harness)](https://github.com/Puppo/docker-harnesses/actions/workflows/ai-harness.yml)

</div>

A small collection of slim, multi-architecture Docker images built on top of `node:lts-slim`, each packaging a CLI tool with a small `entrypoint` and a user-overridable `init` script.

## Why

Most AI coding CLIs are released frequently and expect to be run as the container's entrypoint. Pinned base images go stale fast; rebuilding locally every time is friction. These images give you:

- A pre-installed, always-current CLI image that picks up upstream releases on a daily CI rebuild.
- A standard `/usr/local/bin/init` hook so you can mount a custom setup script without rebuilding the image.
- A multi-arch (`linux/amd64`, `linux/arm64`) manifest suitable for both Intel/ARM dev hosts and CI runners.

## Images

| Image | Tags | Tools bundled | Entrypoint |
|---|---|---|---|
| [`delpuppoluca/ai-harness`](https://hub.docker.com/r/delpuppoluca/ai-harness) | `latest` | [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code), [`@openai/codex`](https://www.npmjs.com/package/@openai/codex), [`@opencode/cli`](https://www.npmjs.com/package/@opencode/cli), [`@getpaseo/cli`](https://www.npmjs.com/package/@getpaseo/cli) (`beta`), [`@earendil-works/pi-coding-agent`](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) | `paseo` (unified interface over both providers) |

## Quick start

Pull and run:

```bash
docker pull delpuppoluca/ai-harness:latest

docker run --rm -it \
  -e ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  -v "$PWD:/work" -w /work \
  delpuppoluca/ai-harness:latest
```

Mount a custom init script — it runs before the entrypoint every time the container starts:

```bash
cat > /tmp/my-init.sh <<'EOF'
#!/bin/bash
echo "configuring git…"
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
EOF
chmod +x /tmp/my-init.sh

docker run --rm -it \
  -v "$PWD:/work" -w /work \
  -v /tmp/my-init.sh:/usr/local/bin/init:ro \
  delpuppoluca/ai-harness:latest
```

Launch the bundled `pi` agent directly (bypassing the Paseo entrypoint):

```bash
docker run --rm -it \
  -e ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  -v "$PWD:/work" -w /work \
  --entrypoint=pi \
  delpuppoluca/ai-harness:latest
```

## Layout

Each image is a self-contained folder plus a Dockerfile at the repo root:

```
.
├── ai-harness/
│   ├── entrypoint       # runs /usr/local/bin/init, then exec <tool>
│   └── init             # default no-op — mount your own at runtime
├── Dockerfile.ai-harness
└── .github/workflows/ai-harness.yml
```

The pattern is reusable: see [CONTRIBUTING.md](CONTRIBUTING.md) for how to add a new image.

## CI

Each image has a matching `.github/workflows/<name>.yml` that:

- Triggers on `push` to `main`, a daily `0 3 * * *` cron, and `workflow_dispatch` (any user with write access can trigger a manual run, optionally with a custom tag).
- Queries npm for the latest upstream version of every bundled tool and bakes them in as Docker labels, together with the digest of the base image.
- On scheduled runs, compares those against the labels of the published image and skips the build when nothing changed. Pushes and manual runs always build.
- Passes the versions to the Dockerfile as build args, so each tool's layer is restored from a registry build cache when its version did not change. Pulling a new nightly image only downloads and extracts the tools that actually changed.
- Builds each platform locally first and runs every bundled CLI as a smoke test, so a broken upstream package never reaches the registry.
- Builds and pushes a multi-arch manifest via [`docker/build-push-action`](https://github.com/docker/build-push-action), with layers compressed as zstd for fast extraction on low-power hosts such as a Raspberry Pi (requires Docker Engine 23+ on the client).

Action versions are kept current by Dependabot (`.github/dependabot.yml`), which opens one grouped pull request per week when an action has a new release.

## License

[MIT](LICENSE)