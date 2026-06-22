# Project plan: "Pocket Dev" — ephemeral LazyVim + Copilot CLI on a Raspberry Pi (USB gadget + local console)

> Status: design/feasibility doc + phased build plan, distilled from the lessons of
> `azure-dev-machine`. **Host decided (2026-06-20): Pi 4 / Pi 5** — the Zero 2 W was
> imaged and tested but Copilot CLI was too sluggish interactively (see ADR-0002 and
> §3a). Earlier Zero-2-W-specific reasoning is retained below as exploration history.

## 0. Current status (updated 2026-06-22)

**Milestone reached: the cloud-init local-console build is implemented and
hardware-validated on a Pi 4.** A single NoCloud `cloud-init/user-data` reproduces the
hand-built environment end-to-end; all of the following were confirmed working on real
hardware after a clean reflash:

- Wi-Fi up on **first boot** (early `bootcmd` sets the Wi-Fi country + clears rfkill).
- Boots into the **cage + foot** no-DE console on `tty1` (US keyboard).
- **LazyVim** (idempotent clone) with the **snacks `<C-/>` large-float** terminal.
- **JetBrainsMono Nerd Font** glyphs + **Tokyo Night** `foot` theme.
- **Copilot CLI** installed (nvm + Node 22); **encrypted token vault** works
  (gnome-keyring Secret Service, unlocked via PAM — the fix was the separate
  `libpam-gnome-keyring` package).
- sudo requires the localuser password; repo reviewed safe to make public.

Known rough edge (accepted, not fixed): logging in **before cloud-init finishes** yields a
half-provisioned console (default font, no glyphs/theme) until you wait for
`cloud-init status --wait` and restart the console.

**Milestone landed** (PR #1, merged). **USB gadget mode is now implemented AND
hardware-validated** (PR #2, merged 2026-06-22): CDC-ECM over the Pi 4 USB-C port, with
cable-only SSH + internet (Mac Internet Sharing) confirmed on real hardware, including a
`wlan0`-off run. Two boot fixes were found and fixed during validation: RPi OS Lite ships
**sshd disabled** (now `systemctl enable --now ssh`) and NetworkManager left the
late-created **`usb0` unmanaged** (now a `conf.d` managed drop-in). The Mac-side client
setup (install **JetBrainsMono Nerd Font** + a terminal-agnostic `Ctrl+t` / `<Space>tt`
toggle, since Terminal.app can't send `<C-/>`) is documented in `docs/usb-gadget.md`.

**Remaining work:** the only open *feature* is **OSC52 clipboard**
(`feas-clipboard`/`build-clipboard`). Docs polish is **in progress** on branch
`docs-polish` (this change): a plug-in-and-go `docs/quickstart.md`, a trimmed
`docs/lazyvim-guide.md`, and a factory-reset section in `cloud-init/README.md`. The
`build-ipad` path stays P2/optional. See the updated §7 checklist.

## 1. Problem statement & goals

Keep the **host Mac pristine** (ideally 100% stock) and run all dev work on a **throwaway,
reproducible** machine that can be recreated at any moment. From the current Azure setup, the
operator found that the parts they actually love are a small subset:

- **Copilot CLI in a terminal that toggles with `Ctrl+/`** — which is actually LazyVim's
  `snacks.nvim` terminal (`<C-/>`), *not* i3. Confirmed in
  `azure-dev-machine/cloud-init.yaml:805-828`. This means the entire desktop layer is optional.
- **Nerd Font glyphs** for LazyVim/lazygit icons.
- **LazyVim** (Tokyo Night, LSP, Telescope, Neo-tree, lazygit, delta).

Everything else (browser, screenshot app, window manager, VNC) is unwanted. The provocative
target: do this on a **Pi Zero 2 W**, reached over **USB gadget mode** (plug the Pi into the
Mac; it appears as a USB network device; `ssh` over the cable — power + data on one wire).

### Success criteria
- Plug Pi into Mac → `ssh user@pocketdev.local` → `nvim` → `<C-/>` → Copilot CLI, on a stock Mac.
- The whole device image is reproducible and disposable (re-flash to "factory reset").
- Glyphs render; copy/paste works; Copilot CLI reaches the internet.

## 2. Keep vs. drop (distilled from `azure-dev-machine`)

| Keep | Drop |
|---|---|
| Neovim (latest stable) + LazyVim, Tokyo Night | i3 + polybar + picom + dunst (window manager/bar) |
| Copilot CLI via **nvm** Node 22, `<C-/>` snacks terminal | TigerVNC + the VNC-in-SSH tunnel |
| Nerd Font glyphs (now on the **Mac** terminal — optional) | kitty (terminal now lives on the Mac) |
| lazygit, git-delta, ripgrep/fd/fzf, gh | ksnip / xfce4-screenshooter / flameshot |
| `snacks.nvim` large-float terminal override | Firefox / any browser |
| Reproducible first-boot provisioning (cloud-init → Pi equivalent) | XFCE fallback desktop |
| Ephemeral / disposable rebuild philosophy | Azure control plane: JIT, Entra SSH, Defender Plan 2, self-deallocate/auto-shutdown, `provision.sh`/`connect.sh`/`stop.sh`/`sync.sh`, `vm.*.conf` |
| `gh` CLI, build-essential, python venv/pipx | The `azureuser` locked service-account model (Pi has its own user/access model) |
| No-DE local console (kmscon / cage+foot) for a monitor+keyboard — **co-equal front door, try first**; glyphs + truecolor, coexists with SSH | A full desktop environment (XFCE/GNOME/X11 WM) just to drive a local screen |

## 3. Phase 1 — Feasibility analysis (Pi Zero 2 W primary; fallback if it can't cope)

Deliverable: a short written verdict (could become ADR-0001 of the new repo) on each axis below,
with a clear **go / fallback** recommendation. Items marked *(bench)* require hands-on testing.

### 3a. Compute & memory — THE binding constraint
> **VERDICT (2026-06-20, hardware bench): Zero 2 W REJECTED as primary.** On a real
> Zero 2 W, Copilot CLI runs but is too sluggish for interactive use (even typing to
> exit is painful) — fails the daily-driver bar on its own. **Fall back to Pi 4 / Pi 5**
> (see ADR-0002, §"Fallback options"). The notes below are retained for context.
- Pi Zero 2 W = quad-core Cortex-A53 @ 1GHz, **512 MB RAM**, arm64. Copilot CLI, Neovim,
  treesitter, and LSP servers all run on arm64 fine; **RAM is the risk**, not CPU/arch.
- Rough budget: Copilot CLI (Node) ~150-300 MB; Neovim + a few LSPs ~200-400 MB. On 512 MB this
  is tight and will lean on swap.
- Mitigations to evaluate *(bench)*: **zram** swap (compressed RAM), modest SD/USB swap file,
  a **minimal LSP set** (lazy-load, few servers), trim treesitter parsers, `lua_ls`/heavy
  plugins disabled, `vim.g.lazyvim_*` tuning, headless `:Lazy! sync` at build time.
- **Decision rule:** if a normal session (nvim + 1-2 LSPs + Copilot CLI) thrashes swap or OOM-kills,
  trigger the fallback (Phase 1f).

### 3b. Access model — USB gadget networking (the headline idea)
- Pi Zero 2 W supports **USB OTG** on the data micro-USB port (the inner port; the outer is
  power-only). Configure a **USB Ethernet gadget** so the host sees a new network interface.
- Mechanism *(bench)*: `dtoverlay=dwc2` + `modules-load=dwc2` and a gadget driver. Prefer
  **CDC-ECM** (`g_ether`/libcomposite `usb_f_ecm`) — macOS supports CDC-ECM natively; RNDIS is
  poorly supported on macOS. Single cable carries **power + data**.
- Addressing: avahi/mDNS so `ssh user@pocketdev.local` resolves; assign a small static subnet on
  `usb0` (optionally run `dnsmasq` on the Pi to DHCP the host, or rely on link-local).
- **Net effect:** removes the entire Azure remote-access stack (no JIT, no Entra, no Tailscale).
- **Design for swappability (IMPORTANT):** the access layer is the same regardless of host (Mac
  today, iPad later) — a CDC-ECM gadget + `usb0` + mDNS. Keep the gadget/SSH bring-up independent
  of *how the Pi gets WAN* (§3c) so the internet-provider decision can be swapped without touching
  the access model.

### 3c. Internet path for Copilot CLI — pluggable WAN provider (gadget mode has no WAN by default)
Copilot CLI **requires** internet. Treat the WAN source as a **swappable module** behind a stable
interface (the Pi just needs a default route + DNS on boot); the chosen provider should be a single
config switch, not hard-wired. **Priority order:**

1. **Host Internet Sharing → USB gadget (PREFERRED/P1, Mac).** Pi piggybacks on *whatever the Mac
   is connected to* — home, Starbucks, a phone hotspot — **without the Pi ever joining those
   networks** (no Pi Wi‑Fi creds, no captive-portal hassle on the Pi, nothing to reconfigure when
   the Mac roams). The Mac NATs the gadget subnet out its current uplink; captive portals are
   authenticated **in the Mac's browser**, then the Pi inherits connectivity. Only concession:
   toggling macOS *Internet Sharing* (reversible, no installed software). *(bench: CDC-ECM vs RNDIS
   with Internet Sharing; behavior after the Mac roams networks.)*
2. **Pi onboard Wi‑Fi for WAN (FALLBACK).** Used when host sharing is unavailable/blocked or on a
   trusted home network; Wi‑Fi creds baked via a gitignored overlay. (This is also the path that
   makes an **iPad host** workable — see Phase 2, since iPadOS can't share to a USB gadget.)
3. **Pi cellular / its own hotspot (FUTURE/optional).** Fully host-independent WAN; heaviest.

- **Recommendation:** ship P1 (host Internet Sharing) as the default *and* keep provider #2 a
  one-switch swap, because the iPad-host idea (Phase 2) depends on it.

### 3d. Glyphs on a (preferably) stock Mac
- Direct SSH means the **Mac's terminal** renders glyphs, so the Nerd Font lives Mac-side. Per the
  operator's preference ("prefer stock, document font as optional"): the build works with stock
  Terminal.app (icons may show as tofu); installing **JetBrainsMono Nerd Font** + selecting it in
  Terminal/iTerm is documented as the **optional enhancement** that makes LazyVim/lazygit icons
  render. No other Mac software needed.

### 3e-bis. Local console — monitor + keyboard, **no desktop environment** *(bench — TRY THIS FIRST)*
A **co-equal, independent** front door alongside the USB-gadget+SSH path (same priority, not a
fallback — both coexist on the same Pi and the same LazyVim config; one is HDMI output, the other is
`sshd` over `usb0` — neither disables the other). **This is the option to validate FIRST when we
start building.** **No desktop environment (X11/Wayland WM, panels, compositor chrome) is
required** — LazyVim is just a terminal program. What it *does* need is a console surface that
renders **Nerd Font glyphs** and **Tokyo Night truecolor**, which the stock kernel framebuffer
console (`getty` on `tty1`, PSF bitmap font, ≤16 colors) **cannot** do (icons → tofu, washed-out
colors). No-DE options:
1. **cage + foot (RECOMMENDED — both packaged in Bookworm/Pi OS).** `cage` = single-app Wayland
   kiosk (no WM chrome), `foot` = tiny truecolor terminal with excellent glyphs; boot straight into
   `cage foot` → `nvim`. Pulls in Wayland/libdrm/Mesa but is still far lighter than XFCE on the
   512 MB Zero 2 W, and — crucially — **installs from apt with no source build**.
2. **kmscon (NOT RECOMMENDED — unavailable).** *Feasibility finding (2026-06-20):* kmscon was
   **removed from Debian before Bullseye and is not in Bookworm / Raspberry Pi OS apt**; it is
   unmaintained. Usable only via an old source build with dependency pain — avoid unless cage+foot
   fails. (Same story for **fbterm**, also removed.)
- **Font lives on the Pi** in this mode (vs. on the Mac in the SSH mode).
- **WAN caveat (ties to §3c):** with no Mac attached there's no host Internet Sharing, so Copilot CLI
  needs the Pi's **own WAN (Wi‑Fi provider #2)**. Plain LazyVim editing + LSP work fully offline;
  only Copilot CLI requires the route.
- **Decision rule:** co-equal with the SSH path and the **first build target to try**; still must
  never pull in a desktop stack or compromise the headless capability (SSH must keep working too).

### 3e. Clipboard
- Over direct SSH (unlike VNC), **OSC 52** can sync yank→Mac clipboard if the Mac terminal
  supports it (iTerm2 yes; Terminal.app limited). Evaluate OSC52 via snacks/`vim.clipboard` or
  `ojroques/nvim-osc52`. This is strictly better than the VNC clipboard limitation noted in
  ADR-0003. *(bench)*

### 3f. Ephemerality & reproducibility
- "Throw away and recreate" on a Pi = a **reproducible flashable image** + a **first-boot
  provisioner**. Options weighed:
  - **cloud-init (RECOMMENDED — primary).** **Raspberry Pi OS Bookworm and newer now officially
    support cloud-init** via the **NoCloud** datasource: drop `user-data` / `meta-data` / optional
    `network-config` on the FAT32 boot partition (`/boot/firmware`) and it self-configures on first
    boot. Ubuntu Server arm64 on the Pi has supported this for years. **No control machine and no
    SSH needed** — the Pi configures itself from files on the SD card, which fits the disposable /
    "re-flash = factory reset" philosophy. Big head start: much of `azure-dev-machine/cloud-init.yaml`
    (users, packages, `runcmd`, `write_files`) **ports over with the same mental model**.
    *(Source: raspberrypi.com/news/cloud-init-on-raspberry-pi-os.)*
  - **`rpi-imager` customization + a first-boot script** (simplest; subset of what cloud-init does —
    fine for trivial cases, but cloud-init supersedes it).
  - **Ansible playbook (optional later add-on, NOT primary).** Most portable across Pi models / a
    cloud-VM fallback, and idempotent/re-runnable any time — but it adds a dependency: either a
    **push** model from a separate **control machine over SSH** (Ansible's control node needs
    Linux/macOS/WSL, not Windows), or running it **locally on the Pi** (`--connection=local` /
    `ansible-pull` from a git repo, no second machine). Its main win — managing long-lived machines
    you *don't* re-flash — matters less when the whole premise is throw-away images.
- **Recommendation:** **cloud-init as the primary provisioner** (no control machine; reuse the
  azure-dev-machine config), with a thin `rpi-imager`/image-prep step. Keep **Ansible as an optional
  later layer** if/when an idempotent re-runnable app layer or a shared Pi-or-cloud-VM playbook is
  wanted. Caveats either way: cloud-init runs mainly **first boot**, and the **Copilot CLI device
  login still needs a human**, so full unattended provisioning stops just short of auth.

### 3g. Power & stability *(bench)*
- Confirm the Mac USB port reliably powers a Zero 2 W under compile/LSP load (spikes to ~0.5A);
  watch for under-voltage throttling. Document a powered-USB-hub fallback if needed.

### Fallback options (if Zero 2 W fails 3a or 3g)
1. **Pi 4 / Pi 5 (2-8 GB RAM).** Same gadget-mode approach (Pi 4 OTG via USB-C; Pi 5 OTG support
   differs — verify). Removes the RAM constraint entirely.
2. **Minimal cloud VM** (reuse large parts of `azure-dev-machine`, stripped of the desktop). Loses
   the "plug-in-USB" magic but keeps reproducibility; access reverts to SSH (JIT/Tailscale).

## 4. Phase 2 — Build plan for the winning option

(Executed only after Phase 1 picks a host. Tasks are written generically; the provisioner is
**cloud-init per 3f** (NoCloud on the boot partition), with Ansible as an optional later layer.)

1. **Repo skeleton** for the new project: `README.md`, `docs/decisions/` (seed ADR-0001 access
   model = USB gadget, ADR-0002 host choice from Phase 1), `.gitignore` (ignore any Wi‑Fi/secret
   overlays — never commit creds, mirroring this repo's `connect.*.local` rule).
2. **Base image prep**: choose OS (Pi OS Lite arm64 vs Ubuntu Server arm64), enable SSH, set
   hostname `pocketdev`, create the user, install avahi.
3. **USB gadget networking**: `dtoverlay=dwc2`, CDC-ECM gadget bring-up, `usb0` static addressing,
   mDNS; document the macOS side (one-liner `ssh`, optional Internet Sharing for 3c option 2).
4. **Internet for Copilot CLI (pluggable WAN — design as a swap point)**: implement WAN as a
   single switchable provider behind a stable "Pi has a default route + DNS" contract.
   **P1/default = host Internet Sharing** over `usb0` (Pi takes DHCP + default route from the Mac;
   works at Starbucks/hotspot without the Pi joining any venue Wi‑Fi). **Swap target = Pi Wi‑Fi**
   (gitignored creds overlay), which is also what an iPad host requires (step 11). Cellular is a
   later, optional provider. Keep this orthogonal to the gadget/SSH bring-up (step 3).
5. **Toolchain (reuse from `azure-dev-machine/cloud-init.yaml`)**: build-essential, git, curl,
   unzip, python venv/pipx, **gh**, ripgrep/fd/fzf, git-delta, lazygit (GitHub release binary),
   latest Neovim (release, not apt), **nvm + Node 22 + `@github/copilot`**.
6. **LazyVim**: clone starter, strip git history, add the `snacks.nvim` large-float terminal
   override (`<C-/>`), pre-`:Lazy! sync` headless. Add a **low-RAM profile** (trimmed LSP/parsers)
   gated for the Zero 2 W.
7. **Glyphs**: ensure font config is a Mac-side doc step (optional); no server-side VNC font needed.
   *(For the co-equal local console — step 7b — also install JetBrainsMono Nerd Font on the Pi.)*
7b. **Local console (no DE) — BUILD/VALIDATE THIS FIRST**: install a glyph+truecolor console —
    **kmscon** (preferred: replace `getty` on a VT, TrueType Nerd Font, truecolor) or **cage + foot**
    kiosk; boot into `nvim`. Co-equal with the SSH path (both coexist), install the Nerd Font on the
    Pi, and note Copilot CLI needs Pi Wi‑Fi WAN (#2) when no Mac is attached. *(needs: build-lazyvim,
    build-wifi; gated by feas-localconsole)*
8. **Clipboard**: wire OSC52 yank if 3e validates.
9. **Ephemerality**: document the "re-flash / re-run playbook = factory reset" workflow; make the
   provisioner idempotent so a rebuild is one command.
10. **Docs**: port a trimmed `lazyvim-for-vscode-users.md`; write a short "plug in & go" quickstart;
    record decisions as ADRs.
11. **iPad host support — P2 (may or may not happen, design for it but don't block on it).** Lower
    priority than the Mac path, but the build must not *preclude* it:
    - Because **iPadOS can't NAT/share to a USB gadget**, an iPad host **requires WAN provider #2
      (Pi Wi‑Fi or cellular)** from step 4 — which is exactly why step 4 is built as a swap point.
    - Client side is app-level, nothing "installed on the host OS" beyond an App Store app:
      **Blink Shell** (SSH+mosh, OSC52 clipboard, installable Nerd Font) or Termius; pair a
      hardware keyboard.
    - USB‑C iPad + CDC‑ECM gadget for the SSH link; Lightning iPad needs a **powered Camera
      Connection Kit**. Treat as a documented optional path / future ADR, validated only if pursued.

## 5. Open questions to resolve during Phase 1
- Zero 2 W 512 MB: does a real session survive without painful swap thrash? (3a — the make/break.)
- macOS + CDC-ECM gadget: does it come up reliably across macOS versions without kexts? (3b)
- Internet: confirm **Mac Internet Sharing → USB gadget** works reliably (incl. CDC-ECM vs RNDIS
  with Internet Sharing, and after the Mac roams between networks/captive portals). (3c)
- OS/provisioner: confirmed direction = **cloud-init (NoCloud) primary** on Pi OS Lite arm64; open
  sub-question is just Pi OS Lite vs Ubuntu Server arm64 for the base image. (3f)
- Pi 5 OTG gadget support specifics, if fallback to Pi 5.
- Local console (3e-bis — **try first**): does **kmscon** (or **cage+foot**) build/run on the chosen
  OS and render Nerd Font glyphs + Tokyo Night truecolor on the Zero 2 W without pulling in a desktop,
  at acceptable RAM cost? (Co-equal with SSH; the first path to validate.)
- iPad host (P2): does a USB‑C iPad reliably bring up the CDC‑ECM gadget link, and is Blink Shell's
  Nerd Font + OSC52 sufficient? (Only if/when the iPad path is pursued; depends on WAN provider #2.)

## 6. Notes / guardrails carried over from this repo
- **Never commit secrets/network details** (Wi‑Fi creds, IPs) — use gitignored overlays.
- First-boot/image config that is fed to tooling should stay ASCII where the toolchain is picky.
- Keep provisioning **idempotent and reproducible**; a rebuild must restore the full setup.
- Never commit/push without explicit user approval.

## 7. Todo checklist (portable copy of the tracked task list)

> Mirrors the session task tracker so the breakdown travels with this file. Phase-2 items list
> their blocking dependencies in parentheses. Order within a phase is roughly execution order.

### Phase 0 — Land the validated milestone
- [x] **land-milestone** — DONE: PR #1 (local-console build) merged; PR #2 (USB gadget
      mode) merged 2026-06-22. Follow-on work: OSC52 clipboard + docs polish (this branch).

### Phase 1 — Feasibility (do these first; no hard inter-deps)
- [~] **feas-ram** — N/A: Zero 2 W rejected as primary host (ADR-0002); Pi 4/5 removes the
      RAM constraint. Retained for history.
- [x] **feas-gadget** — DONE (HW 2026-06-21): dwc2 CDC-ECM gadget up on the Pi 4 USB-C port;
      Mac sees "Pocket Dev USB Ethernet"; `usb0` managed + addressed; mDNS/SSH over the cable work.
- [x] **feas-internet** — DONE (HW 2026-06-21): Mac Internet Sharing -> `usb0` gave the Pi
      192.168.2.2 + default route (metric 200, preferred over Wi-Fi); `ping github.com` over the
      cable succeeds. CDC-ECM worked natively on macOS (no RNDIS needed). Pi Wi-Fi = fallback.
- [x] **feas-glyphs** — DONE both sides: Pi-side JetBrainsMono Nerd Font glyphs render in
      the cage+foot console; Mac-side client-font setup (Homebrew `font-jetbrains-mono-nerd-font`
      + Terminal.app profile font) is documented in `docs/usb-gadget.md` and HW-validated.
- [x] **feas-localconsole** — DONE on Pi 4: **cage+foot** renders Nerd Font glyphs + Tokyo Night
      truecolor, coexists with SSH. (kmscon confirmed unavailable in Bookworm; cage+foot chosen.)
- [ ] **feas-clipboard** — Test OSC52 yank→host clipboard over SSH (iTerm2/Terminal.app/Blink);
      wire snacks/nvim-osc52 if it works.
- [x] **feas-repro** — DONE: cloud-init (NoCloud) chosen and implemented; reflash = factory reset.
- [~] **feas-power** — N/A for Pi 4 host (Zero 2 W power concern moot).
- [x] **feas-fallback** — DONE: Pi 4/5 + cloud-VM fallbacks documented (ADR-0002).

### Phase 2 — Build (after Phase 1 picks a host)
- [x] **build-skeleton** — DONE: repo scaffolded (README, `docs/decisions/`, `.gitignore` for
      secret overlays, `cloud-init/`).
- [x] **build-image** — DONE via cloud-init: Pi OS arm64, SSH, hostname `pocketdev`, localuser,
      avahi/mDNS.
- [x] **build-gadget** — DONE, **hardware-validated 2026-06-21** (Pi 4 + Mac): `config.txt`
      `dtoverlay=dwc2,dr_mode=peripheral` + `cmdline.txt` `modules-load=dwc2`, a libcomposite
      **CDC-ECM** bring-up script + `pocketdev-usb-gadget.service`, a DHCP-client `usb0`
      NetworkManager connection, and a `conf.d` drop-in forcing NM to manage the late-created
      `usb0` netdev. After reboot `usb0` auto-comes-up at 192.168.2.2 with a default route;
      `ssh localuser@192.168.2.2` over the cable + internet-over-cable (Mac Internet Sharing)
      both work. Also fixed: cloud-init now `systemctl enable --now ssh` (RPi OS Lite ships
      sshd disabled). Host-side doc: `docs/usb-gadget.md`. *(needs: build-image, feas-gadget)*
- [x] **build-wifi** — DONE: P1 default (host Internet Sharing over `usb0`) HW-validated
      2026-06-21; Pi onboard Wi-Fi (provider #2) validated as fallback. WAN is pluggable: usb0
      default route (metric 200) preempts wlan0 (600) when the cable has WAN. *(needs: feas-internet)*
- [x] **build-toolchain** — DONE: build-essential, gh, ripgrep/fd/fzf, delta, lazygit, Neovim
      release, nvm + Node 22 + `@github/copilot` (all installed by user-data, validated).
- [x] **build-lazyvim** — DONE: idempotent LazyVim clone, snacks large-float `<C-/>` override,
      headless `:Lazy! sync` (validated). Low-RAM profile not needed on Pi 4.
- [ ] **build-clipboard** — Wire OSC52 yank if feas-clipboard validated.
      *(needs: build-lazyvim, feas-clipboard)*
- [~] **build-ephemeral** — In progress (docs-polish branch): provisioner is idempotent
      (LazyVim clone fixed), reflash = factory reset. Adding a factory-reset / ephemerality
      section to `cloud-init/README.md`. *(needs: build-toolchain)*
- [~] **build-docs** — In progress (docs-polish branch): adding a plug-in-and-go
      `docs/quickstart.md` (both front doors) + a trimmed `docs/lazyvim-guide.md`, on top of
      the existing `docs/quickstart-local-console.md` + `cloud-init/README.md`.
      *(needs: build-lazyvim)*
- [x] **build-localconsole** — DONE on Pi 4: cage+foot kiosk, Nerd Font, boots into LazyVim,
      Tokyo Night, encrypted Copilot vault; coexists with SSH. *(was: build first — done)*
- [ ] **build-ipad** — *(P2, may or may not happen)* Ensure design doesn't preclude an iPad host:
      iPad needs WAN provider #2 (Pi Wi‑Fi/cellular) since iPadOS can't share to a USB gadget.
      Client = Blink Shell (SSH+mosh, OSC52, Nerd Font) + HW keyboard; USB‑C CDC-ECM link, Lightning
      needs powered CCK. Document as optional path / future ADR. *(needs: build-lazyvim, build-wifi)*
