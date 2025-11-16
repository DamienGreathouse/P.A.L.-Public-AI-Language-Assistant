# P.A.L. Public Ai Language Assistant
Deliver a lightweight, self‑hostable, privacy‑first language model stack that any community, small business or individual can run locally and keep 100 % of the data on their own hardware.

**Description**  
**P.A.L.** is a modular, open‑source AI platform that lets anyone run a state‑of‑the‑art language model on their own hardware—no cloud, no data‑outs.  Built on top of Ollama, pgVector, n8n, and Open‑WebUI, the stack is packaged in a single Docker‑Compose file that you can **easily update** (pull a new image, rebuild a Dockerfile, or switch tags) and the system will automatically restart with the new version.  It is accessed through a sleek Caddy‑based reverse proxy and can be refreshed with a single `docker compose pull` + `docker compose up -d`.  P.A.L. empowers local knowledge workers, educators, and hobbyists to have a private, always‑on AI assistant that respects their privacy while staying cutting‑edge.

**Summary**  
1. **Self‑hosted, zero‑cloud** – run everything on your own server or laptop, no external API calls.  
2. **Easy to update, self‑patch** – a simple Docker pull and compose‑up keeps the stack secure without manual edits.  
3. **Community‑friendly** – all code is on GitHub, all docs are Markdown, and the Docker stack can be pulled and launched in under five minutes. 

### **Security at the core**  
P.A.L. is engineered with *security‑by‑design* in mind. <br/><br/>
• **Local data residency** – all user data stays on the host; there are no outbound network calls to cloud providers that could expose your conversations.  
• **Network isolation** – the Postgres/pgVector database is confined to a private `dbnet` bridge that only the n8n orchestrator can reach, effectively cutting off attackers who compromise the public web tier from accessing the data layer.  
• **Minimal attack surface** – the only services exposed to the internet are the reverse‑proxy (Caddy) and the n8n and Open‑WebUI UIs. IP filter is implemented in Caddy file.
• **Immutable image updates** – by pinning tags in the Compose file and using a lightweight updater (e.g., Watchtower or a cron job), you avoid “unknown‑source” updates and retain full control over which releases are deployed.  
• **Transparent provenance** – every component is open‑source and publicly audited; pull requests and vulnerability scans are part of the normal development cycle, so you can verify the integrity of the code before you run it.  
• **Runtime patching** – optional post‑restart commands (e.g., `apt‑get upgrade –y`) ensure that the underlying OS packages stay current without needing a full rebuild, protecting against known CVEs.  

Together, these measures give you a **secure, self‑contained AI assistant** that you can trust to keep your data confidential while still delivering state‑of‑the‑art language capabilities.

## Table of Contents  

| Section | Description |
|---------|-------------|
| [What it is](#what-it-is) | Overview of the stack |
| [Architecture](#architecture) | Services & networking |
| [Getting Started](#getting-started) | Quick‑start steps |
| [Configuration](#configuration) | `.env`, `Caddyfile`, and other files |
| [Persistence & Backup](#persistence-and-backup) | Where data lives |
| [Upgrading & Maintenance](#upgrading--maintenance) | Keeping the stack up‑to‑date |
| [Troubleshooting](#troubleshooting) | Common issues & fixes |
| [License & Credits](#license--credits) | Who built this? |

---

## What it is  

P.A.L. Public Ai Language Assistant is a **single‑file Docker‑Compose** stack that gives you:

| Service | Purpose |
|---------|---------|
| **Open WebUI** | Front‑end for the Ollama LLM – chat, code, docs, etc. |
| **n8n** | Low‑code workflow automation engine. |
| **Ollama** | Local LLM runtime (GPU‑enabled). |
| **Caddy** | Reverse‑proxy that automatically obtains Let’s Encrypt TLS certificates and redirects all HTTP → HTTPS. |
| **pgvector** | PGvector‑enabled PostgreSQL. (GPU‑enabled) |
| **searXNG** | web search for Open WebUI and n8n |

All services are exposed only via **HTTPS** on two public domains:

* `https://pal.example.com` → Open WebUI  
* `https://automation.example.com` → n8n

---

## Architecture  

```
┌───────────────────────────────────────────────────────────────┐
│                      Public Internet                          │
│  ┌─────┐  HTTPS  ┌─────────────────────┐  HTTPS  ┌─────┐      │
│  │ 80  │────────►│      Caddy (proxy)  │────────►│ 443 │      │
│  └─────┘         └─────────────────────┘         └─────┘      │
└───────────────────────────────────────────────────────────────┘
                     ▲                                 ▲
                     │                                 │
                     │                                 │
                     ▼                                 ▼
          ┌───────────────────────┐       ┌─────────────────────────┐
          │  open‑webui (8080)    │       │        n8n (5678)       │
          │                       │       │ (uses PGvector for RAG) │     
          └───────────────────────┘       └─────────────────────────┘
               ▲                                       ▲     ▲
               │                                       │     │
               │─────────────┬─────────────────────────┘     │
               │             │                               │
               ▼             ▼                               ▼                     
┌─────────────────┐   ┌─────────────────┐  ┌─────────────────────┐
│     searXNG     │   │      ollama     │  │  postgres (pgvector)│
│   (WEbSearch    │   │   (11434, GPU)  │  │     (5432)          │
└─────────────────┘   └─────────────────┘  └─────────────────────┘
```

* All services share the `web` bridge network – they talk to each other by container name (`open-webui`, `n8n`, `ollama`).
* Vector database is isolated to n8n dbNet network, to further secure RAG data.
* Caddy reads the `Caddyfile` for routing rules and Let’s Encrypt settings.  
* The stack uses **bind mounts** for data persistence, so a container restart or image upgrade never wipes your settings or models.

---

## Getting Started  

> **Prerequisites** –  
> * Docker ≥ 20.10  
> * Docker‑Compose ≥ 1.29  
> * A server with a public IP and two A‑records:  
>   * `pal.example.com` → your server IP  
>   * `automation.example.com` → your server IP  

> **Hardware** – GPU‑accelerated container for Ollama is optional.  
> If you don’t have an NVIDIA GPU, just remove the `device_requests` block in `docker‑compose.yml` (or keep it; Docker will ignore it if no GPU is present).

### 1️⃣  Clone & Prepare

```bash
git clone https://github.com/DamienGreathouse/P.A.L.-Public-AI-Language-Assistant.git
cd P.A.L.-Public-AI-Language-Assistant
```

> The repo already contains a ready‑to‑run `docker-compose.yml`.

### 2️⃣  Edit `.env`

```bash
cp .env.example .env          # Copy the Template

# Ollama container
OLLAMA_PORT=11434        # Port to expose

# Domains
PAL_DOMAIN="pal.example.com"
AUTOMATION_DOMAIN="automation.example.com"
LETSENCRYPT_EMAIL="jdoe@example.com"

# Open‑WebUI container
OLLAMA_BASE_URL=http://ollama:${OLLAMA_PORT}

# POSTGRES pgVector
POSTGRES_USER=postgres
POSTGRES_PASSWORD=supersecret
POSTGRES_DB=knowledge
PGDATA=/var/lib/postgresql/data/pgdata
POSTGRES_URL=postgres://$POSTGRES_USER:$POSTGRES_PASSWORD@postgres:5432/$POSTGRES_DB

# Microsoft SSO for OPen-WebUI
MICROSOFT_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
MICROSOFT_CLIENT_SECRET=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
MICROSOFT_CLIENT_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
MICROSOFT_REDIRECT_URI=https://ai.ireadgood.com/oauth/microsoft/callback
OPENID_PROVIDER_URL=https://login.microsoftonline.com/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/v2.0/

# Enable the OneDrive integration feature globally
ENABLE_ONEDRIVE_INTEGRATION=true

# --- Business & SharePoint Configuration ---

# The Application (client) ID from your Business App Registration
ONEDRIVE_CLIENT_ID_BUSINESS="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# The Directory (tenant) ID from your Business App Registration
ONEDRIVE_SHAREPOINT_TENANT_ID="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"

# The root URL of your SharePoint instance
ONEDRIVE_SHAREPOINT_URL="https://your-tenant-name.sharepoint.com"# Enable the OneDrive integration feature globally
```

### 3️⃣  Create data directories

```bash
mkdir -p \
  ollama_models \
  openwebui_data \
  openwebui_logs \
  n8n_data \
  caddy_data \
  caddy_config \
  pg_data
```

### 4️⃣  Start the stack

```bash
docker compose up -d
```

> *If you already have the images locally, this will pull the newest ones.  
>  Caddy will request certificates automatically (first run may take a minute).*

### 5️⃣  Verify

```bash
curl -I https://pal.example.com
# Should return 200 and a TLS certificate from Let’s Encrypt

curl -I https://automation.example.com
# Should return 200
```

Open your browser to the URLs above – you should see Open WebUI and n8n dashboards.

---

## Configuration

| File | Purpose |
|------|---------|
| `.env` | Runtime variables for Docker & services |
| `Caddyfile` | Routing & TLS settings – uses `${PAL_DOMAIN}`, `${AUTOMATION_DOMAIN}`, `${LETSENCRYPT_EMAIL}` from `.env` |
| `docker-compose.yml` | Defines services, volumes, networks, GPU support |
| `scripts/` (optional) | Convenience helpers (`up.sh`, `down.sh`) |

### Caddyfile Highlights

```caddy
{
    email ${LETSENCRYPT_EMAIL}
}

${PAL_DOMAIN} {
    reverse_proxy open-webui:8080
}

${AUTOMATION_DOMAIN} {
    reverse_proxy n8n:5678
}
```

*No need to manually add `https://` – Caddy handles it.*

---

## Persistence & Backup

| Service | Mounted dir | What’s stored | Where |
|---------|-------------|---------------|-------|
| Ollama | `./ollama_models` | Model files | Host dir |
| Open WebUI | `./openwebui_data` | SQLite DB, settings | Host dir |
| Open WebUI | `./openwebui_logs` | Logs | Host dir |
| n8n | `./n8n_data` | Workflow data, credentials | Host dir |
| Caddy | `./caddy_data` | TLS certs, Caddy logs | Host dir |
| Caddy | `./caddy_config` | Optional Caddy overrides | Host dir |
| pgvector | `./pg_data` | vector database files | Host dir |

**Backup tip** – back up the host directories, not the containers:

```bash
tar czf backup-$(date +%F).tar.gz \
  ollama_models/ \
  openwebui_data/ \
  openwebui_logs/ \
  n8n_data/ \
  caddy_data/ \
  caddy_config/ \
  pg_data/
```

Restore by unpacking to the same paths and running `docker compose up -d` again.

---

## Upgrading & Maintenance

1. **Pull new images** (once per release):

   ```bash
   docker compose pull
   ```

2. **Restart** (to pick up the new images and apply any config changes):

   ```bash
   docker compose up -d
   ```

3. **Upgrade Ollama models** – just download the new model into `ollama_models/` (or let Ollama pull it automatically).

4. **Check logs**:

   ```bash
   docker compose logs -f caddy
   docker compose logs -f open-webui
   docker compose logs -f n8n
   ```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Caddy says “*error while reading config*” | Variable not set (e.g. `${LETSENCRYPT_EMAIL}`) | Verify that the variable exists in `.env` and that you didn’t typo the name. |
| HTTPS fails with “*certificate not trusted*” | Caddy didn’t finish obtaining a cert | Wait a few minutes, then check `docker compose logs caddy` for errors. |
| Open WebUI shows “*Connection refused*” | Ollama not running or port wrong | Make sure `OLLAMA_PORT` matches the port exposed in the `ollama` service and that the container is healthy (`docker compose ps`). |
| n8n logs show “*Failed to connect to database*” | Database file missing or corrupted | Re‑create `n8n_data/` or delete the corrupt SQLite file and restart. |
| GPU errors in Ollama | NVIDIA driver not installed or Docker not configured for GPU | Install the driver and run `docker run --gpus all nvidia/cuda:12.1-base nvidia-smi` to test. |

---

## License & Credits  

- **Open WebUI** – © Open WebUI Authors (MIT).  
- **n8n** – © n8n.io (MIT).  
- **Ollama** – © Ollama, Inc. (MIT).  
- **Caddy** – © Caddy Server (Apache‑2.0).
- **pgvector** – © pgvector Authors (MIT).
- **This stack** – Damien Greathouse (MIT).  

Feel free to fork, improve, and share! 🚀

---
