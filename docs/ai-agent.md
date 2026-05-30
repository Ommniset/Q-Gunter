# AI Agent — Q-Gunter

## Design philosophy

The agent is not a chatbot wrapper around a pentesting tool. It is an **autonomous decision-making loop** that chains real tool executions to reach an objective — the same way a human pentester would, but without requiring continuous human input.

The key insight: tool-use capability in modern LLMs makes it possible to build a system where the model decides *what* to run, interprets the output, and decides *what to run next* — all within a defined ethical and operational boundary.

---

## Tool-use loop

```
┌─────────────────────────────────────────────────┐
│              AI Agent (claude-haiku-4-5)         │
│                                                  │
│  system_prompt (scope + ROE + methodology)       │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Loop:                                   │   │
│  │  1. Call Anthropic API with history      │   │
│  │  2. If response = tool_use:              │   │
│  │     - Execute tool on CLI / orchestrator │   │
│  │     - Append result to history           │   │
│  │     - Go to 1                            │   │
│  │  3. If response = add_finding:           │   │
│  │     - POST finding to orchestrator       │   │
│  │     - Continue loop                      │   │
│  │  4. If response = submit_report:         │   │
│  │     - POST final report                  │   │
│  │     - End loop                           │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

## Tools

| Tool | Target | Description |
|------|--------|-------------|
| `bash_execute` | Client CLI | Executes shell commands on the client machine (nmap, sqlmap, gobuster, dirb, curl, dig, etc.) |
| `file_read` | Client CLI | Reads files from the client machine during the engagement |
| `file_write` | Client CLI | Writes files to the client machine (wordlists, scripts, intermediate results) |
| `add_finding` | Orchestrator | Registers a validated, structured vulnerability finding immediately |
| `submit_report` | Orchestrator | Sends the final engagement report and closes the session |

**Why `add_finding` is real-time:** findings are persisted to the database as soon as they are validated, not at the end of the engagement. If the agent or connection fails mid-run, all findings up to that point are already saved.

---

## System prompt structure

The system prompt is built dynamically by `build_system_prompt()`, injecting the engagement configuration into a base prompt. It covers:

**Operational principles**
- Scope compliance (positive targets + explicit exclusions)
- Professional persistence — don't give up on a vector after one failure
- Business impact orientation — document *why* a finding matters

**Pentesting methodology — five phases**
1. Reconnaissance — passive and active information gathering
2. Discovery — service enumeration, attack surface mapping
3. Exploitation — targeted exploitation attempts within scope
4. Post-exploitation — access validation, lateral movement within authorized scope
5. Reporting — structured findings with evidence

**Ethical limits (non-negotiable)**
```
NEVER test outside scope.
NEVER exfiltrate real data — proof of access only.
NEVER perform DoS unless explicitly authorized.
NEVER leave persistent backdoors.
ALWAYS clean up artifacts.
ALWAYS escalate to a human if you encounter evidence of a third-party breach.
```

**Finding quality standards**
- CVSS 3.1 score and full vector string
- MITRE ATT&CK tactic/technique mapping
- Reproducible PoC (exact commands)
- Terminal output as evidence
- Specific remediation guidance
- External references (CVE, CWE, OWASP)

---

## Finding schema

Each finding registered via `add_finding` is structured as:

```json
{
  "external_id": "F-001",
  "title": "Unauthenticated RCE via CVE-XXXX-XXXX",
  "severity": "Critical",
  "cvss_score": 9.8,
  "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
  "mitre_attack": "TA0002 - Execution / T1059 - Command and Scripting Interpreter",
  "affected_assets": "192.168.1.10:8080",
  "description": "Technical vulnerability description",
  "business_impact": "An unauthenticated attacker can execute arbitrary commands...",
  "proof_of_concept": "curl -X POST http://target:8080/exploit -d 'cmd=id'",
  "evidence": "uid=0(root) gid=0(root) groups=0(root)",
  "remediation": "Update to version X.Y.Z. Apply patch advisory...",
  "refs": "CVE-XXXX-XXXX, CWE-78, OWASP A03:2021"
}
```

---

## Validated capabilities

The agent has been tested against:
- Lab environments with intentionally vulnerable machines
- CTF challenges on PicoCTF and TryHackMe

Demonstrated capabilities: network reconnaissance, service enumeration, web application vulnerability identification (OWASP Top 10 vectors), automated exploitation of known CVEs, credential analysis, report generation.
