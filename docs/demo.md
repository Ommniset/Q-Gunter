# Demo — Q-Gunter

## Full engagement walkthrough

The demo shows a complete end-to-end flow from CLI launch to report download.

---

## 📹 Video demo

<!-- Replace this section with your actual video embed once uploaded -->

> **[▶ Watch full demo on YouTube](#)** *(link coming soon)*

---

## What the demo covers

| Step | What you'll see |
|------|----------------|
| **1. CLI launch** | `docker run -it qgunters/cli-agent:v1.1.0` — the interactive wizard collecting API key, target scope, and ROE confirmation |
| **2. WebSocket handshake** | Connection authenticated and routed to the client's isolated AI container |
| **3. Reconnaissance** | Agent autonomously runs nmap, enumerates services, maps the attack surface |
| **4. Exploitation** | Targeted exploitation attempts against discovered vectors |
| **5. Real-time findings** | Each validated vulnerability registered via `add_finding` — visible in the dashboard immediately |
| **6. Report generation** | Agent calls `submit_report`; structured Markdown report stored in MinIO |
| **7. Dashboard download** | Report downloaded from the web dashboard with all findings, CVSS scores and PoCs |

---

## Live platform

The platform is live at [q-gunter.cat](https://q-gunter.cat).

Access requires signing the Rules of Engagement. To request access, use the contact form on the platform.

---

## Running the CLI (public image)

The CLI agent image is public on Docker Hub:

```bash
# Interactive mode (wizard)
docker run -it qgunters/cli-agent:v1.1.0

# Non-interactive mode (config file)
docker run -it qgunters/cli-agent:v1.1.0 --config engagement.json
```

**Requirements:**
- Docker installed
- Kali Linux or equivalent environment recommended
- A valid Q-Gunter instance API key
- Your Anthropic API key linked to your instance

**Capabilities pre-installed in the image:**
`nmap` · `sqlmap` · `gobuster` · `dirb` · `curl` · `dig` · and more
