# ADR-0002: Host hardware choice

- Status: Accepted — **Zero 2 W rejected as primary; fall back to a larger Pi**
- Date: 2026-06-20 (verdict recorded 2026-06-20 after hardware bench)
- Related: docs/plan.md 3a, 3g, "Fallback options"; ADR-0003

## Context

The provocative target is a **Raspberry Pi Zero 2 W** (quad-core A53, **512 MB
RAM**, arm64). RAM — not CPU or architecture — is the binding constraint: Copilot
CLI (Node) ~150-300 MB and Neovim + a few LSPs ~200-400 MB is tight on 512 MB and
will lean on swap.

## Decision

**Target the Zero 2 W as primary, but defer the final commitment to the Phase 1
bench** (`feas-ram`, `feas-power`, `feas-localconsole`). Mitigations to apply
before judging: zram, a modest swap file, a minimal/lazy-loaded LSP set, trimmed
treesitter parsers, headless `:Lazy! sync` at build time.

**Decision rule:** if a normal session (nvim + 1-2 LSPs + Copilot CLI) thrashes
swap or OOM-kills even after mitigation, fall back to:

1. **Pi 4 / Pi 5 (2-8 GB)** — same gadget + local-console approach, removes the
   RAM constraint (verify Pi 5 OTG specifics).
2. **Minimal cloud VM** — reuse stripped `azure-dev-machine`; loses the
   plug-in-USB magic but keeps reproducibility.

## Consequences

- This ADR will be updated to **Accepted** with the chosen host once the bench
  data (especially `feas-ram`) is in.
- The local-console engine choice (cage+foot, ADR-0003) is independent of host
  and carries over to any fallback.

## Bench verdict (2026-06-20)

Imaged a real Zero 2 W (Pi OS Lite 64-bit) and ran the local-console stack.
Copilot CLI **does run**, but interactive use is **too sluggish to be pleasant —
even typing to exit the CLI is painful.** That fails the daily-driver bar on its
own, independent of exact swap/OOM numbers, so the operator called it.

**Decision:** Zero 2 W is **rejected as the primary host.** Fall back to a larger
Pi (**Pi 4 / Pi 5, 2-8 GB**) — same USB-gadget + local-console approach, which
removes the RAM constraint. The cloud-VM option remains the secondary fallback.
The cage+foot console, Nerd Font, and toolchain steps validated so far carry over
unchanged to the larger board.

**Update (2026-06-21):** The full runbook was completed end-to-end on a **Pi 4**
and **works great** — local console (cage+foot, Tokyo Night), Nerd Font glyphs,
LazyVim, and Copilot CLI all responsive. Pi 4 is the confirmed host.
