# cloud-init provisioning (NoCloud) — automated Pocket Dev build

This directory reproduces the validated Pi 4 local-console build
(`docs/quickstart-local-console.md`) automatically on first boot, using
cloud-init's **NoCloud** datasource. No control machine, no SSH, no Ansible —
cloud-init reads these files straight off the SD card's boot partition.

Raspberry Pi OS Bookworm (and newer) ships with cloud-init and enables the
NoCloud datasource when it finds `user-data` on the FAT32 boot partition. See
<https://www.raspberrypi.com/news/cloud-init-on-raspberry-pi-os/>.

## Files

| File                     | Purpose                                              |
|--------------------------|------------------------------------------------------|
| `user-data`              | The `#cloud-config` that installs the whole stack.   |
| `meta-data`              | Minimal instance-id / hostname (required by NoCloud).|
| `network-config.example` | Wi-Fi template — copy to `network-config` and fill.  |

## What `user-data` builds

Everything from the hand runbook, in one shot:

- A non-autologin user `localuser` (you log in with username + password on the
  console; `sudo` also prompts for that password - no passwordless sudo).
- Toolchain: git, build-essential, ripgrep, fd, fzf, git-delta, gh, pipx, python3-venv.
- **Neovim** (latest stable, arm64 tarball) + **LazyVim** starter (git history stripped).
- **lazygit** (arm64, via the release-redirect version trick — the api.github.com
  method is rate-limited on shared IPs).
- **Node 22 via nvm** + `@github/copilot` (the Copilot CLI).
- **cage + foot** no-desktop console, JetBrainsMono Nerd Font, noto-color-emoji,
  and a Tokyo Night `foot.ini` so the shell/TUI match LazyVim.
- A `tty1` launcher (`/etc/profile.d/10-pocketdev-console.sh`) that drops you into
  the fullscreen console after login — quitting nvim returns you to a shell.
- **US keyboard layout** and the **Wi-Fi regulatory country** set early (see the
  Networking note below) so the radio isn't rfkill-blocked on first boot.
- **Encrypted Copilot CLI token storage** via a headless **gnome-keyring** Secret
  Service. The login keyring is unlocked by your console password (PAM), and the
  tty1 launcher starts the `secrets` component on the per-user D-Bus bus, so the
  CLI's keytar backend stores the token encrypted instead of warning
  `system vault not available`. Verify with `secret-tool` (see Troubleshooting).
- **USB gadget mode** (CDC-ECM Ethernet over the Pi 4 **USB-C** port) so you can
  `ssh localuser@pocketdev.local` over a single cable - no monitor, no Wi-Fi setup.
  Activates after the first reboot. See [`docs/usb-gadget.md`](../docs/usb-gadget.md)
  for the macOS host side (incl. Internet Sharing for Copilot CLI WAN).

## Image a new SD card and apply this config

You need: a microSD card (16 GB+), a card reader, and the
[Raspberry Pi Imager](https://www.raspberrypi.com/software/) (macOS, Linux, or
Windows). The whole flow is: **flash the OS -> drop these files on the boot
partition -> boot the Pi**.

### Step 1 - Flash Raspberry Pi OS Lite (64-bit)

1. Insert the SD card and open **Raspberry Pi Imager**.
2. **Choose Device:** Raspberry Pi 4 (or Pi 5).
3. **Choose OS:** `Raspberry Pi OS (other)` -> **`Raspberry Pi OS Lite (64-bit)`**
   (no desktop; this build provides its own console).
4. **Choose Storage:** your SD card. Double-check you picked the card, not an
   internal disk.
5. When asked **"Would you like to apply OS customisation settings?"** choose
   **`No`** (or "Edit Settings" then leave everything unset). cloud-init handles
   the user, hostname, and Wi-Fi - leaving Imager's customisation off avoids it
   fighting with `user-data`.

   > Exception: if you'd rather configure Wi-Fi in Imager than via
   > `network-config`, you may set just the Wi-Fi fields here and skip Step 3's
   > `network-config`. Don't also set a username/password in Imager - let
   > `user-data` own that.
6. Click **Write** and wait for it to flash + verify, then remove and re-insert
   the card so the host re-mounts the small FAT32 boot partition (labeled
   **`bootfs`**).

### Step 2 - Set the localuser password hash

Generate a SHA-512 hash and paste it into `user-data` in place of
`__LOCALUSER_PASSWORD_HASH__`:

```sh
openssl passwd -6
# enter your chosen password twice; copy the whole $6$... string into user-data
```

### Step 3 - (Optional) Wi-Fi

If you didn't set Wi-Fi in Imager and aren't using Ethernet:

```sh
cp network-config.example network-config   # then edit network-config
```

Fill in your SSID and PSK. This file is git-ignored (it holds your PSK).

> **Wi-Fi country (important):** Raspberry Pi OS Bookworm keeps the Wi-Fi radio
> rfkill-blocked until a regulatory country is set, so without it `wlan0` stays
> DOWN and first boot has no internet. `user-data` sets this early (in `bootcmd`)
> and it defaults to **`US`** - if you're elsewhere, change every `US` in
> `user-data`'s `bootcmd` block to your ISO 3166-1 code (e.g. `GB`, `DE`, `MX`).
>
> **Most reliable first boot:** use **Ethernet** for the initial provisioning
> boot - it needs no country and never hits the rfkill block, so cloud-init can
> always reach the internet. Switch to Wi-Fi afterwards.

### Step 4 - Copy the cloud-init files onto the boot partition

Copy `user-data`, `meta-data`, and (if you made one) `network-config` to the
root of the `bootfs` partition. The mount path differs by host OS:

- **macOS:** the partition mounts at `/Volumes/bootfs`

  ```sh
  cp user-data meta-data /Volumes/bootfs/
  cp network-config      /Volumes/bootfs/   # only if you created it in Step 3
  ```

- **Linux:** usually `/media/$USER/bootfs` (check with `lsblk` / `findmnt`)

  ```sh
  cp user-data meta-data /media/$USER/bootfs/
  cp network-config      /media/$USER/bootfs/   # only if you created it
  ```

- **Windows:** the partition shows up as a drive letter (e.g. `E:`). Copy
  `user-data`, `meta-data`, and optionally `network-config` into the root of
  that drive (File Explorer or `copy user-data E:\`). Make sure they have **no
  file extension** (not `user-data.txt`).

Then eject/unmount the card safely.

### Step 5 - Boot the Pi

Connect the monitor and keyboard **before** powering on (HDMI is detected at
boot under the KMS driver), then power up. First boot runs the full install -
give it several minutes and expect **one automatic reboot**. Progress is logged
on the card to `/var/log/cloud-init-output.log`.

### Step 6 - Log in

At the prompt, sign in as **`localuser`** with the password you hashed in Step 2. You
land in the fullscreen cage+foot console. Run `nvim` for LazyVim.

## One manual step left

The Copilot CLI device login can't be automated. Once, run:

```sh
copilot
```

and complete the device-code flow in a browser. After that it's authenticated.

The token is saved **encrypted** in the gnome-keyring login keyring, which is
unlocked by your login password (via PAM) and exposed to the CLI's keytar backend
by the tty1 launcher (it starts the `secrets` component on the per-user D-Bus bus).
You should *not* see `system vault not available`. If you do, see Troubleshooting.

## Notes

- `user-data` is intentionally **pure ASCII** and every network install is
  guarded (`|| true`) so a transient download failure never wedges first boot —
  the affected step simply retries on next `nvim`/login.
- **Keyboard** defaults to **US** (`/etc/default/keyboard`). Change `XKBLAYOUT`
  in `user-data` if you need another layout.
- To re-run provisioning on an already-booted card, clean cloud-init state:
  `sudo cloud-init clean --logs` then reboot. This is also the fix if first boot
  had no internet (e.g. Wi-Fi country issue): get online, then clean + reboot to
  let cloud-init reinstall everything.

## Troubleshooting

- **`wlan0` DOWN / no internet, and the console didn't launch (you got a plain
  shell).** These go together: no network -> cloud-init couldn't install
  `cage`/`foot`/etc. -> the tty1 launcher falls through to a shell. Almost always
  the **Wi-Fi country** (rfkill) issue. On the Pi:

  ```sh
  sudo raspi-config nonint do_wifi_country US   # your country code
  sudo rfkill unblock wifi
  sudo systemctl restart NetworkManager
  ip -brief addr                                 # wlan0 should now be UP
  # if it didn't auto-connect:
  sudo nmcli device wifi connect "YOUR_SSID" password "YOUR_PASSWORD"
  ```

  Then finish the build: `sudo cloud-init clean --logs && sudo reboot`.
- **Wrong keys (GB layout).** `sudo raspi-config nonint do_configure_keyboard us`
  (persists in `/etc/default/keyboard`).
- **Copilot CLI: `system vault not available` (or `copilot` hangs on first
  `/login`).** The keyring isn't being unlocked at login. The usual cause is a
  missing PAM module: on Debian, `pam_gnome_keyring.so` ships in the **separate
  `libpam-gnome-keyring`** package (the `gnome-keyring` package is only the daemon),
  so the PAM lines are silently ignored and the login keyring stays locked. A locked
  collection makes `keytar` block on a non-existent unlock prompt -> the hang. Check
  and repair on the Pi:

  ```sh
  ls /usr/lib/*/security/pam_gnome_keyring.so 2>/dev/null || echo MODULE MISSING
  sudo apt install -y libpam-gnome-keyring libsecret-tools
  grep -q pam_gnome_keyring /etc/pam.d/login || sudo bash -c \
    'printf "\nauth     optional  pam_gnome_keyring.so\nsession  optional  pam_gnome_keyring.so auto_start\n" >> /etc/pam.d/login'
  sudo reboot   # PAM only unlocks at login; reboot also clears stray keyring daemons
  ```

  After reboot, from the console: `secret-tool store --label=t pd probe` (type a
  value) should succeed, and `copilot` -> `/login` should store the token encrypted
  with no warning.
- **USB gadget: host sees no `usb0` / `ssh pocketdev.local` fails.** Gadget mode
  only activates after the **first reboot** (the firmware reads `config.txt`/
  `cmdline.txt` at boot). Confirm and repair on the Pi:

  ```sh
  ls /sys/class/udc                 # must be non-empty (e.g. fe980000.usb)
  grep dwc2 /boot/firmware/config.txt /boot/firmware/cmdline.txt
  sudo systemctl status pocketdev-usb-gadget.service
  ip -brief addr show usb0          # interface should exist
  ```

  If `/sys/class/udc` is empty, the `dtoverlay=dwc2,dr_mode=peripheral` (config.txt)
  + `modules-load=dwc2` (cmdline.txt) edits didn't apply - re-check both files and
  reboot. Also use a **data** USB-C cable (not charge-only), and on macOS enable
  **Internet Sharing** to the RNDIS/`pocketdev` device for Copilot CLI WAN. Full
  host-side walkthrough: [`docs/usb-gadget.md`](../docs/usb-gadget.md).
