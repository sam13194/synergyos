# SynergyOS

**Linux that understands itself.**

An AI SysAdmin layer for the Debian/Ubuntu servers you *already run* — no new
distro, no reinstall, no cloud dependency. Local-first by design.

> `synergy status` → your server explains its own health, in plain language.

## Why

Linux tooling is powerful but fragmented. Most operators never check SSL
expiry, suspicious SSH attempts, disk pressure or failed backups until
something breaks — usually at 3am. SynergyOS explores making operational
intelligence native to the system, using small **local** models (via Ollama),
so insight stays private, offline-capable and auditable.

Unlike full AI-native distributions, SynergyOS is designed to run on the
boring, real-world Debian/Ubuntu servers that already exist in production.

## Proof of concept

This repo ships a working seed, not just a manifesto:

```bash
ollama pull llama3.1:8b
chmod +x synergy && sudo cp synergy /usr/local/bin/

synergy status              # AI-summarized system health
synergy explain nginx       # plain-language explanation of a service
```

`synergy` reads **real** system state (df, free, systemctl, journalctl) — all
read-only — and asks a local model to interpret it. No data leaves the machine.

## Principles

- **Local-first** — works without cloud APIs. Cloud is optional, never required.
- **Human-supervised** — assists diagnosis, does not replace the admin.
- **Read before write** — current PoC is strictly read-only. Any future
  mutating action stays behind explicit permission layers and audit logs.
- **Explainable** — translates `OOM Killer invoked` into *"the server ran out
  of memory; a process likely exceeded available RAM."*

## Roadmap (concept)

- [x] `synergy status` — AI health summary (PoC)
- [x] `synergy explain <service>` — log interpretation (PoC)
- [ ] `synergy security scan` — SSH/SSL/firewall posture
- [ ] `synergy check backups`
- [ ] Permission layers (read-only → safe ops → admin) + audit ledger

## Looking for contributors

Interested in Linux internals, observability, local LLMs, Rust/Go/Python,
DevOps or self-hosting? This is an early concept with a working seed. Issues
and PRs welcome.

## Not a goal

A surveillance system, a cloud-dependency layer, or a black-box autonomous
root agent. Philosophy: transparency, auditability, local-first, human-supervised.

## Status

Early concept + functional proof of concept. No roadmap guarantees. MIT licensed.
