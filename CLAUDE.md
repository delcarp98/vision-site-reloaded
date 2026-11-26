# Vision Site Reloaded (v2)

A computer vision web app built with real engineering design: people sign in, upload
images and short videos, and get objects detected by CPU workers pulling from a queue.
Runs on Kubernetes (EKS), state in RDS + S3, scales to ~1000 concurrent users with a
bounded bill. v1 lives in `~/vision_site_project`; why v2 exists is in `docs/PLAN.md`.

## How to work with me

- **Teach, and build.** The goal is for me to understand and be able to rebuild this
  myself. Explain concepts and tradeoffs, give me specs, and let me write the core code while
  also being able to use the LLM. Continuously review my code the way a senior engineer would.
- Only write boilerplate or config once I understand what it does, and explain each part.
- Default to showing me the code to write (full file or exact snippet, with a short note on
  each non-obvious line) so I can type it myself. Only write files directly when I say
  "implement" or "do it".
- I have 5 years of experience as an engineer. Keep replies short and skip the basics. If I need
  clarification, I'll ask.
- No analogies. Answer at getting-started depth: the minimum needed to move forward; I'll
  ask for more.
- **Design before code.** Nothing gets built until its piece of the architecture is
  written down and decided in `docs/`.

## Context

- Plan, decisions and cost targets: `docs/PLAN.md`. Follow its "Before the how" order
  unless I say otherwise.
- Learning notes: `docs/learning/` (01–05 carried over from v1). Add a note when we finish a topic.
- Hardware: RTX 5060 Ti 16 GB, WSL2, Docker Engine (no Docker Desktop). Develop on a
  local Kubernetes cluster; AWS only for the ~2-week live window.
- CPU architecture is **undecided**: default to x86_64 (amd64) images and nodes. Don't
  assume ARM/Graviton until we decide otherwise.
- Constraints: CPU-only inference, Apache-2.0/MIT models only (no Ultralytics/AGPL),
  no NAT gateway, servers are disposable (all state in RDS/S3).
- Dependencies: uv (`uv add`, `uv sync`), never `uv pip install` for project deps.
- Logging: every new feature logs its key events and failures, with job/user IDs.
- Claude workflow is part of the experiment: when we try a skill or plugin, note in
  `docs/claude-workflow.md` whether it actually helped vs. using Claude raw.
