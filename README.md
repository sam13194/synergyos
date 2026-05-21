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

## Principles

- **Local-first** — works without internet or cloud APIs. Cloud is optional, never required.
- **Human-supervised** — assists diagnosis; does not replace the admin or take autonomous action.
- **Read before write** — current version is strictly read-only. Any future mutating
  action will sit behind explicit permission layers and audit logs.
- **Explainable** — translates `OOM Killer invoked` into *"the server ran out of RAM;
  a process exceeded available memory."* Not a black box.

---

## Multi-server setup

For teams running multiple servers, SynergyOS can work in a hub-and-spoke model:
one **dedicated AI node** (8 GB RAM) runs Ollama and serves the entire fleet.
Each managed server sends its state there for analysis — logs never leave your network.

```
                        ┌─────────────────────────────┐
[web server]   ──────►  │  AI node (192.168.1.10)     │
[db server]    ──────►  │  Ollama + llama3.1:8b        │
[ci server]    ──────►  │  8 GB RAM, always-on         │
[backup server]──────►  └─────────────────────────────┘
                                      ▲
                             you SSH here to query
```

### Step 1 — Configure the AI node

The AI node is just a Linux server with Ollama exposed on the local network.

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull the model (do this once)
ollama pull llama3.1:8b

# By default Ollama only listens on localhost.
# To accept connections from other servers on the LAN:
echo 'OLLAMA_HOST=0.0.0.0:11434' | sudo tee -a /etc/environment
sudo systemctl restart ollama   # or: ollama serve

# Install synergy on the AI node too (optional, for local queries)
chmod +x synergy
sudo cp synergy /usr/local/bin/synergy
```

> **Security note:** expose port 11434 only within your LAN or VPN.
> There is no authentication on the Ollama API — don't expose it to the internet.

### Step 2 — Configure each managed server

Each managed server needs only the `synergy` script and the `SYNERGY_HOST` variable
pointing to the AI node. Ollama is **not** required on managed servers.

```bash
# Install synergy
chmod +x synergy
sudo cp synergy /usr/local/bin/synergy

# Point to the AI node permanently
echo 'SYNERGY_HOST=192.168.1.10:11434' | sudo tee -a /etc/environment
source /etc/environment

# Verify it works
synergy status
synergy explain nginx
```

From this point, all AI processing happens on the AI node.
The managed server only collects local state (df, free, journalctl) and sends the text prompt.

---

## Roadmap

- [x] `synergy status` — AI health summary (disk, RAM, load, failed services, errors)
- [x] `synergy explain <service>` / `synergy diagnose <service>` — log interpretation
- [ ] `synergy security scan` — SSH attempts, SSL expiry, open ports, firewall posture
- [ ] `synergy check backups` — detect stale or absent backup jobs
- [ ] `synergy watch` — continuous monitoring with threshold-based alerts
- [ ] `synergy report` — send structured JSON state to a central AI node
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
