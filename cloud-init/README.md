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

- A non-autologin user `localuser` (you log in with username + password on the console).
- Toolchain: git, build-essential, ripgrep, fd, fzf, git-delta, gh, pipx, python3-venv.
- **Neovim** (latest stable, arm64 tarball) + **LazyVim** starter (git history stripped).
- **lazygit** (arm64, via the release-redirect version trick — the api.github.com
  method is rate-limited on shared IPs).
- **Node 22 via nvm** + `@github/copilot` (the Copilot CLI).
- **cage + foot** no-desktop console, JetBrainsMono Nerd Font, noto-color-emoji,
  and a Tokyo Night `foot.ini` so the shell/TUI match LazyVim.
- A `tty1` launcher (`/etc/profile.d/10-pocketdev-console.sh`) that drops you into
  the fullscreen console after login — quitting nvim returns you to a shell.

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

## Notes

- `user-data` is intentionally **pure ASCII** and every network install is
  guarded (`|| true`) so a transient download failure never wedges first boot —
  the affected step simply retries on next `nvim`/login.
- To re-run provisioning on an already-booted card, clean cloud-init state:
  `sudo cloud-init clean --logs` then reboot (rarely needed).
