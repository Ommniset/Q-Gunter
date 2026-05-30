# Architecture — Q-Gunter

## Overview

Q-Gunter runs on six virtual machines deployed on IsardVDI, fronted by Cloudflare. The architecture is designed around three principles: **high availability at every critical layer**, **complete client isolation**, and **strict network segmentation**.

---

## Network topology

Three isolated networks with controlled inter-network access:

| Network | Range | Purpose |
|---------|-------|---------|
| Default | Internet-facing | Cloudflare ingress, Tailscale VPN |
| CEC1 | `10.10.110.0/24` | Frontend ↔ Backend ↔ Monitoring |
| CEC2 | `10.10.220.0/24` | Backend ↔ Database ↔ MinIO ↔ Monitoring |

Database nodes have **zero internet access** — only reachable from CEC2. The backend is the sole bridge between CEC1 and CEC2.

---

## Components

### Cloudflare (perimeter)

All public traffic enters through Cloudflare CDN/WAF before reaching the infrastructure. Custom WAF rules handle: rate limiting on sensitive endpoints, blocking of offensive tool user-agents (sqlmap, nmap, nikto...), Tor exit node blocking, WebSocket bypass rule for `/ws/*` (Bot Challenge blocked WebSocket upgrades).

### Frontend HA — Nginx + keepalived

Two Nginx nodes (fr01/fr02) form an active/standby HA pair using keepalived (VRRP). A floating VIP (`10.10.110.210`) is held by the master; if fr01 fails, fr02 claims the VIP automatically. The nodes serve the vanilla JS SPA statically and proxy API/WebSocket traffic to the backend.

### Backend — Orchestrator + Web API

Two FastAPI services co-located on bk01:

**Web API (port 8000):** Handles all SPA-facing operations — auth (JWT), instance lifecycle management, report listing and download. Communicates with the orchestrator and the MariaDB cluster.

**Orchestrator (port 9100 REST + 8080 WS):** The core of the system. Manages Docker container lifecycle via `docker-py`, routes authenticated WebSocket connections from client CLIs to the correct AI agent container, stores reports in MinIO, and exposes internal endpoints for agents to register findings.

### Client isolation model

Each client instance maps to one Docker container (`qgunter-inst-{N}`). The orchestrator reads the API key on the WebSocket handshake and proxies the connection exclusively to that client's container. Containers never share execution context or network namespace.

```
CLI (client machine)
    │  WSS authenticated by API key
    ▼
Orchestrator (WebSocket router)
    │  lookup: api_key → container_id
    ▼
qgunter-inst-N  (isolated AI agent)
    │  bash_execute, file_read/write
    ▼
Pentesting tools on client machine
```

The AI agent container has **no direct database access**. All persistence flows through orchestrator internal endpoints, which validate a `SERVICE_TOKEN` on every call.

### Database HA — MariaDB Galera

Two-node MariaDB Galera cluster (db01/db02) on CEC2. Synchronous multi-master replication ensures RPO=0. Keepalived provides a database VIP (`10.10.220.210`) with sub-2-second failover (RTO < 2s). The backend connects exclusively through the VIP.

### MinIO (object storage)

S3-compatible object storage co-located on the database nodes. Stores pentest reports as Markdown files under `reportes-pentest/{instance_id}/{report_id}.md`. Separating report content (MinIO) from metadata (MariaDB) keeps database queries fast regardless of report size.

### Monitoring — mg01

Dedicated node with access to all three networks, running:
- **Wazuh:** SIEM/HIDS with active response (auto-blocks IPs on SSH brute force)
- **Zabbix:** Performance and availability monitoring with CPU threshold alerts
- **Suricata:** Network IDS via port mirroring on internet-facing interfaces
- **Tailscale:** Mesh VPN for secure remote admin access without exposing additional ports

---

## End-to-end engagement flow

```
1. Client runs:  docker run -it qgunters/cli-agent:v1.1.0
2. CLI wizard collects: API key, target scope, ROE confirmation
3. CLI connects: WSS → Cloudflare → Nginx → Orchestrator
4. Orchestrator: validates API key, proxies to client's AI container
5. AI agent: enters tool-use loop with scope injected into system prompt
   ├─ bash_execute → nmap scan → interpret results
   ├─ bash_execute → targeted exploitation attempts
   ├─ add_finding → finding persisted to DB immediately
   └─ (repeat until coverage complete)
6. AI agent: calls submit_report → Orchestrator uploads .md to MinIO
7. Client: downloads report from web dashboard
```

---

## High availability summary

| Component | HA mechanism | Failover time |
|-----------|-------------|---------------|
| Frontend | keepalived VRRP (fr01/fr02) | < 2s |
| Database | MariaDB Galera + keepalived VIP | < 2s |
| CDN/WAF | Cloudflare (managed) | Transparent |
| Backend | Single node (identified improvement for v2) | — |
