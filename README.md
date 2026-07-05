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
| [`delpuppoluca/ai-harness`](https://hub.docker.com/r/delpuppoluca/ai-harness) | `latest` | [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code), [`@openai/codex`](https://www.npmjs.com/package/@openai/codex), [`@getpaseo/cli`](https://www.npmjs.com/package/@getpaseo/cli) | `paseo` (unified interface over both providers) |

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
- Queries npm for the latest upstream version of every bundled tool and bakes them in as Docker labels.
- Builds and pushes a multi-arch manifest via [`ilteoood/docker_buildx`](https://github.com/ilteoood/docker_buildx).

## License

[MIT](LICENSE)