# FreeLLMAPI, Open WebUI & SearXNG Setup Guide

![FreeLLMAPI Banner](FreeLLMAPI_plus_Open_WebUI_banner.png)

A complete step-by-step guide to deploying **FreeLLMAPI**, **Open WebUI**, and **SearXNG** together via a single Docker Compose stack, including firewall hardening, SSH/VPN access, mobile app connection, and backup routines. This all runs on an Oracle Cloud Free Tier instance, though you can use any Linux host.

## 🚀 Why FreeLLMAPI?

**FreeLLMAPI** is an open-source (MIT), local-first AI gateway that aggregates free-tier APIs from dozens of LLM providers (up to **87 providers** and **1,300+ free model endpoints**, totaling over **10 billion free tokens per month**) behind a single, unified OpenAI-compatible `/v1` endpoint.

Instead of constantly hitting rate limits on a single provider or paying expensive monthly API bills for personal projects and coding agents, FreeLLMAPI pools your free API keys together and handles routing and failover automatically.

---

### ✨ Key Benefits

* **Massive Combined Free Quota:** Turns small, isolated free tiers (from providers like Google, Groq, Cerebras, Mistral, and more) into a massive, continuous inference pool.
* **Single OpenAI-Compatible Endpoint:** Connect any tool—such as Open WebUI, Claude Code, Cline, Aider, or custom scripts—using one local `/v1` URL and a single unified API key.
* **Smart Routing & Silent Failover:** Automatically routes requests based on reliability, speed, and intelligence scores. If one provider hits a rate limit (`429`) or goes down, FreeLLMAPI seamlessly fails over to the next available provider without interrupting your workflow.
* **Private Web Search via SearXNG:** Pairs seamlessly with a self-hosted SearXNG metasearch instance inside the same Docker network so Open WebUI can ground responses in live web results without relying on paid search APIs.
* **Local-First & Encrypted Privacy:** Runs directly on your own machine (or a lightweight Raspberry Pi / VPS) with a tiny ~40 MB memory footprint. Your upstream API keys are stored locally in an encrypted SQLite database, and requests go straight from your machine to the providers—no middleman proxy reading your prompts.
* **Self-Updating Model Catalog:** Automatically pulls a signed catalog of active free endpoints, rate limits, and context windows so you don't have to manually track API changes.
* **Custom Local & Remote Endpoints:** Easily plug in local models (Ollama, LM Studio, llama.cpp, vLLM) alongside cloud free tiers.

---

### ⚖️ Why FreeLLMAPI is Better Than OmniRoute

While both **FreeLLMAPI** and **OmniRoute** are local, OpenAI-compatible routing gateways, FreeLLMAPI takes a cleaner, safer, and more lightweight approach for aggregating free LLM access:

| Feature | FreeLLMAPI | OmniRoute |
| :--- | :--- | :--- |
| **Core Philosophy** | Purpose-built specifically for **official free-tier API keys**. | Heavy multi-purpose proxy mixing API keys, paid subscriptions, and OAuth/session scraping. |
| **Account Safety & ToS** | **Safer API-key pooling:** Uses standard developer API keys rather than hijacking or scraping consumer web/CLI subscription logins. | Relies heavily on wrapping consumer subscription logins (e.g., Claude Pro, Codex) with header obfuscation/identity rotation, risking account bans. |
| **Resource Footprint** | **Ultra-lightweight (~40 MB RAM):** Runs cleanly on any Node 20+ environment, Docker container, or Raspberry Pi. | **Heavyweight architecture:** Bundles a full Next.js dashboard, MCP servers, A2A protocols, and embedded sub-processes. |
| **Setup & Maintenance** | **Zero-maintenance catalog:** Pulls a curated, signed catalog of free endpoints and rate limits automatically. | Requires heavy manual configuration of custom combos, 13+ balancing strategies, and per-model circuit breakers. |
| **Security at Rest** | Encrypts all stored provider keys inside a local SQLite database out of the box. | Primarily focused on complex multi-tier routing policies and local/cloud state sync. |

> **TL;DR:** Choose **FreeLLMAPI** if you want a lightweight, fast, and clean gateway that maximizes **10B+ free API tokens/month** with zero bloat and automatic failover. OmniRoute is overly complex if you aren't trying to juggle multiple paid consumer subscriptions or run an enterprise-style routing stack.


> **Security Note:** FreeLLMAPI is single-user by design and protected only by your dashboard login and unified bearer token. Avoid exposing ports `3000`, `3001`, or `8080` directly to the public internet. Access the instance via SSH port forwarding, a private VPN (such as WireGuard, AmneziaWG, or Tailscale), or a reverse proxy with TLS and authentication.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Connect and Update the OS](#step-1-connect-and-update-the-os)
- [Step 2: Install Docker & Docker Compose](#step-2-install-docker--docker-compose)
- [Step 3: Configure Firewall & Network Access](#step-3-configure-firewall--network-access)
- [Step 4: Deploy FreeLLMAPI, Open WebUI & SearXNG with Docker Compose](#step-4-deploy-freellmapi-open-webui--searxng-with-docker-compose)
- [Step 5: Enable JSON Output in SearXNG](#step-5-enable-json-output-in-searxng)
- [Step 6: Initial Setup & Adding Provider Keys in FreeLLMAPI](#step-6-initial-setup--adding-provider-keys-in-freellmapi)
- [Step 7: Test the OpenAI-Compatible Endpoint](#step-7-test-the-openai-compatible-endpoint)
- [Step 8: Configure Open WebUI (Gateway & SearXNG Web Search)](#step-8-configure-open-webui-gateway--searxng-web-search)
- [Step 9: Install and Connect Open MobileUI on Your Phone](#step-9-install-and-connect-open-mobileui-on-your-phone)
- [Maintenance & Operations](#maintenance--operations)

---

## Prerequisites

- Any Linux server (e.g., Ubuntu 24.04 LTS on ARM64 or x86_64) with Docker and Docker Compose installed.
- SSH access to the instance (e.g., private IP `10.0.0.10` over WireGuard/AmneziaWG/VCN, or via its assigned public IP).
- At least one free-tier API key from a supported upstream provider (e.g., Google AI Studio, Groq, Cerebras, Mistral, OpenRouter).

---

## Step 1: Connect and Update the OS

SSH into your OCI instance (or any instance you're using), update the system packages and install the necessary utilities:

```bash
ssh ubuntu@10.0.0.10
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git openssl ca-certificates
```

*(Optional)* If you are running a smaller `VM.Standard.E2.1.Micro` (`1 GB` RAM) instance, add a 2 GB swap file to prevent out-of-memory errors:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Step 2: Install Docker & Docker Compose

Install Docker Engine and the Docker Compose plugin using the official convenience script:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

Verify the installation:

```bash
docker --version
docker compose version
```

---

## Step 3: Configure Firewall & Network Access

Oracle Cloud instances enforce both a **VCN Security List** (cloud-level) and local **`iptables`** rules (OS-level). Choose the access option that matches your network setup.

### Option A: Recommended — Access via SSH Tunnel or Private VPN (No Public Ports Opened)

Keep ports `3000` (Open WebUI), `3001` (FreeLLMAPI), and `8080` (SearXNG) closed to the public internet and forward them securely over SSH from your local workstation when needed:

```bash
ssh -L 3000:127.0.0.1:3000 -L 3001:127.0.0.1:3001 -L 8080:127.0.0.1:8080 ubuntu@10.0.0.10
```

You can then open `http://localhost:3000` (Open WebUI) or `http://localhost:3001` (FreeLLMAPI) in your local browser.

### Option B: Internal VCN / VPN Subnet Access (`10.0.0.10`)

If you are connecting directly to your private VPN IP (`10.0.0.10`) over WireGuard, AmneziaWG, Tailscale, or an internal VCN subnet:

1. **OCI Console:** Ensure your VCN **Security List** (or Network Security Group) allows TCP ingress on ports `3000`, `3001`, and optionally `8080` only from your trusted CIDR (e.g., `10.0.0.0/16` or your VPN subnet).
2. **OS Firewall (`iptables`):** OCI Ubuntu images block non-SSH ports by default. Allow ports `3000`, `3001`, and `8080` for your trusted subnet and persist the rules:

```bash
sudo iptables -I INPUT 2 -m state --state NEW -p tcp -s 10.0.0.0/8 --dport 3000 -j ACCEPT
sudo iptables -I INPUT 2 -m state --state NEW -p tcp -s 10.0.0.0/8 --dport 3001 -j ACCEPT
sudo iptables -I INPUT 2 -m state --state NEW -p tcp -s 10.0.0.0/8 --dport 8080 -j ACCEPT
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## Step 4: Deploy FreeLLMAPI, Open WebUI & SearXNG with Docker Compose

Create a dedicated stack directory along with local folders for SearXNG configuration and cache data:

```bash
mkdir -p ~/freellmapi/searxng/config ~/freellmapi/searxng/cache
cd ~/freellmapi
```

Generate a persistent 64-character hex `ENCRYPTION_KEY` (used for AES-256-GCM encryption of your upstream API keys at rest):

```bash
openssl rand -hex 32
```
> **Note:** Copy the output to use as the `ENCRYPTION_KEY` value in the YAML file below.

Create and edit `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Paste the following unified configuration. Replace `<copy output from previous step here>` with your generated key, and adjust the SearXNG bind IP (`10.0.0.10:8080:8080`) if your private VPN IP differs:

```yaml
services:
  freellmapi:
    image: ghcr.io/tashfeenahmed/freellmapi:latest
    container_name: freellmapi
    restart: unless-stopped
    environment:
      - PORT=3001
      # Encrypts your stored provider keys. Back this value up!
      # To generate your own: openssl rand -hex 32
      - ENCRYPTION_KEY=<copy output from previous step here>
    ports:
      # Exposes FreeLLMAPI on your server interfaces (restrict via firewall/VPN).
      - "0.0.0.0:3001:3001"
    volumes:
      - freellmapi-data:/app/server/data

  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      # Bound strictly to your private VPN IP so SearXNG is never public.
      # Change 10.0.0.10 to 127.0.0.1 if you only use SSH tunneling.
      - "10.0.0.10:8080:8080"
    volumes:
      - ./searxng/config:/etc/searxng:rw
      - ./searxng/cache:/var/cache/searxng:rw
    environment:
      - SEARXNG_BASE_URL=http://10.0.0.10:8080/
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      # Pre-fills the FreeLLMAPI connection URL. The unified API key is added
      # afterwards in Open WebUI (Admin Panel > Settings > Connections).
      - OPENAI_API_BASE_URL=http://freellmapi:3001/v1
      # Pre-configures SearXNG internal container DNS for Web Search
      - ENABLE_RAG_WEB_SEARCH=True
      - RAG_WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search
    volumes:
      - open-webui-data:/app/backend/data
    depends_on:
      - freellmapi
      - searxng

volumes:
  freellmapi-data:
  open-webui-data:
```

> **Why all three communicate effortlessly:** Because `freellmapi`, `searxng`, and `open-webui` are defined in the same `docker-compose.yml`, Docker automatically places them on a shared internal bridge network. `open-webui` reaches FreeLLMAPI via `http://freellmapi:3001/v1` and SearXNG via `http://searxng:8080/search` without any port conflicts (`open-webui` maps host port `3000` to its internal `8080`, while `searxng` uses its own internal `8080`).

Start all three containers in detached mode:

```bash
docker compose up -d
```

Verify that all containers are running:

```bash
docker compose ps
docker compose logs -f freellmapi
```

*(Press `Ctrl+C` to exit the log stream.)*

---

## Step 5: Enable JSON Output in SearXNG

On its first boot, SearXNG automatically generates `~/freellmapi/searxng/config/settings.yml`. By default, SearXNG only enables `html` output and blocks `json` queries with a `403 Forbidden` error—which will prevent Open WebUI from retrieving search results.

1. Open the generated configuration file:
   ```bash
   nano ~/freellmapi/searxng/config/settings.yml
   ```
2. Find the `search:` section and add `json` under `formats`:
   ```yaml
   search:
     formats:
       - html
       - json
   ```
3. *(Optional)* Verify that `server.secret_key` has a unique random value (SearXNG normally generates one on first run).
4. Restart SearXNG to apply the change:
   ```bash
   cd ~/freellmapi && docker compose restart searxng
   ```
5. Verify that Open WebUI can query SearXNG over the internal Docker network and receive JSON:
   ```bash
   docker exec open-webui curl -s "http://searxng:8080/search?q=ubuntu+24.04&format=json" | head -c 300
   ```
   You should see JSON search results printed to your terminal.

---

## Step 6: Initial Setup & Adding Provider Keys in FreeLLMAPI

1. Open the FreeLLMAPI web dashboard in your browser:
   - **Via SSH tunnel:** `http://localhost:3001`
   - **Via VPN / private network:** `http://10.0.0.10:3001`
2. Create your initial admin account (email and password) on the first-run setup screen. The one-time setup key can be found in the logs:
   ```bash
   cd ~/freellmapi && docker compose logs freellmapi
   ```
3. Navigate to the **Chat models** page and add your free-tier Provider API keys (e.g., Google AI Studio, Groq, Cerebras, Mistral). FreeLLMAPI will validate each key and display its health status.
4. Configure your preferred **Fallback Chain** and routing strategy.
5. Copy your unified bearer token (`freellmapi-...`) from **Keys** -> **Unified API Key**.

---

## Step 7: Test the OpenAI-Compatible Endpoint

Test the `/v1/chat/completions` endpoint using `curl` (replace `freellmapi-your-unified-key` with your token):

```bash
curl http://10.0.0.10:3001/v1/chat/completions \
  -H "Authorization: Bearer freellmapi-your-unified-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "auto",
    "messages": [{"role": "user", "content": "Hello from Oracle Cloud Free Tier!"}]
  }'
```

---

## Step 8: Configure Open WebUI (Gateway & SearXNG Web Search)

### 8.1 Connect Open WebUI to FreeLLMAPI

1. Open your browser and navigate to `http://10.0.0.10:3000` (or `http://localhost:3000` over an SSH tunnel).
2. Click **Get Started** and create your Open WebUI admin account.
3. Go to **Admin Panel** -> **Settings** -> **Connections**.
4. Edit the existing OpenAI API connection (pre-filled as `http://freellmapi:3001/v1` from `docker-compose.yml`).
5. Paste your unified FreeLLMAPI bearer token (`freellmapi-...`) into the **API Key** field and click **Save**. Your FreeLLMAPI models will now appear in the model selector.

### 8.2 Verify & Tune SearXNG Web Search in Open WebUI

1. Go to **Admin Panel** -> **Settings** -> **Web Search**.
2. Confirm that **Enable Web Search** is toggled **On**, **Web Search Engine** is set to `searxng`, and **SearXNG Query URL** is `http://searxng:8080/search`.
3. Adjust **Search Result Count** (e.g., `3`–`5`) and **Concurrent Requests** (`1`–`2` is ideal on Oracle Cloud Free Tier to avoid upstream rate limits).
4. *(Optional ARM64 / Low-RAM Tip)* If full-page scraping and embedding slows down responses, toggle **Bypass Web Loader** or **Bypass Embedding and Retrieval** on so Open WebUI feeds SearXNG's snippets directly to the model.
5. Click **Save**, open a new chat, toggle the **Web Search** globe icon on in the prompt bar, and ask a current-events question to verify citations appear.

---

## Step 9: Install and Connect Open MobileUI on Your Phone

**Open MobileUI** is a dedicated native mobile client for Open WebUI (available on iOS and Android) that syncs your chats, models, and SearXNG web-search toggle directly with your self-hosted Open WebUI server.

### 9.1 Connect Your Phone to Your Private Network

Because your Open WebUI instance (`10.0.0.10:3000`) is intentionally kept off the public internet, ensure your phone is connected to your private VPN before launching the app:

* **WireGuard / AmneziaWG:** Activate your VPN tunnel on your phone so `10.0.0.10` is routable.
* **Tailscale:** Ensure the Tailscale app is active on both your Oracle Cloud server and your phone.
* *(Optional)* Verify reachability first by opening `http://10.0.0.10:3000` in your phone's mobile browser.

### 9.2 Install Open MobileUI

* **Android:** Install **Open MobileUI** from the Google Play Store (or obtain the latest APK from the official GitHub releases page).
* **iOS:** Install **Open MobileUI** from the Apple App Store.

### 9.3 Link Open MobileUI to Your Server

1. Open **Open MobileUI** on your phone.
2. On the server connection screen, enter your Open WebUI URL:
   ```text
   http://10.0.0.10:3000
   ```
   *(Do not add a trailing slash `/`; if you put Open WebUI behind an HTTPS reverse proxy, use `https://your-domain.example` instead.)*
3. Tap **Connect** and sign in with the Open WebUI email and password you created in **Step 8.1**.
4. Select your preferred FreeLLMAPI model (such as `auto`) from the top model dropdown and toggle **Web Search** whenever you want SearXNG-backed answers on the go.

> **Troubleshooting Mobile Connection:** If Open MobileUI fails to reach `http://10.0.0.10:3000`, confirm that (1) your mobile VPN client routes `10.0.0.0/8` (or `10.0.0.10/32`) through the tunnel, and (2) `iptables` on the Oracle Cloud host permits TCP port `3000` from your VPN subnet (see **Step 3, Option B**).

---

## Maintenance & Operations

### Updating the Stack (FreeLLMAPI, Open WebUI & SearXNG)

Pull the latest multi-arch images and recreate the containers (your Docker volumes and `./searxng/config` remain intact):

```bash
cd ~/freellmapi
docker compose pull
docker compose up -d
```

### Backing Up Your Configuration

Your encrypted database is stored in the `freellmapi-data` Docker volume (`/app/server/data`). Without the original `ENCRYPTION_KEY`, the stored provider keys cannot be decrypted, so back up both the key and the data.

The `ENCRYPTION_KEY` is set in your `docker-compose.yml`, so back up that file alongside your SearXNG configuration:

```bash
cd ~/freellmapi
mkdir -p ~/backup
cp docker-compose.yml ~/backup/docker-compose.yml.backup
cp -r searxng/config ~/backup/searxng-config
```

To avoid copying a half-written SQLite database, stop `freellmapi` briefly while copying its data folder:

```bash
docker compose stop freellmapi
docker compose cp freellmapi:/app/server/data ~/backup/freellmapi-data
docker compose start freellmapi
```

Optionally, also back up your Open WebUI chats and settings:

```bash
docker compose cp open-webui:/app/backend/data ~/backup/open-webui-data
```

> **Keep the backup private.** `docker-compose.yml` contains your encryption key and `searxng/config/settings.yml` contains your SearXNG secret key, so store the backup somewhere safe and never commit it to a public GitHub repository.

### Resetting a Forgotten FreeLLMAPI Dashboard Password

1. Click **Forgot password?** on the FreeLLMAPI login page and select **Send reset code**.
2. Retrieve the one-time reset code from the container logs:
   ```bash
   cd ~/freellmapi && docker compose logs --tail=20 freellmapi
   ```
3. Enter the printed code in the web UI to set a new password.

---
