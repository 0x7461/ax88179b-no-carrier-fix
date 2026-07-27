# AX88179B — fix for permanent `NO-CARRIER` on Linux

A two-line fix for ASIX **AX88179B** USB-Ethernet adapters (USB ID `0b95:1790`) that link fine on
Windows but never get a carrier on Linux.

**Symptom.** The interface negotiates correctly — `ethtool` reports `1000Mb/s`, `Full`, and sees the
link partner — but the kernel keeps the interface `NO-CARRIER` and no traffic passes. `dmesg` repeats:

```
ax88179_178a 2-3:1.0 eth0: ax88179 - Link status is: 0
```

If that line brought you here, this repo is the fix.

## Root cause

The mainline `ax88179_178a` driver (and ASIX's own driver, at its default settings) derives carrier
from the **USB interrupt endpoint**:

```c
axdev->link = event->link & AX_INT_PPLS_LINK;
```

On this AX88179B silicon that bit is always 0, so the driver calls `netif_carrier_off()` forever.
Meanwhile the PHY is perfectly healthy — the MII `BMSR` link bit reads correctly, which is exactly why
`ethtool` shows a negotiated gigabit link while the kernel insists there's no carrier.

ASIX's driver already ships a working alternative: an **MII-BMSR polling** path. It just isn't reachable
on this hardware, because it's gated behind "only poll if there is no interrupt endpoint" — and this
adapter *has* an interrupt endpoint. It's simply feeding a bogus link bit.

**The fix: compile in the polling path and force it on.** When forced, the interrupt URB is suppressed
(`ax_usb_submit_intr_urb()` early-returns) and carrier is driven entirely by the MII status that works.

This is an [open, unfixed mainline bug](https://lkml.iu.edu/2511.1/04339.html).

## The patch

Two hunks against ASIX's GPL v4.1.0 release — see `git log` (the base commit is the pristine vendor
tarball, so `git diff` against it shows exactly this and nothing else):

- **`ax_config.h`** — `ENABLE_INT_POLLING` `0` → `1`, compiling in the BMSR polling path.
- **`ax_main.c`** — replace the auto-gate with an unconditional `axdev->int_polling_enable = true;`.

Nothing else is modified. All other vendor defaults are untouched, including USB Link Power Management.

## Install

Requires kernel headers, `gcc`, `make`, and `dkms`.

```sh
git clone https://github.com/0x7461/ax88179b-no-carrier-fix
sudo cp -r ax88179b-no-carrier-fix /usr/src/asix-ax88179-4.1.0-patched1

sudo dkms add     -m asix-ax88179 -v 4.1.0-patched1
sudo dkms build   -m asix-ax88179 -v 4.1.0-patched1
sudo dkms install -m asix-ax88179 -v 4.1.0-patched1
```

DKMS (`AUTOINSTALL=yes`) rebuilds the module on every kernel upgrade, which you want — a stale module
means no wired network after the next kernel bump.

**Keep the mainline driver out of the way.** It binds first otherwise and reinstates the bug:

```sh
sudo cp ax_usb_nic_blacklist.conf /etc/modprobe.d/
```

> ⚠ Trade-off: if a future DKMS rebuild ever fails you get *no* wired driver rather than a broken one.
> Check `dkms status` first when debugging.

**Force the vendor USB configuration.** The AX88179B exposes three USB configurations and boots into
CDC-NCM (config 2), where `cdc_ncm` claims it as a dumb pipe with no PHY control — `ethtool` shows no
link modes at all. Config 1 is the vendor-specific one a real Ethernet driver can bind to:

```sh
sudo cp 50-ax88179b.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
```

That rule also disables USB autosuspend for this device, which prevented a re-enumeration loop
(`register`/`unregister 'cdc_ncm'` every ~4 s) on the hardware this was developed against.

> **Rule ordering matters.** `bConfigurationValue` is set *before* `power/control`. Setting the
> configuration re-initialises the device and wipes any preceding `power/control` write, so the
> intuitive order silently loses the autosuspend setting.

Then replug the adapter, or reboot.

## Verify

```sh
ip -br link                                   # interface should NOT show NO-CARRIER
cat /sys/class/net/<iface>/carrier            # 1
ethtool <iface> | grep 'Link detected'        # yes
cat /sys/class/net/<iface>/carrier_changes    # should stay put, not climb
```

The interface is briefly named `eth0` between driver bind and udev's rename, so find it by vendor ID
rather than a guessed name:

```sh
for n in /sys/class/net/*; do
  [ "$(cat $n/device/../idVendor 2>/dev/null)" = 0b95 ] && basename $n
done
```

## If the link comes up but flaps

**Suspect your switch before you suspect this driver.** During development the link came up at gigabit
and then dropped ~550 ms later, with a ~184 s gap before retrying — which looked exactly like a driver
bug. It wasn't. On a consumer router the *same module binary* held carrier indefinitely with zero
`carrier_changes`.

That signature — sub-second link loss followed by a multi-minute dead period with the adapter's RJ45
LED dark — is a **managed switch shutting the port**: flap-protection, `errdisable` with a recovery
timer, 802.1X, or MAC-based port security reacting to an unfamiliar USB NIC. No driver patch fixes
that; the port configuration has to change.

A driver-internal timer would misbehave identically on every network. If yours only misbehaves on one
network, it's the network.

## Tested on

- Void Linux, kernel **6.18.37**, `x86_64` — vendor source claims support "up to kernel 6.14.x"; no
  kernel-API drift was hit building against 6.18.
- Adapter `0b95:1790`, `iProduct` = `AX88179B`, USB 3 (5 Gbps).

Note that **the brand on the box tells you nothing** — adapters sold under different brands turned out
to carry identical ASIX silicon. Check `lsusb` for `0b95`, not the label.

## Upstreaming

Not yet submitted. The mainline-appropriate fix is narrower than this vendor-driver workaround: correct
the 179B interrupt-status bit layout in `ax88179_178a.c`, or derive carrier from MII BMSR there too.
Ground truth for the interrupt format would come from a `usbmon` capture of the interrupt endpoint on
Windows (link up) versus Linux (link down), diffing the status bytes. PRs welcome.

## License

GPL-2.0, inherited from ASIX's GPL driver release. Copyright © 2022 ASIX Electronics Corporation for
the original source; the patch is offered under the same terms. Vendor source: ASIX
[AX88179B product page](https://www.asix.com.tw/en/product/USBEthernet/Super-Speed_USB_Ethernet/AX88179B).

This is an unofficial patch. It is not affiliated with or endorsed by ASIX.
