# A way forward

Planning notes for the next project ("v2"): another computer vision web app, built
with real engineering design this time. Written 2026-11-22. This covers the **what**
and **why**; the **how** (architecture, Kubernetes, frontend) comes next.

## Why a v2

v1 (this repo) was about learning the process of deploying something to the internet.
It worked, but the code and design came second. What went wrong:

- **Built quick and easy, not engineered.** Nothing about it was designed to be brought
  up, torn down, or scaled.
- **State lived on the server.** Users, jobs and videos were on the EC2 instance's disk,
  so the project died with the instance. (Stopping keeps the disk; terminating deletes it.
  The real problem was keeping state on the same machine as compute.)
- **Login felt barebones.** Homemade username/password, no OAuth, no email verification.
  It worked, but it felt like a sketchy site.
- **Caddy was a mystery.** It was there, it worked, but I didn't know why.
- **SSE and WebSockets were fuzzy.** Now covered in
  [learning/05-polling-sse-websockets.md](learning/05-polling-sse-websockets.md).
- **Claude used "raw".** No skills, no plugins. I want to find out for myself which of
  those are actually useful and which are hype.

## What I learned about cost

The big surprise: **GPU inference is what makes ML projects expensive**, not hosting.

| Thing | Cost |
|---|---|
| One `g4dn.xlarge` (NVIDIA T4), on 24/7 | ~$0.53/hr ≈ **$384/month**. Same price since 2019 |
| A small CPU server that runs a whole normal website | ~$10–15/month |
| Server-side **live video** detection, 300 users at 10 FPS | 3,000 detections/s → hundreds of vCPUs → **thousands/month** at full load |
| **Uploads** processed by a queue on CPU | cost follows total work: ~$1 per million images, ~$0.002–0.005 per 1-min video |

Takeaways:

- GPUs are expensive when idle and cheap per detection when busy. Always-on is the trap.
- Live video is the most compute-hungry feature possible. Dropping it collapsed the cost.
- With a queue, cost follows **total work**, not how many people are online at once.
- Most of a small project's bill is the **fixed baseline** (cluster, database, load
  balancer), not usage.
- Hobby ML projects stay cheap by running on the user's device, limiting usage, or
  calling pay-per-use APIs.

### Side discovery: inference in the browser

WebGPU lets a web page use the visitor's own GPU; ONNX Runtime Web / Transformers.js
run models on it (falling back to CPU via WebAssembly). Tried
[RF-DETR Nano WebGPU](https://huggingface.co/spaces/webml-community/RF-DETR-Nano-WebGPU)
on my 5060 Ti: real-time detection with zero server cost. `chrome://gpu` shows
"WebGPU: Hardware accelerated". **Not in v2 scope**; a candidate for phase 2.

## What v2 is

A website where people make an account, upload images and short videos, and get
objects detected on them.

- **CPU only.** A tiny, fast, Apache-2.0 licensed detector (candidates: YOLOX-Nano/Tiny,
  NanoDet-Plus, PP-PicoDet, D-FINE-N). Ultralytics YOLO is out (AGPL). No GPU, no
  WebGPU for now.
- **No live video.** Uploads only, processed asynchronously by workers pulling from a queue.
- **Scalable to ~1000 users at once**, proven with a load test, even though real traffic
  will likely never get there.
- **All of this is a cost decision**: CPU servers instead of GPU inference.

## Decisions

| Area | Decision |
|---|---|
| Cloud | **AWS**. I want to learn it, even though cheaper providers exist |
| Orchestration | **Kubernetes on EKS**. Develop on a local cluster for free |
| Compute | **Graviton (ARM)** nodes, ~20% cheaper; spot instances for workers |
| Networking | **No NAT gateway** (saves ~$32/month). Public subnets + strict security groups |
| Database | **Managed Postgres (RDS)**, separate from the servers |
| Files | **S3**. Uploads and results **auto-delete after 30 days** |
| Persistence | Servers are disposable. Tear down the cluster and nothing is lost |
| Scaling | Web pods scale on traffic; workers scale on **queue length**; a **max-replica cap** bounds the bill |
| Overload | Extra load waits in the queue with a **visible position** ("you're #14"), never a bigger bill |
| Job progress | **SSE**, driven by events instead of polling |
| Sign-in | **OAuth (Google/GitHub) and email/password**, both done properly (below) |
| Frontend | A real **landing page and dashboard**, designed together. Not decided yet |
| CI/CD | **GitHub Actions**: test, build ARM images, deploy on push |
| Infrastructure | **Defined in code** (Terraform or `eksctl`): up in ~20 min, destroyed in ~10 |
| Repo | **Private**, for now |
| Live window | About **2 weeks** on AWS, then tear down |

### Email/password "done properly"

- Verification email on signup; password reset by email. Sent with **AWS SES**
  (~$0.10 per 1000 emails). Request SES production access a few days ahead: new
  accounts start in a sandbox that only sends to verified addresses.
- **Account linking**: signing up with email and later using "Sign in with Google" on
  the same address is one user, not two.

## What "holds up at 1000 users" means

The load test passes when:

> 1000 users are active at once. Within 10 minutes they submit **1000 images and 100
> short videos**. **95% of page loads take under 0.5 s**, and **95% of jobs start within
> 2 minutes**.

That needs roughly 10–20 vCPUs of workers. More load than that queues up instead of
scaling cost up.

## Expected cost

Rough, to verify when building. AWS bills by the hour.

| Piece | AWS, per month |
|---|---|
| EKS control plane | $73 |
| 2 small Graviton nodes | ~$25–50 |
| RDS Postgres (smallest) | ~$15–25 |
| Load balancer | ~$18–22 |
| S3 + downloads | a few dollars |
| SES email | ~$0 |
| GitHub Actions (private repo, 2,000 free min/month) | $0 |
| **Baseline, full month** | **~$135–175** |
| **2-week live window, trimmed** | **~$65–85** |

Usage adds little: 1000 users, 20k images and 5k videos in a month is ~$20–30 of
processing. Check **Billing → Credits** first: accounts created since mid-2025 get
$100–200 in AWS credits.

## Before the "how"

1. **Learn Kubernetes basics.** I don't know k8s yet and need a mental model before
   designing around it.
2. **Benchmark the candidate models on my PC** (CPU, 320 and 640 input) to get real FPS
   and pick the model. Every capacity and cost number depends on it.
3. **Architecture design**: services, data flow, queue, storage, auth. Drawn and decided
   before code.
4. **Claude workflow**: new repo, CLAUDE.md, try a few skills/plugins on real work and
   compare against using Claude raw.
5. **Frontend and product**, designed together.
