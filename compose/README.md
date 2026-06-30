# IntentGate — install (Compose bundle)

This is the one thing an engineer runs to stand up IntentGate on a host —
no code, only `.env` configuration. It comes in **two topologies**; pick
the one that matches the deployment:

| Topology | When | File | Brings up |
|----------|------|------|-----------|
| **Single host** | a pilot, or one production node | `docker-compose.yml` | Postgres · extractor · gateway · console |
| **High availability** | production, no single point of failure | `docker-compose.ha.yml` + `nginx-lb.conf` | the above **+ Redis + a load balancer**, with ≥2 gateway replicas |

The console's **Setup wizard** generates this same config from a few
answers (Topology → Single / High availability), so you can use the wizard
or these files directly — they produce the same result.

This is the **Install** stage of the journey. When it's done you open the
console and continue: Discover → Govern → Attest → Prove.

> Full prerequisites checklist: **IntentGate-Before-You-Install.html**.
> Full HA / resiliency detail (load balancer, Redis, failure modes):
> **IntentGate-Resiliency-HA.html**.

---

## Before you start

- A Linux host with **Docker** and **Docker Compose v2**.
- The host must reach your **tool/MCP server**; your **agents must reach
  the host** on port 8080.
- From IntentGate: your **license key**, the **release version** to pin
  (e.g. `0.1.3`), and a **registry token** to pull the images.
- HA only: a **managed Redis** and **managed PostgreSQL** for production
  (the bundle ships a test Redis/Postgres so you can try the topology
  first), and a **load balancer** you own.

---

## Common config (both topologies)

**1 — Get the bundle and log in to the registry.**

```bash
# copy this folder onto the host, then:
docker login ghcr.io        # username + the token IntentGate gave you
```

**2 — Create your config.**

```bash
cp .env.example .env
```

Generate the secrets — don't make them up:

```bash
openssl rand -hex 16   # -> POSTGRES_PASSWORD
openssl rand -hex 32   # -> INTENTGATE_MASTER_KEY
openssl rand -hex 32   # -> INTENTGATE_ADMIN_TOKEN
openssl rand -hex 32   # -> AUTH_SECRET
openssl rand -hex 32   # -> INTENTGATE_TOTP_ENCRYPTION_KEY (must be 64 hex chars)
```

Then set the plain values: `INTENTGATE_VERSION` (the release to pin), your
license key, the tool server URL (`INTENTGATE_UPSTREAM_URL`), the gateway
URL agents will call (`INTENTGATE_PUBLIC_GATEWAY_URL`), the console URL
(`CONSOLE_PUBLIC_URL`), your owner email domain, and your IdP's OIDC details.

---

## Path A — Single host (pilot)

**3A — Bring it up.**

```bash
docker compose up -d
```

Compose starts in dependency order: Postgres → extractor → gateway → console.

**4A — Verify.**

```bash
docker compose ps                        # all "healthy"/"running"
curl -fsS http://localhost:8080/healthz  # gateway: {"status":"ok",...}
```

Open **`CONSOLE_PUBLIC_URL`** in a browser and sign in with your IdP.

---

## Path B — High availability (multiple gateways)

Uses `docker-compose.ha.yml` and `nginx-lb.conf`. Same `.env`, with two
differences: point `INTENTGATE_PUBLIC_GATEWAY_URL` at your **load balancer**
(agents call the LB, never a gateway directly), and for production set
`INTENTGATE_REDIS_URL` / `INTENTGATE_POSTGRES_URL` to your **managed**
Redis and Postgres (then delete the bundled `redis`/`postgres` services).

**3B — Bring it up with N gateways.**

```bash
docker compose -f docker-compose.ha.yml up -d --scale gateway=2
```

This starts: N gateway replicas + Redis (shared budget) + a load balancer
(the single entry point) + Postgres + extractor.

**4B — Configure your load balancer.**

The load balancer is **your** infrastructure — IntentGate can't configure
it. The bundled `nginx-lb.conf` is a working reference; reproduce the same
settings on your real LB (cloud LB / NGINX / HAProxy / k8s Service+Ingress):

- **Backend pool:** the gateway replicas on port 8080
- **Algorithm:** round-robin — **no session affinity** (gateway is stateless)
- **Health check:** `GET /healthz` → 200; eject after 2–3 failures
- **TLS:** terminate HTTPS at the LB with your certificate
- **Timeouts:** connect ~5s; read ≥ your slowest tool
- **Retry:** next replica on connection error / 502 / 503 / 504

**5B — Verify.**

```bash
docker compose -f docker-compose.ha.yml ps          # gateway shows N replicas
curl -fsS https://<your-load-balancer>/healthz       # routed to a replica
```

Scale up or down anytime — it's one number:

```bash
docker compose -f docker-compose.ha.yml up -d --scale gateway=4
```

> For machine-level redundancy (replicas on different hosts) use the Helm
> chart in the repo root: `gateway.replicas`, `redisUrl`, external Postgres.

---

## What gets installed

| Container  | Role                          | Single | HA | Reached by |
|------------|-------------------------------|:------:|:--:|------------|
| `gateway`  | enforcement (data plane)      | 1 | N | agents, via the LB in HA |
| `console`  | management UI (control plane) | 1 | 1 | operators, browser |
| `extractor`| intent extraction             | 1 | 1 | gateway (internal) |
| `postgres` | policies · audit · inventory  | bundled | managed | gateway + console |
| `redis`    | shared budget counters        | — | required | gateways (internal) |
| `lb`       | single entry point            | — | required | agents |

The gateway enforces on its own; the console manages it. If the console is
down, enforcement keeps running.

---

## After install

The console opens on the **Install** stage of the Journey bar. From here:
Discover your agents → Govern (own + policy + route + secure) → Attest
ownership → Prove with reports. The console walks you through each.

## Documentation

- **IntentGate-Before-You-Install.html** — prerequisites & checklists (single node).
- **IntentGate-Resiliency-HA.html** — high availability: load balancer (who
  owns it, what to configure), Redis, managed Postgres, failure modes, scaling.
