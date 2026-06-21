# local-dev-machine — "Pocket Dev"

Ephemeral, reproducible **LazyVim + Copilot CLI** on a Raspberry Pi (Pi 4 / Pi 5),
distilled from `azure-dev-machine` down to the parts worth keeping (LazyVim /
Tokyo Night / Nerd Font glyphs / Copilot CLI on `<C-/>`). No desktop environment.

> The provocative original target — a **Pi Zero 2 W** — was imaged and tested, but
> Copilot CLI was too sluggish for interactive use, so the host is now a **Pi 4 /
> Pi 5** (see [ADR-0002](docs/decisions/0002-host-choice.md)).

## Two co-equal front doors (same Pi, same LazyVim config)

1. **Local console** — plug the Pi into a **monitor + keyboard**; it boots
   straight into a fullscreen truecolor terminal running `nvim`
   (`cage` + `foot`, no desktop). Glyphs render via a Nerd Font installed on the
   Pi. *(This is the path we validate first — see ADR-0003.)*
2. **USB gadget + SSH** — plug the Pi into a Mac (CDC-ECM gadget over one cable);
   `ssh localuser@pocketdev.local` → `nvim`. Glyphs render via the Mac's terminal font.

Neither disables the other; `sshd` is independent of what's on HDMI.

## Status

Design/feasibility stage. The full design and phased build plan live in
[`plan.md`](docs/plan.md). Hardware `(bench)` items are validated on a real Pi.

## Start here

- **Image a Pi and validate the local-console path (hands-on runbook):**
  [`docs/quickstart-local-console.md`](docs/quickstart-local-console.md)
- **Reproduce the same build automatically on first boot (cloud-init):**
  [`cloud-init/`](cloud-init/) - drop `user-data` on the boot partition, no SSH needed.
- **Decisions:** [`docs/decisions/`](docs/decisions/) (ADR-0001 access model,
  ADR-0002 host choice, ADR-0003 local-console-first).

## Guardrails

- Never commit secrets / Wi-Fi creds / network details — use gitignored overlays.
- Keep provisioning idempotent and reproducible; a rebuild restores the full setup.
- Never commit or push without explicit approval.
