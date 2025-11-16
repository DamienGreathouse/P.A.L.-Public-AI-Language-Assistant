# P.A.L.-Public-AI-Language-Assistant
Deliver a lightweight, self‑hostable, privacy‑first language model stack that any community, small business or individual can run locally and keep 100 % of the data on their own hardware.

**Description**  
> **P.A.L.** is a modular, open‑source AI platform that lets anyone run a state‑of‑the‑art language model on their own hardware—no cloud, no data‑outs.  Built on top of Ollama, pgVector, n8n, and Open‑WebUI, the stack is packaged in a single Docker‑Compose file that you can **easily update** (pull a new image, rebuild a Dockerfile, or switch tags) and the system will automatically restart with the new version.  It is accessed through a sleek Caddy‑based reverse proxy and can be refreshed with a single `docker compose pull` + `docker compose up -d`.  P.A.L. empowers local knowledge workers, educators, and hobbyists to have a private, always‑on AI assistant that respects their privacy while staying cutting‑edge.

**Summary**  
1. **Self‑hosted, zero‑cloud** – run everything on your own server or laptop, no external API calls.  
2. **Easy to update, self‑patch** – a simple Docker pull and compose‑up keeps the stack secure without manual edits.  
3. **Community‑friendly** – all code is on GitHub, all docs are Markdown, and the Docker stack can be pulled and launched in under five minutes. 

### Security benefits

> **Security at the core**  
> P.A.L. is engineered with *security‑by‑design* in mind.  
> • **Local data residency** – all user data stays on the host; there are no outbound network calls to cloud providers that could expose your conversations.  
> • **Network isolation** – the Postgres/pgVector database is confined to a private `dbnet` bridge that only the n8n orchestrator can reach, effectively cutting off attackers who compromise the public web tier from accessing the data layer.  
> • **Minimal attack surface** – the only services exposed to the internet are the reverse‑proxy (Caddy) and the n8n and Open‑WebUI UIs. IP filter is implemented in Caddy file.
> • **Immutable image updates** – by pinning tags in the Compose file and using a lightweight updater (e.g., Watchtower or a cron job), you avoid “unknown‑source” updates and retain full control over which releases are deployed.  
> • **Transparent provenance** – every component is open‑source and publicly audited; pull requests and vulnerability scans are part of the normal development cycle, so you can verify the integrity of the code before you run it.  
> • **Runtime patching** – optional post‑restart commands (e.g., `apt‑get upgrade –y`) ensure that the underlying OS packages stay current without needing a full rebuild, protecting against known CVEs.  
> 
> Together, these measures give you a **secure, self‑contained AI assistant** that you can trust to keep your data confidential while still delivering state‑of‑the‑art language capabilities.
