# Homelab Setup — SwarmSentinel Distributed Security Stack

A two-node homelab running a distributed security monitoring platform, AI inference, self-hosted services, and a Flux network node — all behind a single router with port forwarding.

---

## Hardware

| Node | Role | RAM | Storage |
|------|------|-----|---------|
| Node 1 (ProDesk) | Flux node + SwarmSentinel | 32GB | 500GB SSD + NVMe |
| Node 2 (ProDesk) | Inference + services | 16GB | 240GB SSD |
| Pi-hole | LAN DNS filter | — | — |
| Cloud VM | MongoDB / collective intel | — | — |

---

## Network Layout

```
Internet
    │
    ├── Flux ports (P2P, API, UI) ──► Node 1
    ├── SwarmSentinel dashboard   ──► Node 1 :5000
    │
  Router / NAT
    │
    ├── Node 1  (primary server)
    ├── Node 2  (inference + Docker services)
    ├── Pi-hole (LAN DNS)
    └── Windows workstation (Zelcore wallet + browser extensions)

Node 1 ──► Node 2 Ollama (llama3 inference)
Node 1 ──► Cloud VM (threat intel sync)
Browser extensions ──► Node 1 SwarmSentinel API
```

---

## Node 1 — Primary Server

### Flux Node (Cumulus Tier)
- FluxOS v8.14.2 + fluxd daemon + fluxbenchd
- Collateral held in Zelcore wallet on a separate Windows machine — never stored on the node itself
- Required open ports (forward at router to Node 1 LAN IP):

| Port | Protocol | Purpose |
|------|----------|---------|
| 16125 | TCP | Flux daemon P2P |
| 16127 | TCP | FluxOS API + P2P overlay |
| 16137 | TCP | FluxOS secondary API |
| 16147 | TCP | FluxOS P2P alt |
| 16167 | TCP | FluxOS P2P alt |
| 16187 | TCP | FluxOS UI |

**Config** (`~/.flux/flux.conf`):
```ini
rpcallowip=127.0.0.1
port=16125
txindex=1
addressindex=1
timestampindex=1
spentindex=1
insightexplorer=1
experimentalfeatures=1
server=1
daemon=1
zelnode=1
zelnodeoutpoint=<COLLATERAL_TXID>
zelnodeprivkey=<NODE_PRIVKEY>
externalip=<YOUR_PUBLIC_IP>
bind=0.0.0.0
addnode=explorer.runonflux.io
addnode=explorer.flux.zelcore.io
```

**Node config** (`~/.flux/fluxnode.conf`):
```
# alias  IP:port  privkey  collateral_txid  output_index
cumulus1 <YOUR_PUBLIC_IP>:16125 <NODE_PRIVKEY> <COLLATERAL_TXID> 0
```

**Node start procedure:**
1. Ensure all Flux ports are forwarded at the router
2. Start `fluxd`, `flux` (FluxOS), and `fluxbenchd` services
3. Open Zelcore → FluxNodes → Start (broadcasts signed confirmation tx from collateral wallet)
4. Wait 5–15 min for `STARTED` → `CONFIRMED`
5. If node enters `DOS` state: restart the `flux` service to reset local DOS counter, then Start again from Zelcore
6. If DOS persists: update FluxOS (`cd /opt/flux && git pull && sudo npm install --production`), restart, Start from Zelcore again

---

### SwarmSentinel

Security monitoring platform with 15 background agents.

**Systemd services:**
```
swarmsentinel.service  — Flask API on :5000, SQLite backend
ssai-agents.service    — 15 agent threads (loads .env via EnvironmentFile)
ssai-github.service    — sanitized threat intel auto-push to GitHub
```

**Environment** (`.env`):
```ini
FLASK_ENV=production
SECRET_KEY=<CHANGE_THIS>

# AI inference — split across two nodes intentionally
OLLAMA_URL=http://<NODE2_IP>:11434    # llama3 on Node 2
HERMES_URL=http://127.0.0.1:11434    # hermes3 local on Node 1
OLLAMA_MODEL=llama3
HERMES_MODEL=hermes3
```

**Active agents:**

| Agent | Role | Endpoint |
|-------|------|----------|
| OllamaAnalyzerAgent | AI scoring — high/critical threats | Node 2 Ollama (llama3) |
| HermesAnalyzerAgent | IOC extraction — medium threats | Node 1 local Ollama (hermes3) |
| ThreatFeedAgent | Live URLhaus / abuse.ch feeds | Internet |
| NetworkConnAgent | Active connection monitoring | Local |
| AuthLogAgent | SSH / auth log parsing | Local |
| SyslogAgent | Syslog anomaly detection | Local |
| PortMonitorAgent | Unexpected listener detection | Local |
| ProcessMonitorAgent | Suspicious process detection | Local |
| FileIntegrityAgent | Critical file hash monitoring | Local |
| ServiceWatchAgent | Systemd service health | Local |
| DiskSpaceAgent | Storage headroom monitoring | Local |
| PiHoleAgent | DNS query anomaly monitoring | Pi-hole API |
| BrowserExtAgent | Browser extension threat reports | Local |
| CosmosDBSyncAgent | Cloud threat intel sync | Cloud VM MongoDB |
| NetworkScanAgent | LAN network scan | Local |

**AI split design rationale:**

Running both models on Node 1 would compete with the Flux node for CPU/RAM, degrading Flux uptime and node confirmations. The split keeps Node 1 lean:

- `HermesAnalyzerAgent` → hermes3 → **Node 1 local Ollama** — fast IOC extraction with no network hop; lightweight enough to coexist with Flux
- `OllamaAnalyzerAgent` → llama3 → **Node 2 Ollama** — heavier general-purpose model fully offloaded to Node 2, so it has zero impact on Flux performance

Both nodes sync detected threats to the Azure cloud VM (MongoDB) independently — SwarmSentinel agents on Node 1 push/pull from MongoDB over the internet regardless of which Ollama endpoint they hit.

Each model has its own env var (`HERMES_URL` / `OLLAMA_URL`) so either can be redirected without touching the other.

---

### Ollama — Node 1 (hermes3 only)

```ini
# /etc/systemd/system/ollama.service
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

[Install]
WantedBy=default.target
```

- Listens on `127.0.0.1:11434` only — not exposed to LAN
- Pull: `ollama pull hermes3`

---

## Node 2 — Inference + Services Server

Ubuntu Server 24.04 LTS, headless. DHCP reservation by MAC at router.

### Ollama — primary inference

```ini
# /etc/systemd/system/ollama.service (add to [Service] section)
Environment="OLLAMA_HOST=0.0.0.0"
```

- Listens on `0.0.0.0:11434` — accessible to Node 1 over LAN
- Pull: `ollama pull llama3 && ollama pull hermes3`
- CPU-only (no GPU required)

### Docker Services

**Traefik v3.0** (`~/traefik/docker-compose.yml`):
```yaml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: always
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

**Gitea** (`~/gitea/docker-compose.yml`):
```yaml
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    environment:
      - USER_UID=1000
      - USER_GID=1000
    ports:
      - "3000:3000"
      - "2222:22"
    volumes:
      - gitea_data:/data
volumes:
  gitea_data:
```

**Wazuh SIEM** (single-node):
```bash
git clone https://github.com/wazuh/wazuh-docker.git --branch v4.7.0
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```

**Portainer CE:**
```bash
docker volume create portainer_data
docker run -d -p 9000:9000 --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

---

## Kubernetes — K3s Two-Node Cluster

### Node 1 — Control Plane
```bash
curl -sfL https://get.k3s.io | sh -

# Allow worker join connections through firewall
ufw allow from <LAN_SUBNET>/24 to any port 6443

# Retrieve join token for worker
cat /var/lib/rancher/k3s/server/node-token
```

### Node 2 — Worker
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<NODE1_IP>:6443 \
  K3S_TOKEN=<JOIN_TOKEN> sh -
```

**Verify cluster (from Node 1):**
```bash
kubectl get nodes
```

---

## Pi-hole — LAN DNS Filtering

Blocks ads and known malicious domains network-wide. SwarmSentinel `PiHoleAgent` monitors query logs for suspicious DNS activity.

```bash
curl -sSL https://install.pi-hole.net | bash
```

Set as primary DNS in router DHCP settings.

---

## Cloud VM — Collective Threat Intel

Runs MongoDB. SwarmSentinel `CosmosDBSyncAgent` pushes detected threats to the cloud and pulls collective IOCs back, enabling cross-installation threat correlation across any SwarmSentinel deployment.

---

## Security Hardening

Applied to both nodes:

```bash
# SSH — disable password auth, key-only
sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# UFW baseline
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw enable
```

### Disable Energy Efficient Ethernet (Intel e1000e NICs)
Prevents link flapping on Intel NICs under Linux:

```bash
# /etc/udev/rules.d/99-eno1-eee.rules
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eno1", \
  RUN+="/usr/sbin/ethtool --set-eee eno1 eee off"

# /etc/modprobe.d/e1000e.conf
options e1000e SmartPowerDownEnable=0
```

### Disable cloud-init network management
Prevents cloud-init from overwriting netplan on every reboot:
```bash
touch /etc/cloud/cloud-init.disabled
```

`/etc/netplan/50-cloud-init.yaml`:
```yaml
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: true
```

---

## Income Nodes (Node 1)

| Node | Network | Notes |
|------|---------|-------|
| Flux (Cumulus) | Decentralized compute | Collateral in separate Zelcore wallet — not on node |
| Mysterium | Decentralized VPN | Runs as systemd service |
| Pawns | Bandwidth sharing | Runs as systemd service |

---

## Full Topology

```
┌─────────────────────────────────────────────────────┐
│                      Node 1                         │
│  ┌──────────────────┐  ┌───────────────────────┐    │
│  │  Flux (Cumulus)  │  │    SwarmSentinel       │    │
│  │  Mysterium       │  │  15 agents · API :5000 │    │
│  │  Pawns           │  └────────────┬───────────┘    │
│  └──────────────────┘               │                │
│            Ollama (hermes3 only — local, keeps       │
│            resources free for Flux)  ◄───────────────┘
└────────────┬───────────────────┬────────────────────┘
             │ llama3 over LAN   │ threat intel sync
             ▼                   ▼
┌────────────────────┐  ┌─────────────────────────────┐
│      Node 2        │  │         Cloud VM            │
│  Ollama llama3     │  │  MongoDB — collective IOCs  │
│  (offloaded —      │  │  Both nodes push/pull here  │
│  no Flux conflict) │  └─────────────────────────────┘
│  Traefik · Gitea   │             ▲
│  Wazuh · Portainer │─────────────┘ threat intel sync
│  K3s worker        │
└────────────────────┘

LAN-wide: Pi-hole DNS filtering
Windows:  Zelcore wallet (Flux collateral) + browser extensions
```
