# FreeLLMAPI & Open WebUI Setup Guide

![FreeLLMAPI Banner](FreeLLMAPI_plus_Open_WebUI.png)

A complete step-by-step guide to deploying **FreeLLMAPI** via Docker Compose, including firewall hardening, SSH/VPN access, and backup routines. This all runs on a Oracle Cloud Free Tier instance, but you can use any other.

## 🚀 Why FreeLLMAPI?

**FreeLLMAPI** is an open-source (MIT), local-first AI gateway that aggregates free-tier APIs from dozens of LLM providers (up to **87 providers** and **1,300+ free model endpoints**, totaling over **10 billion free tokens per month**) behind a single, unified OpenAI-compatible `/v1` endpoint.

Instead of constantly hitting rate limits on a single provider or paying expensive monthly API bills for personal projects and coding agents, FreeLLMAPI pools your free API keys together and handles routing and failover automatically.

---

### ✨ Key Benefits

* **Massive Combined Free Quota:** Turns small, isolated free tiers (from providers like Google, Groq, Cerebras, Mistral, and more) into a massive, continuous inference pool.
* **Single OpenAI-Compatible Endpoint:** Connect any tool—such as Open WebUI, Claude Code, Cline, Aider, or custom scripts—using one local `/v1` URL and a single unified API key.
* **Smart Routing & Silent Failover:** Automatically routes requests based on reliability, speed, and intelligence scores. If one provider hits a rate limit (`429`) or goes down, FreeLLMAPI seamlessly fails over to the next available provider without interrupting your workflow.
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


> **Security Note:** FreeLLMAPI is single-user by design and protected only by your dashboard login and unified bearer token. Avoid exposing port `3001` directly to the public internet. Access the instance via SSH port forwarding, a private VPN (such as WireGuard or Tailscale), or a reverse proxy with TLS and authentication.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Connect and Update the OS](#step-1-connect-and-update-the-os)
- [Step 2: Install Docker & Docker Compose](#step-2-install-docker--docker-compose)
- [Step 3: Configure Firewall & Network Access](#step-3-configure-firewall--network-access)
- [Step 4: Deploy FreeLLMAPI with Docker Compose](#step-4-deploy-freellmapi-with-docker-compose)
- [Step 5: Initial Setup & Adding Provider Keys](#step-5-initial-setup--adding-provider-keys)
- [Step 6: Test the OpenAI-Compatible Endpoint](#step-6-test-the-openai-compatible-endpoint)
- [Step 7: Setup Open WebUI (FrontEnd chat) to use the FreeLLMAPI Gateway](#step-7-setup-open-webui-frontend-chat-to-use-the-freellmapi-gateway)
- [Maintenance & Operations](#maintenance--operations)

---

## Prerequisites

- Any Linux server (e.g. Ubuntu 24.04 LTS) with Docker and Docker Compose installed.
- SSH access to the instance (e.g., private IP `10.0.0.10` over VPN/VCN, or via its assigned public IP).
- At least one free-tier API key from a supported upstream provider (e.g., Google AI Studio, Groq, Cerebras, Mistral, OpenRouter).

---

## Step 1: Connect and Update the OS

SSH into your OCI instance (or any instance you're using), update the system packages and install the necessary ones:

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

By default, FreeLLMAPI binds to `127.0.0.1:3001`. Keep port `3001` closed to the public internet and forward it securely over SSH from your local workstation:

```bash
ssh -L 3001:127.0.0.1:3001 ubuntu@10.0.0.10
```

You can then open `http://localhost:3001` in your local browser.

### Option B: Internal VCN / VPN Subnet Access (`10.0.0.10:3001`)

If you are connecting directly to `10.0.0.10` over a private network (such as WireGuard, Tailscale, or an internal VCN subnet):

1. **OCI Console:** Ensure your VCN **Security List** (or Network Security Group) allows TCP ingress on port `3001` from your trusted CIDR (e.g., `10.0.0.0/16` or your VPN subnet).
2. **OS Firewall (`iptables`):** OCI Ubuntu images block non-SSH ports by default. Allow port `3001` for your trusted subnet and persist the rule:

```bash
sudo iptables -I INPUT 2 -m state --state NEW -p tcp -s 10.0.0.0/8 --dport 3001 -j ACCEPT
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## Step 4: Deploy FreeLLMAPI with Docker Compose

Create a dedicated directory for FreeLLMAPI:

```bash
mkdir -p ~/freellmapi && cd ~/freellmapi
```

Generate a persistent 64-character hex `ENCRYPTION_KEY` (used for AES-256-GCM encryption of your upstream API keys at rest):

```bash
openssl rand -hex 32
```
> **Note:** Copy the output to use as the ENCRYPTION_KEY value in underneath yaml file.

```bash
nano docker-compose.yml
```

Paste underneath yaml code. Don't forget to replace the ENCRYPTION_KEY!

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
      # Local access only. Use "3001:3001" to expose on LAN/VPN (trusted networks only).
      - "0.0.0.0:3001:3001"
    volumes:
      - freellmapi-data:/app/server/data

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      # Pre-fills the connection URL. The API key is added afterwards in the
      # Open WebUI interface (Admin Panel > Settings > Connections), because
      # FreeLLMAPI only creates its unified key after it has started.
      - OPENAI_API_BASE_URL=http://freellmapi:3001/v1
    volumes:
      - open-webui-data:/app/backend/data
    depends_on:
      - freellmapi

volumes:
  freellmapi-data:
  open-webui-data:
```

Start the container in detached mode:

```bash
docker compose up -d
```

Verify that the container is running and healthy:

```bash
docker compose ps
docker compose logs -f freellmapi
```

*(Press `Ctrl+C` to exit the log stream.)*

---

## Step 5: Initial Setup & Adding Provider Keys

1. Open the FreeLLMAPI web dashboard in your browser:
   - **Via SSH tunnel:** `http://localhost:3001`
   - **Via VPN / private network (`HOST_BIND=0.0.0.0`):** `http://<your-private-server-ip>:3001`
2. Create your initial admin account (email and password) on the first-run setup screen. The setup key can be found when running `docker compose logs -f freellmapi`
3. Navigate to the **Chat models** page and add your free-tier Provider API keys (e.g., Google AI Studio, Groq, Cerebras, Mistral). FreeLLMAPI will validate each key and display its health status.
4. Configure your preferred **Fallback Chain** and routing strategy.
5. Copy your unified bearer token (`freellmapi-...`) from the **Keys** -> **Unified API Key** page.

---

## Step 6: Test the OpenAI-Compatible Endpoint

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

## Step 7: Setup Open WebUI (FrontEnd chat) to use the FreeLLMAPI Gateway

1. Open your browser and navigate to `http://<your-private-server-ip>:3000`.
2. Go to **Admin Panel** -> **Settings** -> **Connections**.
3. Check the existing connection (created by the yaml file in step 4) and ensure the OpenAI API Base URL is set to `http://<your-private-server-ip>:3001/v1`.
4. On the FreeLLMAPI web page, copy your unified FreeLLMAPI bearer token `freellmapi-...` from the FreeLLMAPI Keys page: **Keys** -> **Unified API key**
5. Paste this key in Open WebUI by editing the existing connection, then save. Your FreeLLMAPI models should now show up in the model dropdown.

---

## Maintenance & Operations

### Updating FreeLLMAPI

Pull the latest multi-arch image from GitHub Container Registry and recreate the container (your `freellmapi-data` SQLite volume remain intact):

```bash
cd ~/freellmapi
docker compose pull
docker compose up -d
```

### Backing Up Your Configuration

Your encrypted database is stored in the `freellmapi-data` Docker volume (`/app/server/data`). Without the original `ENCRYPTION_KEY`, the stored provider keys cannot be decrypted, so back up both the key and the data.

The `ENCRYPTION_KEY` is set in your `docker-compose.yml`, so back that file up too. Run these commands from the folder that contains it:

```bash
cd ~/freellmapi
mkdir -p ~/backup
cp docker-compose.yml ~/backup/docker-compose.yml.backup
```

This copies the whole data folder, including the SQLite database and any extra files SQLite keeps next to it. To avoid copying a half-written database, stop the container first and start it again afterwards:

```bash
docker compose stop freellmapi
docker compose cp freellmapi:/app/server/data ~/backup/freellmapi-data
docker compose start freellmapi
```

Optionally, also back up your Open WebUI chats and settings:

```bash
docker compose cp open-webui:/app/backend/data ~/backup/open-webui-data
```

> **Keep the backup private.** `docker-compose.yml` contains your encryption key, so store the backup somewhere safe and never commit it to a public GitHub repository.

### Resetting a Forgotten Dashboard Password

1. Click **Forgot password?** on the login page and select **Send reset code**.
2. Retrieve the one-time reset code from the container logs:
   ```bash
   cd ~/freellmapi && docker compose logs --tail=20 freellmapi
   ```
3. Enter the printed code in the web UI to set a new password.

---
