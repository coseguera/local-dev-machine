# USB gadget mode: SSH (and internet) over one USB-C cable

Pocket Dev configures the Raspberry Pi 4 as a **USB CDC-ECM Ethernet gadget** on its
**USB-C** port. Plug that port into a host (a Mac today, an iPad later) with a *data*
USB-C cable and the host sees a new network interface — you `ssh` to the Pi over the
cable, with **no monitor, no Wi-Fi setup, and nothing installed on the host** beyond
toggling macOS Internet Sharing (reversible).

This is the SSH front door from [ADR-0001](decisions/0001-access-model-usb-gadget.md).
It is **co-equal** with the local-console path ([ADR-0003](decisions/0003-local-console-first.md))
— neither disables the other; `sshd` over `usb0` is unaffected by what renders on HDMI.

## How it's wired (Pi side, all automatic)

`cloud-init/user-data` sets this up on first boot; it **activates after the first
reboot** (the firmware reads `config.txt`/`cmdline.txt` only at boot):

- `config.txt`: `dtoverlay=dwc2,dr_mode=peripheral` — USB-C controller in gadget mode.
- `cmdline.txt`: `modules-load=dwc2` — load the controller driver early.
- `pocketdev-usb-gadget.service` builds a **libcomposite CDC-ECM** gadget with stable,
  locally-administered MACs, so the host keeps the **same** `usb0` across reboots.
- The Pi's `usb0` is a **DHCP client** (NetworkManager). It does not run a DHCP server
  or NAT — the *host* provides the address and (optionally) the internet.
- `avahi-daemon` advertises `pocketdev.local` over the cable, so you never need to know
  the IP.

> Only the USB-C port does gadget mode; the USB-A ports are host-only.

## macOS host: just SSH (no internet sharing)

1. Connect the Mac to the Pi's **USB-C** port with a data-capable cable.
2. A new interface appears: **System Settings -> Network -> "Pocket Dev USB Ethernet"**
   (or `RNDIS/Ethernet Gadget`). Leave it on its default (it will self-assign a
   link-local address; that's fine).
3. SSH over the cable:

   ```sh
   ssh localuser@pocketdev.local
   ```

   This works even with **no IP configuration** because mDNS resolves `pocketdev.local`
   to the Pi's IPv6 link-local address on `usb0`.

If `pocketdev.local` doesn't resolve, find the link-local interface and use it directly:

```sh
# list interfaces; the gadget is usually the most recently added en*
ifconfig | grep -B3 -i 'gadget\|ecm'
ssh localuser@pocketdev.local%en7      # replace en7 with the gadget interface
```

## macOS host: give the Pi internet (Copilot CLI WAN)

The Copilot CLI needs internet. The preferred path (plan §3c, P1) is **host Internet
Sharing**: the Pi piggybacks on *whatever the Mac is connected to* — home, a cafe, a
phone hotspot — **without the Pi ever joining those networks**. Captive portals are
handled in the Mac's browser; the Pi just inherits connectivity.

1. **System Settings -> General -> Sharing -> Internet Sharing**.
2. *Share your connection from:* your active uplink (Wi-Fi, or iPhone USB, etc.).
3. *To computers using:* check **Pocket Dev USB Ethernet** (the gadget interface).
4. Toggle **Internet Sharing** on (confirm the prompt).

The Mac now NATs the cable subnet out its current uplink and runs a small DHCP server,
so the Pi's `usb0` gets an address, default route, and DNS automatically. Verify on the
Pi:

```sh
ip route                 # should show a default via the usb0 gateway
ping -c1 github.com
copilot                  # authenticate once via the device-code flow
```

When you move the Mac to a different network, **leave Internet Sharing on** — the Pi
keeps working with no reconfiguration because it never joined the venue network itself.

### Fallback: Pi onboard Wi-Fi for WAN

If you can't (or don't want to) use Internet Sharing, the Pi's own Wi-Fi is the fallback
WAN — configure it via `cloud-init/network-config` (gitignored; see
`cloud-init/network-config.example`). SSH-over-cable still works regardless; only the
*internet source* changes. This is also the path an **iPad host** needs, since iPadOS
can't share its connection to a USB gadget.

## Power note (Pi 4 specific)

The Pi 4 normally takes power **through** its USB-C port, but in gadget mode that same
port is the data link to the host. A laptop/host USB-C port often can't supply the Pi
4's full draw under load, so you may see under-voltage warnings or instability. If so,
power the Pi from the **GPIO 5V pins** (a 5V/3A supply on pins 4 & 6) while the USB-C
cable carries data to the host. The lighter-weight Pi Zero 2 W avoids this (separate
power vs. data micro-USB ports) but was rejected as too slow for interactive Copilot CLI
([ADR-0002](decisions/0002-host-choice.md)).

## Troubleshooting

| Symptom | Check / fix |
|---|---|
| Host sees no new interface | Use a **data** USB-C cable (not charge-only). On the Pi: `ls /sys/class/udc` must be non-empty; if empty, the `config.txt`/`cmdline.txt` edits didn't apply — re-check and reboot. |
| `usb0` missing on the Pi | `systemctl status pocketdev-usb-gadget.service`; `journalctl -u pocketdev-usb-gadget`. Re-run `sudo /usr/local/sbin/pocketdev-usb-gadget`. |
| `ssh pocketdev.local` won't resolve | `avahi-daemon` running? Try `ssh localuser@pocketdev.local%<host-iface>`, or read the Pi's `usb0` address from `ip -brief addr` over the console. |
| No internet on the Pi | Internet Sharing must target the **gadget** interface; `ip route` on the Pi should show a default route. macOS prefers **CDC-ECM**; RNDIS works too but is flakier. |
| Under-voltage / random reboots | Power the Pi from GPIO 5V (see Power note). |
