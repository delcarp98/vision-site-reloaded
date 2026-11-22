# 02: The Architecture, and Why

The target system, piece by piece. Read this before Week 2; reread it before Week 3.
Every component is here because of a specific problem it solves. If you can't name
the problem, you don't need the component yet.

```
Browser (React + Vite)
  │  drag/drop, canvas overlay, progress bar, webcam
  ▼  HTTPS
Caddy ─────────────── reverse proxy: TLS certificates, one public entry point
  ▼
API (FastAPI) ──────── auth, validation, creating jobs, reading results
  ├── Postgres ─────── users, jobs, detections (structured data)
  ├── MinIO / S3 ───── image/video files, annotated outputs (big blobs)
  ├── Redis ────────── job queue + progress pub/sub
  │     ▼
  │   GPU worker ───── loads YOLO once, pulls jobs, writes results
  └── WebSocket/SSE ── pushes progress and results to the browser
```

## Component by component

### Frontend: React + Vite
**Problem:** the user needs to upload files and *see* results, not read JSON.
- Drag/drop → `FormData` → `fetch("/api/detect")`.
- Boxes come back in **original image pixel coordinates**; the image is shown at a
  different size, so you scale boxes before drawing them on a `<canvas>`.
- Vite dev server (port 5173) proxies `/api` to FastAPI (port 8000) during development,
  so you avoid CORS. In production, Caddy does the same job.

### API: FastAPI
**Problem:** something has to decide who you are, what you're allowed to do, and turn
HTTP requests into work.
- Loads config, talks to Postgres, MinIO and Redis.
- For **images**, it can run inference directly and quickly (milliseconds with a nano model).
- For **videos**, it creates a job and returns immediately with a job ID.
- **Watch out:** a GPU call is *blocking*. Inside an `async def` endpoint it freezes
  the event loop, so every other request waits. You'll see this happen on Day 2.

### Worker + Redis queue
**Problem:** a 2-minute video takes far longer than an HTTP request should stay open,
and a crash during inference shouldn't take the web server down with it.
- API puts `{job_id}` on a Redis queue → worker picks it up → updates job status in Postgres.
- The worker owns the GPU and loads the model **once** at startup.
- Separate process = separate failure domain. If the worker runs out of GPU memory,
  the website stays up.

### Postgres
**Problem:** you need durable, queryable records: users, which jobs belong to whom,
job status, detections.
- Schema changes go through **migrations** (Alembic), never by hand.
- Data lives in a **named volume**. Forget the volume and a redeploy wipes the database.

### Object storage: MinIO locally, S3-compatible in the cloud
**Problem:** files are big and containers are disposable.
- Files go in a bucket; Postgres stores the *key* (path), not the bytes.
- **Presigned URLs** let the browser download directly from storage, without going
  through your API.
- MinIO speaks the S3 API, so the code is identical locally and in the cloud.

### Live updates: WebSocket or SSE
**Problem:** "is my video done yet?"
- **Polling** (asking every second) works but is wasteful.
- **SSE** (Server-Sent Events): server → browser, one direction, very simple. Good for progress.
- **WebSocket**: both directions. Needed for webcam frames (browser → server → browser).
- The worker publishes progress to Redis; the API relays it to whichever browser is listening.

### Caddy
**Problem:** HTTPS certificates, and a single public door for both the frontend and the API.
- Gets and renews TLS certificates automatically.
- Routes `/api/*` to FastAPI and everything else to the built React files.

### Auth: sessions
**Problem:** your jobs are yours; strangers shouldn't use your GPU.
- **Authentication** = who are you (login). **Authorization** = what you may access
  (only your own jobs). Bugs usually hide in authorization.
- Passwords are **hashed** with argon2, never stored and never encrypted.
- A session ID goes in an `httpOnly; Secure; SameSite` cookie, so JavaScript can't read it.

## Two request flows

**Image (fast path):**
```
browser ──POST /api/detect──▶ API ──▶ model (GPU, ~ms) ──▶ JSON boxes ──▶ browser draws
```

**Video (async path):**
```
browser ──POST /api/jobs──▶ API ──store file──▶ MinIO
                             ├──insert job (queued)──▶ Postgres
                             └──enqueue job_id──▶ Redis ──▶ worker
browser ◀──202 {job_id}─────┘                               │ decode → infer → encode
browser ◀──progress (SSE)── API ◀──pub/sub── Redis ◀────────┤
browser ──GET result URL──▶ MinIO (presigned)    ◀── upload ┘ + status=done in Postgres
```

## What we're deliberately NOT doing
- **Kubernetes / autoscaling:** one user, one VM. Compose is enough.
- **Microservices:** one API, one worker.
- **Training models:** we use pretrained weights; serving them is the skill being learned.

Knowing what to leave out is part of the skill. Most failed projects are over-built
rather than under-built.
