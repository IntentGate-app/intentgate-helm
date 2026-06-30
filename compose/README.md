# IntentGate — install (starter bundle)

This is the one thing an engineer runs to stand up IntentGate on a host.
It brings up four containers — **Postgres, intent extractor, gateway,
console** — wired together, in the right order. No code, only `.env`
configuration.

This is the **Install** stage of the journey. When it's done, you open
the console and the journey continues (Discover → Govern → Attest → Prove).

---

## Before you start

You need:

- A Linux host with **Docker** and **Docker Compose v2**.
- The host must be able to **reach your tool/MCP server**, and your
  **agents must be able to reach this host** on the gateway port (8080).
- From IntentGate: your **license key**, the **release version** to pin,
  and a **registry token** to pull the images.

---

## Steps

**1 — Get the bundle and log in to the registry.**

```bash
# copy this folder (deploy/starter) onto the host, then:
docker login ghcr.io        # username + the token IntentGate gave you
```

**2 — Create your config.**

```bash
cp .env.example .env
```

Open `.env` and fill it in. Generate the secrets — don't make them up:

```bash
openssl rand -hex 16   # -> POSTGRES_PASSWORD
openssl rand -hex 32   # -> INTENTGATE_MASTER_KEY
openssl rand -hex 32   # -> INTENTGATE_ADMIN_TOKEN
openssl rand -hex 32   # -> AUTH_SECRET
openssl rand -hex 32   # -> INTENTGATE_TOTP_ENCRYPTION_KEY (must be 64 hex chars)
```

Then set the plain-value fields: your license key, the tool server URL
(`INTENTGATE_UPSTREAM_URL`), the gateway URL agents will call
(`INTENTGATE_PUBLIC_GATEWAY_URL`), the console URL (`CONSOLE_PUBLIC_URL`),
your owner email domain, and your IdP's OIDC details.

**3 — Bring it up.**

```bash
docker compose up -d
```

Compose starts them in dependency order: Postgres → extractor → gateway
(waits for Postgres healthy) → console (waits for gateway healthy).

**4 — Verify.**

```bash
docker compose ps                              # all should be "healthy"/"running"
curl -fsS http://localhost:8080/healthz        # gateway: {"status":"ok",...}
```

Then open **`CONSOLE_PUBLIC_URL`** in a browser and sign in with your IdP.

**5 — You're at the start of the journey.**

The console opens on the **Install** stage of the Journey bar. From here:
Discover your agents → Govern them (own + policy + route + secure) →
Attest ownership → Prove with reports. The console walks you through each.

---

## What just got installed

| Container  | Role                          | Reached by            |
|------------|-------------------------------|-----------------------|
| `gateway`  | enforcement (data plane)      | **agents**, port 8080 |
| `console`  | management UI (control plane) | **operators**, browser, port 3000 |
| `extractor`| intent extraction             | gateway (internal)    |
| `postgres` | policies · audit · inventory  | gateway + console (internal) |

The gateway enforces on its own; the console manages it. If the console
is down, enforcement keeps running.

---

## Production / HA

This bundle is **single node** (in-memory budget counters, one of each).
For high availability — ≥2 gateway replicas, **Redis** for shared budget
counters, and **managed Postgres** with failover — use the Helm chart
(`deploy/helm/`) instead, which exposes `gateway.replicas`, a Redis
endpoint, and an external Postgres URL. Same four components, scaled.
