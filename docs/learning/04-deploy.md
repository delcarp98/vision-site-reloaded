# 04: Deploying to AWS

What actually happened when the app went live at `https://52-0-189-122.sslip.io` on
2026-11-01. The step-by-step runbook is `docs/DEPLOY.md`; this note explains what each
piece *is* and why it's there.

```
your browser (anywhere)
  │  https://52-0-189-122.sslip.io
  ▼
DNS (sslip.io) ── "that name is 52.0.189.122" ──▶ Elastic IP
  │
  ▼
Security group (AWS firewall): 80 ✓  443 ✓  22 only from home IP
  │
  ▼
EC2 instance g4dn.xlarge (Ubuntu + NVIDIA driver + Docker, from the AMI)
  └── docker compose (same compose.yml as local)
        web (Caddy :80/:443, HTTPS) ──/api──▶ api ──▶ db (Postgres)
                                               │
                                     worker ◀──┘   both use the T4 GPU
```

**Analogy: renting a food truck.** AWS is the truck rental lot. The **instance** is the truck
you rent by the hour. The **AMI** is the kitchen kit already installed (stove = NVIDIA driver,
fridge = Docker). The **Elastic IP** is a reserved parking spot with a fixed address, and
**sslip.io** is the sign at the corner pointing to it. The **security group** is the bouncer:
anyone can walk up to the order window (80/443), but only you, with your key (`.pem`), can
open the back door (22). **Caddy** is the cashier who checks everyone at the window, and the
**api/worker** are the cooks in the back. You pay for the truck while it's out, and for the
parking spot until you give it up.

## The server: EC2 + AMI
**Problem:** you need a computer with a GPU that the internet can reach, without buying one.
- **EC2 instance** = a rented virtual machine. `g4dn.xlarge`: 4 vCPU, 16 GB RAM,
  NVIDIA T4 (16 GB), $0.526/hr. It shares nothing with your PC: own IP, own GPU, own disk.
- **AMI** = the disk image it boots from. We used *Deep Learning Base AMI with Single CUDA
  (Ubuntu 24.04)*, x86: the NVIDIA driver, Docker and the NVIDIA Container Toolkit come
  preinstalled. Not the PyTorch AMI: our image brings its own torch.
- **x86 vs Arm:** `g4dn` is Intel, so the Arm build of the same AMI won't boot.
- **Quota:** counted in vCPUs, not instances. 4 vCPU = exactly one `g4dn.xlarge`.
- **Free plan vs Paid plan:** the free plan blocks GPU types. Upgrading is free; it just
  allows billing.
- **Driver vs CUDA:** the driver lives on the host; `nvidia-smi` showed CUDA 13.2, the
  newest it supports. Our torch cu128 needs ≥ 12.8, so it's fine. The container gets the
  driver injected via `--gpus all` / the compose `deploy` block (see note 03).

## The network: security group, IPs, DNS
**Problem:** people need to find the server, and only the right traffic should get in.
- **Security group** = a firewall in front of the instance, outside of it. Rules we used:
  - 22 (SSH) from `73.x.x.x/32` only. `/32` = one address. "My IP" = your router's public
    IP, which is why the VPN had to be off: with it on, it saves the VPN's IP instead.
  - 80 and 443 from `0.0.0.0/0` (everyone). The site is meant to be public.
  - Nothing else. Postgres and the API publish no ports; only Caddy does.
- **Public IP vs Elastic IP:** the auto-assigned public IP changes on every stop/start.
  An **Elastic IP** is a fixed one you own until you *release* it. It's AWS's address,
  not your home's.
- **Hostname / DNS:** a hostname is a name DNS turns into an IP (like contacts → phone
  number). We didn't buy a domain: **sslip.io** answers any name with the IP written in
  it (`52-0-189-122.sslip.io` → `52.0.189.122`). A real domain would just be an A record
  pointing at the Elastic IP plus `SITE_ADDRESS=yourdomain.com`.

## Getting in: SSH and keys
**Problem:** you need to run commands on a machine in Virginia, and nobody else should.
- **Key pair** = a lock (public key) and its only key (private key, the `.pem`). Key pairs
  are their own EC2 resource, per region, and exist before any instance.
- At launch, AWS installs the public half in `/home/ubuntu/.ssh/authorized_keys`. That's
  the only way in: no password.
- `ssh -i ~/.ssh/vision-site.pem ubuntu@52.0.189.122`: from then on WSL is a remote
  control. Commands run on the server, output comes back. Prompt `ubuntu@ip-172-31-...`
  = you're on the server (`172.31.x.x` is its private IP inside AWS).
- **Deploy key** = the same idea pointed the other way. The repo is private, so the
  server made its own key (`ssh-keygen`) and GitHub got the public half as a
  **read-only, single-repo** key. Safer than `gh auth login`, which would leave a token
  for *all* your repos on an internet-facing machine. Clone with `git@github.com:...`,
  not `https://`.

## Secrets: `.env`
**Problem:** the DB password and signup code must reach the containers but never git or
the web.
- The E3 block is **shell commands that create** `.env`; paste it at the prompt, not into
  the file (the first attempt did the latter, and the file held the script text).
- `PW=$(openssl rand -hex 24)` → random password; used in both `POSTGRES_PASSWORD` and
  `DATABASE_URL` so they always match. `${IP//./-}` → dots to dashes.
- Compose reads `.env` next to `compose.yml` and passes values in as environment variables.
- Kept out of everything: `.gitignore` (not in git), `.dockerignore` (not in images),
  Caddy only serves `/srv` (no URL maps to it), `chmod 600` (only `ubuntu` can read it).

## HTTPS: Caddy + Let's Encrypt
**Problem:** browsers need a certificate proving the server owns the hostname, and the
session cookie is HTTPS-only.
- `SITE_ADDRESS` set to a real hostname makes Caddy request a free certificate from
  Let's Encrypt automatically.
- Let's Encrypt checks control by connecting back to that hostname, which is why port 80
  must be open even though the site runs on 443. Caddy then redirects 80 → 443.
- Certificates live in the `caddy_data` volume. Keep it, or repeated rebuilds hit Let's
  Encrypt rate limits.

## Who can attack it, and what stops them
Bots scan every IP on the internet; expect junk requests within minutes. The layers:
1. Firewall: only 80/443 open to the world; SSH = your IP + your `.pem`.
2. Only Caddy is exposed; db/api/worker have no published ports.
3. Every GPU/data endpoint requires a login; jobs are scoped to their owner.
4. Signup needs the code; login/signup rate-limited to 10 per 5 min per real client IP.
5. Upload caps: images 10 MiB / 40 MP, videos 200 MiB / 5 min / 4K, Caddy body 210 MB.

Known gaps: `POST /logs` is unauthenticated (log noise only, rotation caps it); no
protection from a traffic flood; no backups.

## Billing model
| What | Billed while | ~Cost |
|---|---|---|
| Instance | **running** | $0.526/hr |
| 60 GiB gp3 disk | it **exists** (even stopped) | ~$4.80/month |
| Elastic IP | it's **allocated** (attached or not) | ~$3.60/month |
| Data out | per GB after 100 GB/month | $0.09/GB |

- Bots can't move the fixed costs. The real risk is **forgetting**: a running instance is
  ~$380/month.
- Guards: the $20 budget alarm (emails, data lags hours), a budget action to auto-stop
  (needs the instance ID, so set it after launch), and `sudo shutdown -h +180` per session.
- **Stop** = pause (disk + IP still bill, same URL when restarted).
  **Terminate** = gone, data included. Then release the Elastic IP and delete the
  deploy key, or they linger.

## How this compares to "real" production
Same shape as many small real apps (one VM, reverse proxy with auto-HTTPS, compose,
secrets in a locked file). Bigger teams add: a managed DB with backups (RDS), a secrets
manager, images built in CI and pulled instead of built on the server, and several
servers behind a load balancer.

## Bugs we hit
- **`git clone` asked for a username**: the repo is private. Fix: deploy key + SSH URL.
- **`.env` contained the script**: pasted into the file instead of the shell.
  Check: `grep SITE_ADDRESS .env` must show a real hostname, not `${IP//./-}`.
- **Site loaded on one PC, not another**: server was fine; the other machine's network
  blocked sslip.io. Test from a phone on cellular / `nslookup` to split server problems
  from client problems.
- **AMI name in the runbook was outdated**: AWS renames images. Match on publisher
  (Amazon, verified), OS, architecture and date, not the exact name.

## Checklist for next deploy
```bash
nvidia-smi && docker run --rm --gpus all ubuntu nvidia-smi   # T4, CUDA ≥ 12.8
grep SITE_ADDRESS .env                                        # real hostname
docker compose up -d --build && docker compose ps             # db/api healthy, migrate exited 0
docker compose logs web | grep -i certificate                 # obtained
docker compose logs api | grep "model ready"                  # cuda:0
```
