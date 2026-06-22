# Quickstart: plug in and go

The fastest path from a blank SD card to `nvim` + Copilot CLI on a Raspberry Pi 4 / 5.
Pick the front door you want; both run the **same** Pocket Dev build and can coexist on
the same card.

> First, build the card once with cloud-init (below). After that, "using" the Pi is just
> plugging it in.

## 0. Build the card (once, ~10 min hands-on + one boot)

Flash Raspberry Pi OS Lite (64-bit) and drop the cloud-init files on the boot partition.
Full step-by-step (password hash, optional Wi-Fi, copying files): **[`cloud-init/README.md`](../cloud-init/README.md)**.

On first boot the Pi provisions itself end-to-end. **Wait for it to finish** before judging
anything:

```sh
cloud-init status --wait      # prints: status: done
```

Logging in before this completes gives a half-provisioned console (default font, no theme).

## 1a. Front door A - local console (monitor + keyboard)

1. Connect an HDMI monitor **before** powering on, plus a USB keyboard.
2. Power on; wait for cloud-init (see above) on first boot.
3. Log in as `localuser`. You drop straight into a fullscreen `cage` + `foot` terminal
   running `nvim` (LazyVim, Tokyo Night, Nerd Font glyphs) - no desktop.

Deep runbook (HDMI notes, font, foot config): **[`docs/quickstart-local-console.md`](quickstart-local-console.md)**.

## 1b. Front door B - USB gadget (one cable to a Mac)

1. Connect the Pi's **USB-C** port to the Mac with a *data* cable.
2. (One-time on the Mac) install the matching font + set Terminal.app to use it, so glyphs
   render client-side - see the client-setup section in the gadget guide.
3. SSH over the cable:

   ```sh
   ssh localuser@pocketdev.local
   ```

4. For internet (Copilot CLI needs it), turn on **Internet Sharing** to the gadget
   interface.

Deep runbook (Internet Sharing, power caveat, fonts, troubleshooting):
**[`docs/usb-gadget.md`](usb-gadget.md)**.

> Gadget mode activates after the **first reboot** (firmware reads `config.txt`/`cmdline.txt`
> only at boot), so the very first boot is console-only.

## 2. The one manual step: authenticate Copilot CLI

The device-code login can't be automated. Once, run:

```sh
copilot
```

and finish the device-code flow in a browser. The token is then stored **encrypted** in
the gnome-keyring vault (unlocked by your login password via PAM). After that it just
works.

## 3. Drive LazyVim

Toggle the Copilot CLI terminal float and learn the shipped keymaps in
**[`docs/lazyvim-guide.md`](lazyvim-guide.md)**. The short version: `<C-/>` (or `Ctrl+t` /
`<Space>tt` from a terminal that can't send `<C-/>`) opens the large terminal float; run
`copilot` inside it.

## Reset / start over

Re-flash the card (or `sudo cloud-init clean --logs` + reboot) to return to a pristine
build - nothing precious lives on the Pi. See the **Factory reset** section in
[`cloud-init/README.md`](../cloud-init/README.md).
