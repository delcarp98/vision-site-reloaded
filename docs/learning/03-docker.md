# 03: Docker for the API

How the API became one image, run with one command. Reread this before deploying to a
rented GPU in Week 3: the same image runs there unchanged.

```
docker compose up -d --build
  │
  ├── build: tar up the context (filtered by .dockerignore) → daemon
  │          run the Dockerfile top to bottom, reusing cached layers
  └── run:   start a container from the image, publish 8000, inject the GPU
```

## Build context and `.dockerignore`
**Problem:** `docker build .` sends the whole directory to the daemon before reading the
Dockerfile. Our repo has a ~7 GB `.venv`.
- `COPY` can only see files that made it into the context.
- `.dockerignore` filters the context. Patterns are anchored to the root, unlike
  `.gitignore`: use `**/__pycache__`, not `__pycache__`.
- `!pattern` re-includes, and the last matching line wins: `artifacts` then
  `!artifacts/weights/yolo26n.pt`.
- Belt and braces: the Dockerfile copies explicit paths (`COPY src ./src`), not `COPY . .`.

## Layer caching
**Problem:** installing torch takes minutes; rebuilding it on every code change is unusable.
- Each instruction is a layer. Docker reuses it if the instruction and its inputs
  (for `COPY`, the file checksums) are unchanged.
- **The first miss rebuilds everything after it.**
- So order by change frequency: rarely changing and slow at the top (base, apt, deps),
  frequently changing at the bottom (code).
- Before an expensive step, copy only what that step needs. The weights go *after*
  the deps install, so swapping models doesn't reinstall torch.
- Verified: a comment in `api.py` → everything through the first `uv sync` is `CACHED`.

## The two-step `uv sync`
**Problem:** this is a src-layout package, so installing it needs `src/`, but copying
`src/` before the deps install would break the cache.
1. `COPY pyproject.toml uv.lock` → `uv sync --frozen --no-dev --no-install-project`
   (deps only, the big cached layer).
2. `COPY README.md src weights` → `uv sync --frozen --no-dev` (just our package, seconds).
- `--frozen`: install exactly the lockfile, fail if it's stale. Reproducible builds.
- `--mount=type=cache,target=/root/.cache/uv`: a BuildKit cache that survives between
  builds but isn't in the image. Needs `UV_LINK_MODE=copy`.
- `README.md` is needed because `pyproject.toml` references it.

## GPU in a container
**Problem:** the container needs CUDA and the NVIDIA driver.
- No CUDA base image: the torch cu128 wheels bundle the CUDA runtime.
- The driver comes from the host. The NVIDIA Container Toolkit injects it when asked:
  `docker run --gpus all`, or in compose the `deploy.resources.reservations.devices`
  block with `driver: nvidia`.
- Forget it and the model quietly runs on CPU. Always check `/model` says `cuda:0`.

## Dev image vs prod image
| | Dev container | Prod image |
|---|---|---|
| Purpose | write code | run code |
| Dev deps (pytest, black) | yes | `--no-dev` |
| Code | bind-mounted, live | copied in, frozen |
| Server | `--reload` | no reload, `0.0.0.0` |
| User | `vscode` | non-root `app` |
| Logs | `logs/*.log` | stdout (`LOG_DIR=""`), read with `docker logs` |

## Runtime details that bit or could bite
- `--host 0.0.0.0`: `127.0.0.1` inside a container is unreachable from outside it.
- `EXPOSE` only documents a port; `-p` / `ports:` publishes it.
- Non-root user needs somewhere writable: `YOLO_CONFIG_DIR=/tmp/Ultralytics`.
- `HEALTHCHECK` uses Python's `urllib` because slim images have no `curl`.
  `start-period` covers model load + warmup.
- `restart: unless-stopped`: comes back after crashes and reboots, stays down if you stop it.
- `compose.yaml` is the current preferred name; `docker-compose.yml` still works.

## Bugs we hit
- **Missing `\` after `HEALTHCHECK ... --retries=3`**: the instruction ended there and
  the next line became a stray `CMD`. Multi-line instructions need a `\` on every line
  but the last.
- **Container healthy, curl "connection refused", empty `PORTS` column**: a stale
  container created from an older compose file, running an old image (shown as a bare
  `sha256:`). Fix: `docker compose down && docker compose up -d --build`.
  Lesson: check `docker compose ps` for `PORTS` and `IMAGE` before debugging the app.

## Checklist after any image change
```bash
docker compose up -d --build
docker compose ps                                  # (healthy), PORTS filled in
curl localhost:8000/model                          # "device": "cuda:0"
curl -F file=@artifacts/image.png localhost:8000/detect   # 9 people
```
