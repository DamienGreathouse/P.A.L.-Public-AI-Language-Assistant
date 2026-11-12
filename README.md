# P.A.L. Public AI Language Assistant

Deliver a lightweight, self-hostable, privacy-first language model stack that any community, small business, or individual can run locally and keep 100% of the data on their own hardware.

![P.A.L. Public AI Language Assistant.](https://github.com/DamienGreathouse/P.A.L.-Public-AI-Language-Assistant/blob/cb54c75dcdca627c6efd234d498cedc719ba73ae/Assests/Public-AI-Language-Assistant.png)

## Description

**P.A.L.** is a modular, open-source AI platform that lets anyone run a state-of-the-art language model on their own hardware — no cloud, no data-outs. Built on top of **vLLM**, **Open WebUI**, **Kokoro**, **Docling**, and **searXNG**, the stack is packaged in a single Docker-Compose file that you can **easily update** (pull a new image, rebuild a Dockerfile, or switch tags) and the system will automatically restart with the new version. It is accessed through a Caddy-based reverse proxy that obtains certificates via Cloudflare and can be refreshed with a single `docker compose pull` + `docker compose up -d`. P.A.L. empowers local knowledge workers, educators, and hobbyists to have a private, always-on AI assistant that respects their privacy while staying cutting-edge.

## Summary

1. **Self-hosted, zero-cloud** — run everything on your own server or workstation, no external API calls.
2. **Easy to update, self-patch** — a simple Docker pull and compose-up keeps the stack current without manual edits.
3. **Community-friendly** — all code is on GitHub, all docs are Markdown, and the Docker stack can be pulled and launched in a few minutes.

## Security at the Core

P.A.L. is engineered with *security-by-design* in mind.

- **Local data residency** — all user data stays on the host; the LLM inference, embeddings, TTS, and document parsing all run on your own hardware with no outbound calls to cloud model providers.
- **Single public entry point** — only the Caddy reverse proxy is exposed to the internet; the vLLM, Kokoro, and Docling services are reached only through Caddy (or the host, if you open their ports).
- **Minimal attack surface** — internal services such as the main vLLM container are not published to the host at all, so a compromised web tier cannot reach the model runtime directly.
- **Immutable image updates** — by pinning tags in the Compose file and using a controlled updater, you avoid "unknown-source" updates and retain full control over which releases are deployed.
- **Transparent provenance** — every component is open-source and publicly audited; pull requests and vulnerability scans are part of the normal development cycle, so you can verify the integrity of the code before you run it.
- **Runtime patching** — optional post-restart commands (e.g., `apt-get upgrade -y`) keep the underlying OS packages current without a full rebuild, protecting against known CVEs.

Together, these measures give you a **secure, self-contained AI assistant** that you can trust to keep your data confidential while still delivering state-of-the-art language capabilities.

---

## Table of Contents

| Section | Description |
|---------|-------------|
| [What it is](#what-it-is) | Overview of the stack |
| [Architecture](#architecture) | Services & networking |
| [Prerequisites](#prerequisites) | Hardware & software requirements |
| [Getting Started](#getting-started) | Quick-start steps |
| [Hugging Face Setup](#hugging-face-setup) | Install the `hf` CLI and download models |
| [Configuration](#configuration) | `.env`, `Caddyfile`, and other files |
| [Persistence & Backup](#persistence-and-backup) | Where data lives |
| [Upgrading & Maintenance](#upgrading--maintenance) | Keeping the stack up-to-date |
| [Troubleshooting](#troubleshooting) | Common issues & fixes |
| [License & Credits](#license--credits) | Who built this? |

---

## What It Is

P.A.L. Public AI Language Assistant is a **single-file Docker-Compose** stack that gives you:

| Service | Image | Purpose |
|---------|-------|---------|
| **Open WebUI** | `ghcr.io/open-webui/open-webui:main` | Front-end chat UI; connects to vLLM, embeddings, TTS, and RAG. |
| **vLLM** | `vllm/vllm-openai:latest` | Main LLM runtime (GPU) — serves `Inferact/Qwen3.8-27B-NVFP4`. |
| **vLLM Embed** | `vllm/vllm-openai:latest` | Embedding runtime — serves `Qwen/Qwen3-Embedding-0.6B` for RAG. |
| **vLLM Gemma** | `vllm/vllm-openai:latest` | Alternative LLM runtime — serves `nvidia/Gemma-4-26B-A4B-NVFP4`. |
| **Kokoro** | `ghcr.io/remsky/kokoro-fastapi-gpu:latest` | Text-to-speech (TTS) for Open WebUI audio. |
| **Docling** | `quay.io/docling-project/docling-serve:latest` | Document parsing for RAG (PDFs, docs, images). |
| **searXNG** | `searxng/searxng:latest` | Private web search for Open WebUI / RAG. |
| **Caddy** | `ghcr.io/caddybuilds/caddy-cloudflare:latest` | Reverse proxy; obtains certificates via Cloudflare. |
| **Hugging Face** | (host `hf` CLI) | Downloads models into a single shared cache to save space. |

All vLLM containers share one Hugging Face cache mounted from `/opt/llm-models/huggingface` on the host. The stack is reached through **Caddy** on your domain(s); the main vLLM service is **not** exposed to the host, and only Open WebUI is intended as the public endpoint.

---

## Architecture  

```
Public Internet │ HTTPS (443) / HTTP (80) ▼ ┌───────────────┐ │ Caddy (CF) │ ghcr.io/caddybuilds/caddy-cloudflare │ reverse_proxy│ └───────┬───────┘ ▼ ┌───────────────┐ │ open-webui │ ghcr.io/open-webui/open-webui:main │ (named volume)│ └──────┬────────┘ ┌───────────┬──┴───────────┬──────────────┐ ▼ ▼ ▼ ▼
┌────────┐ ┌─────────┐ ┌───────────┐ ┌──────────────┐
│ vllm │ │ vllm- │ │ vllm- │ │ kokoro- │
│ Qwen │ │ embed │ │ gemma-4 │ │ fastapi │
│ :8000 │ │ :8000 │ │ :8000 │ │ :8880 (TTS) │
└────────┘ └─────────┘ └───────────┘ └──────────────┘
all three read models from /opt/llm-models/huggingface

searxng (web search) ──────► open-webui (RAG web search)
docling (:5001) ──────► open-webui (RAG document parser)
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

# Setting Up Hugging Face on a DGX Spark

This guide walks through installing and configuring the Hugging Face CLI (`hf`) on a DGX Spark, pointing the model cache at a dedicated directory, and downloading a model.
---
- **User account:** `itadmin`
- **Hugging Face home:** `/opt/llm-models/huggingface`

---

# Point Hugging Face at that directory (environment variables)
Edit ~/.bashrc and add (or correct) the following lines:
```bash
export HF_HOME=/opt/llm-models/huggingface
export HF_HUB_CACHE=/opt/llm-models/huggingface
export HF_DATASETS_CACHE=/opt/llm-models/huggingface
export PATH="$HOME/.local/bin:$PATH"
```
## 1. Create the Hugging Face directory

```bash
sudo mkdir -p /opt/llm-models/huggingface
sudo chmod 755 /opt/llm-models/huggingface
sudo chown -R itadmin:itadmin /opt/llm-models/huggingface
```
### 1️⃣  Clone & Prepare

```bash
git clone https://github.com/DamienGreathouse/P.A.L.-Public-AI-Language-Assistant.git
cd P.A.L.-Public-AI-Language-Assistant
```
```bash
cp nvidia-docker-compose.yaml docker-compose.yaml     # Copy correct docker-compose file use the amd file for AMD systems
```

> The repo already contains a ready‑to‑run `docker-compose.yml`.

### 2️⃣  Edit `.env`

```bash
cp env.example .env          # Copy the Template
```
```bash
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
