# Deploying FreeLLMAPI on Oracle Cloud Infrastructure (OCI) Free Tier

A complete step-by-step guide to deploying **FreeLLMAPI** via Docker Compose on an **Oracle Cloud Infrastructure (OCI) Always Free** compute instance (`10.0.0.10`). Supports both `arm64` (Ampere A1) and `amd64` (Micro) shapes.

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
- [Maintenance & Operations](#maintenance--operations)
- [Video Walkthrough](#video-walkthrough)

---

## Prerequisites

- An active **OCI Always Free** compute instance (Ubuntu 22.04 or 24.04 LTS recommended).
- SSH access to the instance (e.g., private IP `10.0.0.10` over VPN/VCN, or via its assigned public IP).
- At least one free-tier API key from a supported upstream provider (e.g., Google AI Studio, Groq, Cerebras, Mistral, OpenRouter).

---

## Step 1: Connect and Update the OS

SSH into your OCI instance and update the system packages:

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
sudo iptables -I INPUT 6 -m state --state NEW -p tcp -s 10.0.0.0/8 --dport 3001 -j ACCEPT
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## Step 4: Deploy FreeLLMAPI with Docker Compose

Create a dedicated directory for FreeLLMAPI and download the official `docker-compose.yml`:

```bash
mkdir -p ~/freellmapi && cd ~/freellmapi
curl -fsSL https://raw.githubusercontent.com/tashfeenahmed/freellmapi/main/docker-compose.yml -o docker-compose.yml
```

Generate a persistent 64-character hex `ENCRYPTION_KEY` (used for AES-256-GCM encryption of your upstream API keys at rest) and create the `.env` file:

```bash
ENCRYPTION_KEY="$(openssl rand -hex 32)"
cat <<EOF> .env
ENCRYPTION_KEY=${ENCRYPTION_KEY}
PORT=3001
# Use 127.0.0.1 for SSH tunnel / reverse proxy, or 0.0.0.0 for private VPN/VCN access
HOST_BIND=127.0.0.1
EOF
```

> **Note:** If you chose **Option B** in Step 3 to reach the dashboard directly at `http://10.0.0.10:3001` over a private network, update `HOST_BIND=0.0.0.0` in your `.env` file.

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
   - **Via VPN / private network (`HOST_BIND=0.0.0.0`):** `http://10.0.0.10:3001`
2. Create your initial admin account (email and password) on the first-run setup screen.
3. Navigate to the **Keys** page and add your free-tier API keys (e.g., Google AI Studio, Groq, Cerebras, Mistral). FreeLLMAPI will validate each key and display its health status.
4. Configure your preferred **Fallback Chain** and routing strategy.
5. Copy your unified bearer token (`freellmapi-...`) from the top of the **Keys** page.

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

*(Use `http://localhost:3001/v1/chat/completions` if testing locally on the instance or over an SSH tunnel.)*

---

## Maintenance & Operations

### Updating FreeLLMAPI

Pull the latest multi-arch image from GitHub Container Registry and recreate the container (your `.env` file and `freellmapi-data` SQLite volume remain intact):

```bash
cd ~/freellmapi
docker compose pull
docker compose up -d
```

### Backing Up Your Configuration

Your encrypted database is stored in the `freellmapi-data` Docker volume (`/app/server/data/freellmapi.db`). Back up both your `.env` file and the SQLite database—without the original `ENCRYPTION_KEY`, stored provider keys cannot be decrypted:

```bash
cd ~/freellmapi
cp .env .env.backup
docker compose cp freellmapi:/app/server/data/freellmapi.db ./freellmapi.db.backup
```

### Resetting a Forgotten Dashboard Password

1. Click **Forgot password?** on the login page and select **Send reset code**.
2. Retrieve the one-time reset code from the container logs:
   ```bash
   cd ~/freellmapi && docker compose logs --tail=20 freellmapi
   ```
3. Enter the printed code in the web UI to set a new password.

---

## Video Walkthrough

For a visual walkthrough of the dashboard setup, provider key configuration, and connecting coding agents to FreeLLMAPI, check out [7.4B Free AI Tokens for Claude Code (FreeLLMAPI Setup Guide)](https://www.youtube.com/watch?v=0CdXUnnvvBA).
