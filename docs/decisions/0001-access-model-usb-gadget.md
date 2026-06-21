# ADR-0001: Access model — USB gadget (CDC-ECM) + SSH

- Status: Accepted
- Date: 2026-06-20
- Related: docs/plan.md sections 3b, 3c; ADR-0003 (co-equal local console)

## Context

The host Mac must stay as close to stock as possible, and the dev machine must
be a throwaway, reproducible Raspberry Pi Zero 2 W. We need a way to reach the
Pi that does not require installing software on the host or joining the Pi to
venue Wi-Fi.

The Pi Zero 2 W exposes USB OTG on the inner ("USB") micro-USB port. A single
cable can carry both power and data.

## Decision

Use a **USB Ethernet gadget** so the host sees a new network interface and we
`ssh` over the cable:

- `dtoverlay=dwc2` + `modules-load=dwc2`, gadget brought up via libcomposite.
- Prefer **CDC-ECM** (`usb_f_ecm`) — macOS supports it natively; RNDIS is poorly
  supported on macOS.
- `usb0` gets a small static subnet; **avahi/mDNS** so `ssh user@pocketdev.local`
  resolves over the cable.
- WAN for Copilot CLI is a **separate, pluggable concern** (see docs/plan.md 3c): the
  default is host Internet Sharing over `usb0`; the Pi never joins venue Wi-Fi.

This removes the entire Azure remote-access stack (no JIT, no Entra, no VNC).

## Consequences

- One cable = power + data + SSH; nothing installed on the host beyond toggling
  macOS Internet Sharing (reversible).
- Glyphs are rendered by the **Mac's** terminal in this mode; the Nerd Font lives
  Mac-side.
- The access layer is host-agnostic (Mac today, iPad later), but iPadOS cannot
  share to a USB gadget — an iPad host therefore requires WAN provider #2 (Pi
  Wi-Fi), see docs/plan.md step 11.
- This SSH path is **co-equal** with the local-console path (ADR-0003); neither
  disables the other. `sshd` is unaffected by what renders on HDMI.
