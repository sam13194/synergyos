# SynergyOS

**AI SysAdmin layer for the Debian/Ubuntu you already run.**

No new distro. No reinstall. No cloud dependency. Works on the boring, real-world
server sitting in your rack or VPS today — including the one running production right now.

> `synergy status` → your server explains its own health, in plain language, using a
> local model. Nothing leaves the machine.

---

## Why not just use an AI-native distro?

Solutions like NixOS-based AI setups are powerful but require migration — a full
reinstall or a parallel system. Most servers can't afford that.

SynergyOS is different: **it's a thin layer you drop on top of what you have.**
One script, one Ollama install, zero infrastructure changes. Your existing services,
logs, and config remain untouched. The AI reads them; it never touches them.

---

## Install

```bash
# 1. Install Ollama (one-time)
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.1:8b

# 2. Install synergy
chmod +x synergy
sudo cp synergy /usr/local/bin/synergy
# or without sudo:
mkdir -p ~/.local/bin && cp synergy ~/.local/bin/synergy
```

---

## Usage

```bash
# AI-summarized health: disk, RAM, load, failed services, recent errors
synergy status

# Plain-language explanation of a specific service's logs
synergy explain nginx
synergy explain sshd
synergy diagnose postgresql

# Use a different local model
synergy status --model qwen2.5:7b
synergy explain docker --model mistral

```

The script reads **real** system state via `df`, `free`, `uptime`, `systemctl`, and
`journalctl` — all strictly read-only. It then asks your local Ollama model to
interpret it in plain Spanish (prompt is in Spanish; change `cmd_status`/`cmd_explain`
if you prefer another language).

---

## Real-world test

Tested on a VirtualBox VM — **no GPU, CPU-only inference**:

- OS: Debian 13 (bookworm)
- CPU: 4 cores (host: Intel)
- RAM: 11 GB
- GPU: none

| Model | Mode | Prompt size | Time | Quality |
|---|---|---|---|---|
| qwen2.5:0.5b | `--fast` | 849 chars | ~1.5 min | ❌ hallucinations, dangerous suggestions |
| llama3.1:8b | normal | 2366 chars | ~10 min | ✅ accurate |
| llama3.1:8b | `--fast` | 705 chars | ~6 min | ✅ accurate |

**Verdict**: `llama3.1:8b --fast` is the sweet spot for CPU-only servers.
Model size matters more than prompt size for this use case — do not go below 7-8B parameters.

### Sample output (`llama3.1:8b --fast`)

Real system state fed to the model:
```
disk: /dev/sda1 19G used 15G (82%)
RAM:  11Gi total, 6.6Gi used, 531Mi free
load: 1.70, 4.63, 3.73
failed services: none
journal errors: sudo pam_unix auth failed for vboxuser (x2)
```

Model response (in Spanish, as prompted):
```
Salud: Todo parece estar en orden.

Alertas reales:
- El disco /dev/sda1 está a 81% de uso y puede ser necesario expandir el espacio.
- El proceso "sudo" ha tenido un error de autenticación para vboxuser,
  lo que puede indicar problemas de seguridad o configuración con PAM.

Acciones:
- Verificar y ajustar el tamaño del disco /dev/sda1 según sea necesario.
- Investigar y solucionar los errores de autenticación para vboxuser.
```

Disk alert and sudo error detected correctly from real `journalctl` output.
RAM pressure (531 Mi free) was missed — known limitation of `--fast` compressed prompt.

### Recommended setup for CPU-only servers

```bash
# run in background every 15 minutes, read the report when you want
*/15 * * * * synergy status --fast --model llama3.1:8b >> /var/log/synergy.log 2>&1
tail -60 /var/log/synergy.log
```

---

## Principles

- **Local-first** — works without internet or cloud APIs. Cloud is optional, never required.
- **Human-supervised** — assists diagnosis; does not replace the admin or take autonomous action.
- **Read before write** — current version is strictly read-only. Any future mutating
  action will sit behind explicit permission layers and audit logs.
- **Explainable** — translates `OOM Killer invoked` into *"the server ran out of RAM;
  a process exceeded available memory."* Not a black box.

---

## Multi-server setup

For teams running multiple servers, SynergyOS works in a hub-and-spoke model:
one **dedicated AI node** (8 GB RAM) runs Ollama and connects to the rest of the fleet
via SSH to collect their state in real time. Managed servers need **nothing installed** —
just SSH access.

```
┌─────────────────────────────┐
│  AI node (192.168.1.10)     │──SSH──► [web server]
│  synergy + Ollama           │──SSH──► [db server]
│  8 GB RAM, always-on        │──SSH──► [ci server]
└─────────────────────────────┘──SSH──► [backup server]
             ▲
    you query this node
```

This is the planned architecture for the next phase. The current version analyzes
the local machine only.

The key insight: `df`, `free`, `systemctl`, and `journalctl` are already installed
on every Debian/Ubuntu server — they're part of the base OS. The AI node will run
them remotely via SSH and analyze the output locally with Ollama.

**Only the AI node needs synergy + Ollama.** Managed servers need nothing installed —
just SSH enabled, which every Linux server already has. One 8 GB node monitors
your entire fleet.

---

## Roadmap

- [x] `synergy status` — AI health summary (disk, RAM, load, failed services, errors)
- [x] `synergy explain <service>` / `synergy diagnose <service>` — log interpretation
- [ ] `synergy security scan` — SSH attempts, SSL expiry, open ports, firewall posture
- [ ] `synergy check backups` — detect stale or absent backup jobs
- [ ] `synergy watch` — continuous monitoring with threshold-based alerts
- [ ] `synergy status --host <server>` — collect state via SSH, analyze on AI node
- [ ] `synergy explain <service> --host <server>` — remote log explanation
- [ ] Web dashboard on the AI node — per-server status, alert history, model activity
- [ ] Permission layers (read-only → safe-ops → admin) + append-only audit ledger
- [ ] Config file (`~/.config/synergy.toml`) for model, language, thresholds

---

## Looking for contributors

Interested in Linux internals, observability, local LLMs, Python/Rust/Go, DevOps, or
self-hosting? This is early-stage with a working seed — the interesting problems are
ahead.

Good first issues to open:
- Add a second language (English prompts, French prompts)
- Package as a proper pip-installable CLI
- Write integration tests that mock Ollama responses
- Implement `synergy security scan`

Issues and PRs are welcome.

---

## Not a goal

- A surveillance or monitoring system that phones home
- A cloud-dependent layer (Datadog, New Relic, etc.)
- A black-box autonomous root agent that acts without confirmation
- Replacing your existing monitoring stack — this is a complement, not a replacement

**Philosophy**: transparency, auditability, local-first, human-supervised.

---

## Status

Early concept + functional proof of concept. No roadmap guarantees. MIT licensed.
