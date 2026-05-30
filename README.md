# Q-Gunter — Automated AI Pentesting Platform

<div align="center">

![Platform Status](https://img.shields.io/badge/status-live%20in%20production-brightgreen?style=flat-square)
![License](https://img.shields.io/badge/license-academic-blue?style=flat-square)
![Model](https://img.shields.io/badge/AI-Claude%20Haiku-orange?style=flat-square)
![Deployment](https://img.shields.io/badge/deployed-q--gunter.cat-black?style=flat-square)

**B2B SaaS platform that automates penetration testing using AI agents.**  
Each client runs in a fully isolated Docker container, powered by their own Anthropic API key.

🌐 [q-gunter.cat](https://q-gunter.cat) · 📄 [Arquitectura](#arquitectura--architecture) · 🎬 [Demo](#demo)

</div>

---

## Demo

> 📹 **Full engagement walkthrough** — from CLI launch to report download.

<!-- VIDEO EMBED — replace with actual link once uploaded -->
[![Q-Gunter Demo](https://img.shields.io/badge/▶%20Watch%20Demo-YouTube-red?style=for-the-badge)](https://github.com/adambenahmednew/q-gunter#demo)

*The demo shows: CLI wizard → AI agent recon & exploitation → real-time finding registration → report generation → dashboard download.*

---

## What is Q-Gunter?

Q-Gunter is a **B2B SaaS pentesting platform** built as the final project of the *Màster d'Especialització en Ciberseguretat* at Institut Tecnològic de Barcelona (2025). The system is fully deployed and running at [q-gunter.cat](https://q-gunter.cat).

The platform lets clients launch autonomous security audits in minutes:

1. Request access and sign the Rules of Engagement (ROE)
2. Create an instance and link your Anthropic API key (**BYO-Key model**)
3. Run the CLI agent from any Kali Linux machine via Docker
4. The AI agent performs the pentest autonomously — recon, scanning, exploitation attempts, finding registration
5. Download the structured report from the web dashboard

```bash
docker run -it qgunters/cli-agent:v1.1.0
```

---

## Architecture

```
                         ┌─────────────────────────────────────────┐
                         │           Cloudflare CDN / WAF           │
                         └──────────────────┬──────────────────────┘
                                            │ HTTPS + WSS
                         ┌──────────────────▼──────────────────────┐
                         │       Frontend HA (Nginx + keepalived)   │
                         │         fr01 (MASTER) · fr02 (BACKUP)    │
                         └──────────────────┬──────────────────────┘
                                            │
                         ┌──────────────────▼──────────────────────┐
                         │              Backend (bk01)              │
                         │   ┌──────────────┐ ┌────────────────┐   │
                         │   │  FastAPI Web │ │  Orchestrator  │   │
                         │   │     API      │ │  REST + WSS    │   │
                         │   └──────────────┘ └───────┬────────┘   │
                         │                            │ docker-py  │
                         │              ┌─────────────▼──────────┐ │
                         │              │  Client containers      │ │
                         │              │  qgunter-inst-1  (AI)  │ │
                         │              │  qgunter-inst-2  (AI)  │ │
                         │              │  qgunter-inst-N  (AI)  │ │
                         │              └────────────────────────┘ │
                         └──────────┬──────────────────────────────┘
                                    │
               ┌────────────────────▼───────────────────────┐
               │           Data layer (CEC2 network)         │
               │   MariaDB Galera HA cluster  ·  MinIO S3    │
               └────────────────────────────────────────────┘
```

**Key design decisions:**

- **True multi-tenancy:** each client runs in a fully isolated Docker container — no shared execution environment
- **BYO-Key:** clients use their own Anthropic API key; Q-Gunter never resells tokens or intermediates model calls
- **HA at every critical layer:** Nginx + keepalived on the frontend, MariaDB Galera cluster on the database
- **WebSocket proxy:** the orchestrator authenticates each CLI connection via API key and routes it to the correct container
- **Real-time findings:** the AI agent calls `add_finding` immediately on each validated vulnerability — if the connection drops mid-engagement, no data is lost

---

## AI Agent

The agent runs a **tool-use loop** powered by `claude-haiku-4-5`. It receives the scope and ROE, then autonomously chains tool calls until the engagement is complete.

**Available tools:**

| Tool | Direction | Description |
|------|-----------|-------------|
| `bash_execute` | Agent → CLI | Runs commands on the client machine (nmap, sqlmap, gobuster...) |
| `file_read` / `file_write` | Agent → CLI | Read/write files during the engagement |
| `add_finding` | Agent → Orchestrator | Registers a validated vulnerability in real time |
| `submit_report` | Agent → Orchestrator | Sends the final structured report on engagement close |

The system prompt enforces strict ethical limits: never test outside scope, never exfiltrate real data, never leave persistent backdoors, always clean up artifacts.

Findings are structured with **CVSS 3.1 scoring**, **MITRE ATT\&CK mapping**, PoC commands and remediation guidance.

---

## Tech Stack

| Layer | Technologies |
|-------|-------------|
| **AI** | Anthropic Claude (Haiku), tool-use loop |
| **Backend** | Python, FastAPI, docker-py, WebSockets |
| **Frontend** | Vanilla JS SPA, Nginx |
| **Database** | MariaDB Galera Cluster (HA), MinIO (S3-compatible) |
| **Infrastructure** | Docker, keepalived, Cloudflare WAF/CDN, Tailscale |
| **Monitoring** | Wazuh (SIEM), Zabbix, Suricata |
| **CLI toolkit** | nmap, sqlmap, gobuster, dirb, Kali Linux base |

---

## Security

The platform is built security-first — it runs offensive tooling, so its own security matters doubly:

- **Perimeter:** Cloudflare WAF with custom rules (bot filtering, rate limiting, Tor blocking, offensive tool UA blocking)
- **Network segmentation:** three isolated networks; database nodes have zero internet access
- **Access control:** all SSH restricted to Tailscale VPN; key-only auth; no root login
- **App security:** JWT authentication, parametrized queries, Pydantic input validation, service tokens between internal components
- **Client isolation:** WebSocket proxy validates API key on handshake; containers never share execution context

---

## Team

| Name | Role |
|------|------|
| **Adam Ben Ahmed Belachi** | Orchestrator · AI agent · CLI Docker · Anthropic integration · Frontend · Legal docs |
| Raúl Molina Kind | Web API · Monitoring infrastructure · Production deployment |
| Ayman Dghoughi Nouri | Network infrastructure · HA frontend · MariaDB HA · Cloudflare · MinIO · Security |

**Institut Tecnològic de Barcelona — Màster d'Especialització en Ciberseguretat — 2025**

---

## Repository structure

```
q-gunter/
├── README.md
└── docs/
    ├── architecture.md      # Detailed architecture walkthrough
    ├── ai-agent.md          # AI agent design and system prompt structure
    ├── security.md          # Security model and threat mitigations
    └── demo.md              # Demo guide and video walkthrough
```

> This is the **public-facing documentation repository**. The production codebase is private.

---

---

# Q-Gunter — Plataforma de Pentesting Automatizado con IA

<div align="center">

**Plataforma SaaS B2B que automatiza pruebas de penetración mediante agentes de IA.**  
Cada cliente ejecuta en un contenedor Docker completamente aislado, con su propia API key de Anthropic.

</div>

---

## ¿Qué es Q-Gunter?

Q-Gunter es una **plataforma SaaS B2B de ciberseguridad** desarrollada como proyecto final del *Màster d'Especialització en Ciberseguretat* del Institut Tecnològic de Barcelona (2025). El sistema está completamente desplegado y en producción en [q-gunter.cat](https://q-gunter.cat).

El modelo es simple: el cliente se registra, firma las Reglas de Engagement, vincula su API key de Anthropic (**modelo BYO-Key**) y lanza el pentest desde un contenedor Docker. La plataforma se encarga del resto — orquestación, ejecución autónoma, registro de findings y generación del informe.

---

## Arquitectura

La infraestructura está formada por seis máquinas virtuales con alta disponibilidad en las capas críticas y aislamiento total entre clientes:

- **Frontend HA:** par Nginx + keepalived (fr01 MASTER / fr02 BACKUP) con IP virtual flotante
- **Backend:** orquestador FastAPI que gestiona contenedores Docker por cliente, enrutamiento WebSocket y almacenamiento de reportes en MinIO
- **Base de datos HA:** clúster MariaDB Galera (replicación síncrona, RPO=0, RTO<2s)
- **Agente IA:** contenedor privado por instancia de cliente, loop tool-use sobre Claude Haiku
- **CLI cliente:** imagen Docker pública con toolkit completo de pentesting (nmap, sqlmap, gobuster...)
- **Monitorización:** Wazuh (SIEM), Zabbix, Suricata en nodo dedicado con acceso a todas las redes

Ver [docs/architecture.md](docs/architecture.md) para el detalle completo.

---

## Agente IA

El núcleo técnico del sistema. El agente recibe el scope y las ROE y entra en un loop de tool-use autónomo hasta completar el engagement. Registra cada finding validado en tiempo real con CVSS 3.1, mapeo MITRE ATT&CK, PoC reproducible y recomendación de remediación.

El sistema prompt incluye límites éticos no negociables: nunca actuar fuera del scope, nunca exfiltrar datos reales, nunca dejar backdoors persistentes, siempre limpiar artefactos.

---

## Diferenciadores

| | Q-Gunter |
|---|---|
| **Aislamiento** | Cada cliente en su propio contenedor Docker. Sin entorno compartido |
| **BYO-Key** | El cliente usa su API key de Anthropic. Q-Gunter no revende tokens |
| **Onboarding** | Del registro al primer pentest en minutos |
| **Findings en tiempo real** | `add_finding` persiste cada hallazgo inmediatamente — resistente a fallos de conexión |
| **HA real** | Keepalived en frontend + Galera en base de datos |

---

## Repositorio

> Este es el **repositorio público de documentación**. El código de producción es privado.

```
q-gunter/
├── README.md
└── docs/
    ├── architecture.md      # Arquitectura detallada
    ├── ai-agent.md          # Diseño del agente IA
    ├── security.md          # Modelo de seguridad
    └── demo.md              # Guía de demo y vídeo
```
