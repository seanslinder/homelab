# Self-Hosted DevOps Homelab

This repository contains the Infrastructure as Code (IaC) configuration for a secure, self-hosted homelab. The project demonstrates modern DevOps practices including containerization, centralized authentication (SSO), reverse proxying, internal Certificate Authority (CA) management, and Split-DNS.

## 🏗 Architecture & Tech Stack

- **Containerization**: `Docker` & `Docker Compose` isolated in a custom internal network.
- **Reverse Proxy & Internal CA**: `Caddy` handles all incoming traffic on ports 80/443. It automatically provisions and manages internal TLS certificates for secure local communication.
- **Single Sign-On (SSO)**: `Authelia` acts as an identity provider, protecting internal endpoints with Multi-Factor Authentication (MFA) and granular access control policies.
- **Split-DNS**: `AdGuard Home` (deployed alongside) routes wildcard subdomains (`*.lenovo.netbird.cloud`) to the internal reverse proxy without exposing services to the public internet.
- **Mesh VPN**: `NetBird` provides a secure overlay network, allowing seamless access to the homelab from remote locations without port forwarding on the edge router.

## 🔒 Security Posture

1. **Zero Exposed Ports**: Application containers do not map ports to the host machine. All traffic flows exclusively through the Caddy proxy via the `homelab` Docker network.
2. **End-to-End Encryption**: `tls internal` is enforced across all subdomains.
3. **SSO Gatekeeper**: Critical dashboards (Homepage, Dashdot, Termix) are hidden behind Authelia's `forward_auth`.
4. **WebSocket Handling**: Services requiring persistent connections (e.g., SiYuan) are configured with explicit `Upgrade` headers, bypassing the SSO wall strictly for the WebSocket endpoint to maintain functionality while securing the main UI.

## 📦 Deployed Services

- **[Homepage](https://gethomepage.dev/)**: Main service dashboard and application portal.
- **[Authelia](https://www.authelia.com/)**: Authentication backend and login portal.
- **[Dashdot](https://getdashdot.com/)**: Modern server telemetry and monitoring.
- **[Termix](https://termix.dev/)**: Web-based terminal/SSH client.
- **[Immich](https://immich.app/)**: High-performance, self-hosted photo and video backup solution.
- **[Vaultwarden](https://github.com/dani-garcia/vaultwarden)**: Lightweight Bitwarden-compatible password manager API (bypasses Authelia to support native clients).
- **[SiYuan](https://b3log.org/siyuan/)**: Local-first personal knowledge management system.

## 🚀 Deployment Instructions

### 1. Prerequisites
- Ubuntu Server (or any Linux distribution with Docker installed).
- Docker and Docker Compose plugin.
- A configured DNS sinkhole/server (e.g., AdGuard Home or Pi-Hole) with wildcard records pointing to the server's LAN IP.

### 2. Initial Setup
Clone this repository to your server (e.g., `/opt/homelab`):

```bash
git clone https://github.com/yourusername/homelab-infra.git /opt/homelab
cd /opt/homelab
```

### 3. Environment Variables & Secrets
Create the master environment file and set up secrets for Authelia and Immich:

```bash
# Set up global variables
cp homelab.env.example homelab.env
nano homelab.env # Adjust FQDN and IP addresses

# Set up Authelia users
cp caddy/authelia/users_database.example.yml caddy/authelia/users_database.yml
# Generate a new Argon2 hash for your password
docker run --rm authelia/authelia authelia crypto hash generate argon2 --password 'YOUR_STRONG_PASSWORD'
nano caddy/authelia/users_database.yml # Replace the placeholder hash

# Set up Immich variables
cp immich/.env.example immich/.env
nano immich/.env # Set secure database passwords

# Generate Authelia encryption keys
mkdir -p caddy/authelia/secrets
openssl rand -base64 32 > caddy/authelia/secrets/JWT_SECRET
openssl rand -base64 32 > caddy/authelia/secrets/SESSION_SECRET
openssl rand -base64 32 > caddy/authelia/secrets/STORAGE_ENCRYPTION_KEY
chmod 600 caddy/authelia/secrets/*
```

### 4. Bootstrapping the Infrastructure
Start the Reverse Proxy and SSO provider first. This step creates the `homelab` Docker network.

```bash
cd /opt/homelab/caddy
docker compose up -d
```

Verify that Caddy and Authelia are healthy, then spin up the remaining services:

```bash
for service in homepage dashdot termix siyuan immich vaultwarden; do
  cd /opt/homelab/$service
  docker compose up -d
done
```

### 5. Trusting the Root CA
Since Caddy signs certificates using its internal CA, you must trust it on your client devices to avoid browser warnings.

Extract the root certificate:
```bash
docker compose -f /opt/homelab/caddy/docker-compose.yml exec caddy cat /data/caddy/pki/authorities/local/root.crt > caddy-root-ca.crt
```
Install `caddy-root-ca.crt` in your OS/Browser's trusted Root Certification Authorities store.
