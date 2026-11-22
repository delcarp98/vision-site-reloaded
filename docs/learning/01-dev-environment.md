# 01: The Dev Environment

What we set up in Week 0, and why each piece exists. The same chain
(Linux + Docker Engine + NVIDIA Container Toolkit) is what you'll set up on the cloud
server in Week 3, so understanding it here pays off twice.

## The layers, from hardware up

```
RTX 5060 Ti
  └─ Windows NVIDIA driver
       └─ WSL2 (Ubuntu 26.04): sees the GPU via /usr/lib/wsl/lib (libcuda.so)
            └─ Docker Engine (runs inside WSL, managed by systemd)
                 └─ NVIDIA Container Toolkit: injects the driver into containers on --gpus
                      └─ Dev container (Ubuntu 24.04 + uv + Python 3.12 + Node 22)
                           └─ .venv: torch (cu128), ultralytics
```

Each layer only needs to know about the one below it.

## What's in the repo, and why

| File / folder | What it is | Why we need it |
|---|---|---|
| `.devcontainer/Dockerfile` | Dev image: system libs, uv, Python 3.12 | Same environment on every rebuild |
| `.devcontainer/devcontainer.json` | GPU flag, ports, volumes, `postCreateCommand` | Opening the folder gives a ready, GPU-enabled env |
| `pyproject.toml` | Direct deps + the PyTorch cu128 index | Blackwell needs cu128 builds. `explicit = true` keeps everything except torch/torchvision on PyPI |
| `uv.lock` | Exact versions and sources of every package | Dev and the prod image install the same thing |
| `.python-version` | `3.12` | Which interpreter uv builds `.venv` with |
| `src/vision_site/` | The app's Python package | The API and worker will live here |
| `.gitignore` | Ignores `.venv`, weights, `runs/`, `artifacts/`, `.env` | Keep downloads, outputs and secrets out of git |
| `artifacts/` (ignored) | Local weights, test media, outputs | One place for generated files |
| `CLAUDE.md`, `docs/` | Working rules, roadmap, these notes | Context for me and for Claude |

## Key ideas

### Docker Engine vs Docker Desktop
Docker Desktop is a Windows/Mac app that runs a hidden Linux VM for you. Docker Engine
is the actual Linux program (`dockerd`). WSL already *is* Linux, so we installed the
engine directly. That's identical to what runs on every Linux server.

- `systemd=true` in `/etc/wsl.conf` is what lets Docker start as a service.
- Being in the `docker` group means you can run `docker` without `sudo`.
  **This is root-equivalent**: anyone in the group can mount `/` into a container.

### GPUs in containers
A container doesn't get the GPU by default. The NVIDIA Container Toolkit hooks into
Docker so that `--gpus all` mounts the driver libraries and device into the container.

**What lives where:**
- **Driver**: on the host (in WSL's case, the Windows driver). Never install an
  NVIDIA Linux driver inside WSL.
- **CUDA runtime libraries**: inside the PyTorch wheel. That's why the dev container
  doesn't use an `nvidia/cuda` base image.

**Blackwell gotcha:** RTX 50-series cards need PyTorch built for CUDA 12.8+
(the `cu128` index). An old build reports `cuda.is_available() == True` and then fails on
the first real operation with "no kernel image is available." Always test with a real
operation, e.g. `torch.ones(3, device="cuda") * 2`.

### Dev container
`.devcontainer/` describes the environment as code:
- `Dockerfile`: what's installed (system libs, uv, Python).
- `devcontainer.json`: how VS Code runs it (GPU flag, ports, volumes, extensions, setup commands).

Change either file → **"Dev Containers: Rebuild Container"** for it to take effect.

`UV_PYTHON_PREFERENCE=only-managed` makes uv ignore the OS's `python3`, which is a
stripped "minimal" build that's missing modules like `json`. The OS Python belongs
to the OS; projects use their own.

### Bind mounts vs named volumes

| | Bind mount | Named volume |
|---|---|---|
| What | An existing host folder shown inside the container | Storage Docker creates and manages |
| Example | `~/vision_site_project` → `/workspaces/vision_site_project` | `vision-site-uv-cache` → `/home/vscode/.cache/uv` |
| Lives at | Wherever the folder is | `/var/lib/docker/volumes/<name>/_data` (root-owned) |
| Use for | Source code you edit | Caches, databases, anything the app owns |
| Inspect | `ls` | `docker volume ls`, `docker volume inspect <name>` |

**What survives what:**

| | Stop/restart container | Rebuild container |
|---|---|---|
| Project files (incl. `.venv`) | kept | kept |
| Named volumes | kept | kept |
| Anything else installed in the container | kept | **lost** |

Rule: if you want it after a rebuild, put it in the Dockerfile, the project, or a volume.

### Reproducible dependencies
- `pyproject.toml`: what you asked for (`fastapi`).
- `uv.lock`: exactly what you got, every sub-dependency pinned.
- `uv add <pkg>` adds a dependency; `uv sync` makes `.venv` match the lock.
- `uv.lock` goes in git; `.venv` doesn't. The lock is the recipe, and the venv is
  the result of following it on one particular machine.

## Debugging habits learned so far
- A command that hangs with no prompt is often waiting for terminal input
  (e.g. `gpg --dearmor` without its pipe). Ctrl+C is safe.
- Warnings aren't failures: read them, find out what actually failed, then decide.
- Read the full error. The cause is usually in the last few lines.

## Commands cheat sheet
```bash
docker ps                                  # running containers
docker volume ls                           # named volumes
docker run --rm --gpus all ubuntu:24.04 nvidia-smi   # GPU-in-Docker smoke test
uv sync                                    # make .venv match uv.lock
uv add <pkg>                               # add a dependency
uv run python script.py                    # run inside the synced env
```
