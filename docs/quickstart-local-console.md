# Runbook: image a Pi 4 / Pi 5 for the local-console-first LazyVim setup

Goal: flash an SD card on a separate machine, boot the Pi on a **monitor +
keyboard** (no desktop environment), and reach **LazyVim (Tokyo Night) with Nerd
Font glyphs** in a fullscreen truecolor terminal. This doubles as the
**`feas-localconsole` / `feas-glyphs` / `feas-internet` bench** —
checkpoints are marked **[VERIFY]** so results can feed the ADRs.

> Console engine = **cage + foot** (kmscon/fbterm are removed from Bookworm — see
> ADR-0003). WAN for Copilot CLI in this mode = the Pi's own **Wi-Fi** (no Mac
> attached), configured at imaging time.
>
> **Host:** Raspberry Pi 4 or Pi 5 (≥2 GB; 4 GB recommended). The **Zero 2 W was
> tried and rejected** — Copilot CLI ran but was too sluggish for interactive use
> (ADR-0002). Everything else in this runbook is identical regardless of board.

---

## 0. Hardware checklist

- Raspberry Pi 4 or Pi 5 (≥2 GB RAM; 4 GB recommended)
- microSD card (16 GB+) and a card reader on your imaging machine
- **micro-HDMI → HDMI** cable/adapter to the monitor (Pi 4/5 have two micro-HDMI
  ports; use the one nearest the USB-C power port = `HDMI0` / `HDMI-A-1`)
- USB keyboard into any of the Pi's **USB-A** ports (no OTG adapter needed)
- USB-C power supply (use an official/3 A supply, especially on Pi 5)
- Imaging machine with **Raspberry Pi Imager** (macOS/Linux/Windows)

---

## 1. Flash the card (Raspberry Pi Imager)

1. Install Imager from <https://www.raspberrypi.com/software/> and launch it.
2. **Choose Device:** Raspberry Pi 4 (or Pi 5).
3. **Choose OS:** Raspberry Pi OS (other) → **Raspberry Pi OS Lite (64-bit)**.
   - 64-bit is required (Copilot CLI / Node need arm64, not armhf). Lite = no
     desktop, which is exactly what we want.
4. **Choose Storage:** your microSD.
5. Click **Next → Edit Settings** (the OS customization gear) and set:
   - **Hostname:** `pocketdev`
   - **Enable SSH** → "Use password authentication" (or paste a public key).
   - **Username / password:** pick your user (this runbook assumes `localuser`).
   - **Configure wireless LAN:** your home Wi-Fi SSID + password + country.
     *(This is WAN provider #2 — it gives Copilot CLI internet and a bonus SSH
     route while we bench the console. It does NOT commit us to Wi-Fi for the
     final design.)*
   - **Locale / keyboard:** set your layout (avoids surprises at the console).
6. **Save → Yes (apply customisation) → Yes (erase & write).** Wait for verify.

---

## 1b. HDMI note (Pi 4/5 + Bookworm: usually nothing to do)

On Pi 4/5 with Bookworm's default **KMS** driver, HDMI **just works** as long as
the **monitor is connected and powered on before you boot** the Pi (boot it
headless and no HDMI output is produced). The old `hdmi_force_hotplug` /
`hdmi_safe` / `hdmi_group` / `hdmi_mode` knobs are **ignored under KMS** — don't
bother with them.

**Only if you still get "No Signal"** with a connected, powered monitor:
- First suspect the **cable/adapter or port** (use a known-good micro-HDMI cable;
  try the other port).
- To pin a specific mode, edit **`cmdline.txt`** on the **`bootfs`** partition
  (mounted at `/boot/firmware/` on the Pi) — it's a single line; append this
  token, space-separated:
  ```
  video=HDMI-A-1:1920x1080@60
  ```
  Use a resolution your monitor supports. `HDMI-A-1` = the port nearest the USB-C
  power; use `HDMI-A-2` for the other one.

> Leave the default `dtoverlay=vc4-kms-v3d` (KMS) in place — `cage`/`foot`/wlroots
> need KMS/DRM in step 8.

---

## 2. First boot on monitor + keyboard

1. With the monitor connected and powered **before boot**, connect the USB
   keyboard and apply USB-C power. First boot resizes the filesystem and reboots
   once.
2. Log in as `dev` at the console.
3. **[VERIFY feas-internet]** confirm Wi-Fi + WAN:
   ```sh
   ip -brief addr            # wlan0 should have an IP
   ping -c3 github.com       # WAN reachable
   ```
4. Baseline RAM snapshot for later comparison:
   ```sh
   free -h
   ```

> Tip: from another computer you can now also `ssh localuser@pocketdev.local` over
> Wi-Fi, which makes copy/paste of the commands below much easier. The console is
> still the thing under test.

---

## 3. System update + toolchain

```sh
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git curl unzip build-essential \
  ripgrep fd-find fzf python3-venv pipx
# Debian names fd as fdfind; expose it as fd:
mkdir -p ~/.local/bin && ln -sf "$(command -v fdfind)" ~/.local/bin/fd
```

lazygit + git-delta (grab the arm64 release binaries):

```sh
# lazygit — resolve latest version via the release redirect (no API rate limit)
LG=$(curl -fsSLI -o /dev/null -w '%{url_effective}' \
      https://github.com/jesseduffield/lazygit/releases/latest \
     | grep -oP 'tag/v\K[^/]+')
curl -L -o /tmp/lg.tgz \
  "https://github.com/jesseduffield/lazygit/releases/download/v${LG}/lazygit_${LG}_linux_arm64.tar.gz"
sudo tar -C /usr/local/bin -xf /tmp/lg.tgz lazygit

# git-delta (Debian package is fine)
sudo apt install -y git-delta || true
```

---

## 4. Neovim (latest stable, arm64 tarball)

The apt Neovim is too old for LazyVim. Use the official release tarball:

```sh
curl -L -o /tmp/nvim.tgz \
  https://github.com/neovim/neovim/releases/latest/download/nvim-linux-arm64.tar.gz
sudo rm -rf /opt/nvim && sudo mkdir -p /opt/nvim
sudo tar -C /opt/nvim --strip-components=1 -xf /tmp/nvim.tgz
sudo ln -sf /opt/nvim/bin/nvim /usr/local/bin/nvim
nvim --version | head -1     # expect v0.11+ (v0.12.x at time of writing)
```

---

## 5. Node + Copilot CLI (nvm → Node 22)

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
export NVM_DIR="$HOME/.nvm" && . "$NVM_DIR/nvm.sh"
nvm install 22 && nvm use 22
npm install -g @github/copilot     # the GitHub Copilot CLI package
```

**[VERIFY feas-internet, login]** launch and authenticate:

```sh
copilot            # follow the device-code prompt in a browser, then /exit
```

---

## 6. LazyVim (Tokyo Night is the default theme)

```sh
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
# Pre-install plugins headlessly (first run pulls a lot):
nvim --headless "+Lazy! sync" +qa
```

LazyVim ships **Tokyo Night** as its default colorscheme, so no extra config is
needed to satisfy the theme requirement. (We will add the `<C-/>` snacks
terminal + a low-RAM LSP profile in a later step; for the bench, stock LazyVim is
enough.)

---

## 7. Nerd Font on the Pi (glyphs live Pi-side in this mode)

Pi OS Lite ships without **fontconfig** (no desktop), so `fc-cache` / `fc-list`
aren't present and `foot` can't resolve a font by name. Install it first.

Also install a **color-emoji font**: Neovim's `:checkhealth` uses emoji markers
(✅ ❌ ⚠️) that are **not** in JetBrainsMono Nerd Font, so without an emoji font
those specific markers render as tofu boxes even though all other glyphs are fine.

```sh
sudo apt install -y fontconfig fonts-noto-color-emoji

mkdir -p ~/.local/share/fonts
curl -L -o /tmp/jbm.zip \
  https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip -o /tmp/jbm.zip -d ~/.local/share/fonts/JetBrainsMonoNerdFont
fc-cache -f
fc-list | grep -i "JetBrainsMono Nerd" | head     # confirm it registered
```

> Tip: emoji glyphs only appear in terminals **launched after** `fc-cache -f`, so
> start (or restart) `foot` after this step. If you'd rather not show emoji at all,
> strip them from the health report instead with this autocmd in your nvim config:
> `autocmd FileType checkhealth :set modifiable | silent! %s/\v( ?[^\x00-\x7F])//g`

---

## 8. The no-DE console: cage + foot

```sh
sudo apt install -y cage foot
# foot config: Nerd Font + truecolor + Tokyo Night palette (matches LazyVim)
mkdir -p ~/.config/foot
cat > ~/.config/foot/foot.ini <<'EOF'
font=JetBrainsMono Nerd Font:size=12

[colors]
alpha=1.0
background=1a1b26
foreground=c0caf5
regular0=15161e
regular1=f7768e
regular2=9ece6a
regular3=e0af68
regular4=7aa2f7
regular5=bb9af7
regular6=7dcfff
regular7=a9b1d6
bright0=414868
bright1=f7768e
bright2=9ece6a
bright3=e0af68
bright4=7aa2f7
bright5=bb9af7
bright6=7dcfff
bright7=c0caf5
EOF
```

**[VERIFY feas-localconsole + feas-glyphs]** from the **console** (tty1, logged
in as `dev`), launch a fullscreen terminal straight into Neovim:

```sh
cage -s -- foot -e nvim
```

Inside Neovim, check the three requirements:
- **Glyphs:** open the file tree (`<leader>e`) and lazygit (`<leader>gg`) — folder
  / git / devicons should render as **icons, not boxes (tofu)**.
- **Theme/truecolor:** Tokyo Night colors should look right (deep blue bg, vivid
  syntax). Run `:checkhealth` and confirm no truecolor/term warnings. (Its ✅/❌/⚠️
  summary markers are emoji — they render only if `fonts-noto-color-emoji` was
  installed in §7 and `foot` was started after `fc-cache`; otherwise they show as
  tofu while every other glyph is fine. Cosmetic.)
- **Copilot CLI:** `:terminal copilot` (or `<C-/>` once the snacks override is
  added) and confirm it reaches the service.

If `cage` complains about DRM/seat permissions, add the user to the right groups
and re-login:
```sh
sudo usermod -aG video,render,input dev
```

**[VERIFY ram sanity]** in a second console (Ctrl+Alt+F2) or over SSH, watch memory
while nvim + an LSP + Copilot CLI are all running:
```sh
free -h          # on a 2-4 GB Pi 4/5 there should be ample headroom
```
On the Pi 4/5 RAM is no longer the binding constraint (that's why the Zero 2 W was
dropped — ADR-0002); this is just a sanity check that nothing is pathological.

---

## 9. (After validation) launch the console automatically — but require login

Only once the checks pass, make the console come up on its own — **without
autologin**: the Pi shows the normal `tty1` login prompt asking for your
**username and password**, and only *after* a successful login does it launch the
fullscreen `foot` terminal. (If you had previously added the autologin drop-in,
remove it: `sudo rm -f /etc/systemd/system/getty@tty1.service.d/autologin.conf`
then `sudo systemctl daemon-reload`.)

**Launch a shell inside `foot`, not `nvim` directly.** If you exec `foot -e nvim`,
quitting nvim ends the session and you'd be dropped back to a re-login — launching
a **shell** gives you a real terminal: `cd` wherever you want, type
`nvim`/`nvim .`/`nvim <file>` to edit, and quitting nvim drops you back to that
shell.

```sh
# On tty1 login (after you type username + password), launch a fullscreen shell
# (and only on tty1). The default getty already prompts for credentials.
cat >> ~/.bash_profile <<'EOF'

if [ "$(tty)" = "/dev/tty1" ]; then
  exec cage -s -- foot
fi
EOF
```

Reboot: the Pi shows a login prompt on the monitor; enter your username and
password, and it comes up in a fullscreen `foot` terminal in your home directory.
From there:
- `cd` to wherever your code is, then `nvim .` (or `nvim path/to/file`) opens
  LazyVim where you want.
- Inside LazyVim, navigate/switch roots with `<leader>ff` (find files),
  `<leader>e` (explorer), `:cd <dir>`, or the `<C-/>` snacks terminal for shell
  commands — then `:q` returns to the foot shell, not a dead end.

Other ways in remain available regardless: switch VTs with **Ctrl+Alt+F2** for a
plain login shell, and SSH over Wi-Fi still works independently (ADR-0001 co-equal
path).

---

## What to report back (feeds the ADRs / plan todos)

- **feas-localconsole:** did `cage -s -- foot` (with `nvim` inside) come up
  fullscreen with no desktop? Any DRM/seat errors?
- **feas-glyphs:** do LazyVim/lazygit icons render (yes/tofu)?
- **feas-internet:** did `copilot` authenticate and respond over Wi-Fi?
- **responsiveness:** is Copilot CLI / nvim comfortably interactive on the Pi 4/5
  (the bar the Zero 2 W failed)?

These map to the Phase 1 checklist in `docs/plan.md` §7. (Host RAM/power are no longer
make/break now that the host is a Pi 4/5 — see ADR-0002.)
