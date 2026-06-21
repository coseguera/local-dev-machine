# ADR-0003: Local console (no desktop environment), validated first

- Status: Accepted
- Date: 2026-06-20
- Related: docs/plan.md 3e-bis; ADR-0001 (co-equal SSH path)

## Context

The operator wants to drive the Pi directly from a **monitor + keyboard** as a
front door that is **co-equal** with the USB-gadget+SSH path, and to **validate
it first**. LazyVim is just a terminal program, so **no desktop environment**
(X11/Wayland window manager, panels, compositor chrome) is required. What is
required is a console that renders **Nerd Font glyphs** and **Tokyo Night
truecolor** — which the stock Linux framebuffer VT (PSF bitmap font, ≤16
colors) cannot do.

## Decision

Boot the Pi straight into a single fullscreen truecolor terminal running
`nvim`, with no desktop, using **`cage` + `foot`**:

- `cage` is a minimal single-application Wayland kiosk compositor (no WM).
- `foot` is a small Wayland terminal with full truecolor + Nerd Font glyph
  rendering.
- Autologin on the console seat → `cage -s -- foot -e nvim`.
- Install **JetBrainsMono Nerd Font** on the Pi (the font lives Pi-side in this
  mode, unlike the SSH mode where it lives on the Mac).

### Why not kmscon / fbterm

*Feasibility finding (2026-06-20, web-verified):* **kmscon was removed from
Debian before Bullseye and is not packaged in Bookworm / Raspberry Pi OS;** it
is unmaintained. **fbterm** was likewise removed. Both would require building
from old sources with dependency pain. They are therefore **rejected** as the
default; cage+foot installs cleanly from apt. kmscon remains only a documented
last-resort source-build option.

## Consequences

- The local console coexists with SSH (ADR-0001); `sshd` is unaffected by what
  renders on HDMI. Both front doors share one LazyVim config.
- In this mode there is no Mac, so **Copilot CLI needs the Pi's own WAN**
  (Wi-Fi provider #2). Plain LazyVim editing + LSP work fully offline.
- cage+foot pulls in Wayland/libdrm/Mesa — heavier than a bare VT, but far
  lighter than a desktop and acceptable on 512 MB (to be confirmed by
  `feas-localconsole` / `feas-ram` bench).
- Must never grow into a desktop; SSH headless capability must keep working.
