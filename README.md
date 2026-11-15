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

On this AX88179B silicon that bit reads 0 — **unless some driver has performed an initialisation that
mainline never does** (see the next section) — so the driver calls `netif_carrier_off()` forever.
Meanwhile the PHY is perfectly healthy — the MII `BMSR` link bit reads correctly, which is exactly why
`ethtool` shows a negotiated gigabit link while the kernel insists there's no carrier.

ASIX's driver already ships a working alternative: an **MII-BMSR polling** path. It just isn't reachable
on this hardware, because it's gated behind "only poll if there is no interrupt endpoint" — and this
adapter *has* an interrupt endpoint. It's simply feeding a bogus link bit.

**The fix: compile in the polling path and force it on.** When forced, the interrupt URB is suppressed
(`ax_usb_submit_intr_urb()` early-returns) and carrier is driven entirely by the MII status that works.

This is an [open, unfixed mainline bug](https://lkml.iu.edu/2511.1/04339.html) — still unfixed as of
**6.18.40** (checked against the 6.18.38/39/40 changelogs and the mainline history of
`drivers/net/usb/ax88179_178a.c`).

## Why this bug looks intermittent — read this before concluding anything

The most confusing thing about this bug: **it sometimes appears to fix itself.**

If another driver initialises the chip first — the Windows driver, or ASIX's own — and you then
**warm-reboot** into Linux, the *unmodified mainline driver* reports carrier correctly and everything
looks fine. A warm reboot doesn't drop VBUS, so the adapter never loses power and keeps the register
state the previous driver programmed. Mainline inherits an already-configured chip.

The illusion breaks the moment the adapter actually loses power — unplug it, or cold-boot far enough to
cut VBUS — and `NO-CARRIER` is back.

This is an effective trap. During development it produced a confidently wrong conclusion — that a kernel
upgrade had fixed the driver upstream — which survived until the adapter was physically unplugged.

**Deterministic reproducer:**

1. Unplug the adapter for ~10 s, then replug. *This step is the whole point: it must lose power.*
2. Bind mainline `ax88179_178a` — it negotiates 1000/Full and reports `Link detected: no`, indefinitely.
3. Let the ASIX vendor driver (or Windows) initialise the chip, then warm-reboot so VBUS is preserved.
4. The **same unmodified mainline driver** now reports carrier correctly.

> **If you are testing any fix for this bug, power-cycle the adapter first.** Otherwise you may be
> measuring leftover chip state rather than your change.

It also suggests the real defect is narrower than "mainline misreads a bit": more likely **mainline
omits an initialisation step that makes the link bit valid**, and both the Windows driver and the ASIX
driver perform it. That distinction matters for anyone attempting a proper upstream fix.

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

> ⚠ **`blacklist` alone is not enough.** On the test system mainline still loaded and bound the device
> after a reboot *and* after a replug, with the blacklist in place and parsed (`modprobe -c` showed it).
> `blacklist` only suppresses *alias-based* autoload. The shipped file therefore also contains:
>
> ```
> install ax88179_178a /bin/false
> ```
>
> which blocks explicit and dependency loads too. Verify with `modprobe ax88179_178a` — it should fail
> with `Error running install command '/bin/false'`. If mainline is already loaded, `rmmod` it once;
> the block prevents loading, not unloading.
>
> If your init system builds an initramfs, rebuild it after copying this file (e.g.
> `dracut --force`) so the config is present on the next boot.

> ⚠ Trade-off: with mainline fully blocked, a failed DKMS rebuild leaves you with *no* wired driver
> rather than a broken one. Check `dkms status` first when debugging, and keep a fallback kernel that
> still has the module built.

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

### Notes from a Void Linux install (2026-11-14)

- **`/etc/udev/rules.d` may not exist.** Void ships eudev, whose rules live in
  `/usr/lib/udev/rules.d`; the `/etc` override directory is only created when something puts a rule
  there. `install -m644 50-ax88179b.rules /etc/udev/rules.d/` then fails with
  `Not a directory`. `sudo mkdir -p /etc/udev/rules.d` first — `/etc` is the right home, since
  `/usr/lib/udev/rules.d` is package-managed.
- **The DKMS rebuild hook is not `dkms autoinstall`.** Void's `/etc/kernel.d/post-install/10-dkms`
  walks `/var/lib/dkms/*` itself and builds + installs every registered module for the new kernel, so
  `AUTOINSTALL="yes"` in `/etc/dkms/framework.conf` has no effect there (nothing calls
  `dkms autoinstall`). What the hook *does* check is
  `/lib/modules/$VERSION/build/include` — i.e. a `linux<series>-headers` package matching the new
  kernel. Without it the hook prints `cannot build DKMS modules! missing kernel headers package` and
  exits 0, skipping the build silently. With mainline blacklisted, that leaves **no** wired driver, so
  keep the headers package in the upgrade set and check `dkms status` after every kernel bump.
- These paths are outside chezmoi on that machine: a reinstall loses
  `/etc/udev/rules.d/50-ax88179b.rules` and `/etc/modprobe.d/ax_usb_nic_blacklist.conf`, and they have
  to be re-copied from this repo.

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

- Void Linux, kernels **6.18.37** and **6.18.40**, `x86_64` — vendor source claims support "up to kernel
  6.14.x"; no kernel-API drift was hit building against 6.18. DKMS rebuilt cleanly across the
  6.18.37 → 6.18.40 upgrade with no intervention.
- Adapter `0b95:1790`, `iProduct` = `AX88179B`, USB 3 (5 Gbps).
- **Second machine, 2026-11-14** — Void Linux kernel **6.18.63_1** (past the 6.18.40 checked above):
  the vendor source built unchanged through DKMS, the device came up as `version 3.20` / `speed 5000`
  on a USB 3 port with config forced to 1, and `ax_usb_nic` 4.1.0 bound. Against the same router over
  wifi: **2.9–3.3 ms** RTT (wifi 5.7 ms), upload **87 vs 80 Mbit/s**, download equal at ~90 Mbit/s
  (plan-limited), `carrier_changes` steady at 2 across ~300 MB of transfers. Before the fix the same
  dongle sat on `cdc_ncm` / config 2 and managed **20 Mbit/s at 14 ms** — which is what an unfixed
  install looks like in latency terms, not just as `NO-CARRIER`.
- Verified across a reboot and a physical replug: `ax_usb_nic` claims the device and brings up a
  **cold-reset** chip (carrier, gigabit, DHCP). That last part matters — it shows the vendor driver does
  the initialisation itself rather than inheriting state, unlike mainline.

Note that **the brand on the box tells you nothing** — adapters sold under different brands turned out
to carry identical ASIX silicon. Check `lsusb` for `0b95`, not the label.

## Upstreaming

Not yet submitted, and still worth doing — the bug is confirmed unfixed upstream as of 6.18.40 and the
LKML report is still open.

The mainline-appropriate fix is narrower than this vendor-driver workaround. Given the intermittency
finding above, the most promising line is **adding the missing initialisation to `ax88179_178a`** rather
than correcting a bit position or switching carrier detection to MII BMSR — the polling route only
sidesteps the interrupt path instead of fixing it.

Ground truth would come from a `usbmon` capture comparing **register writes during probe**, Windows (or
the ASIX driver) versus mainline — not just the interrupt-endpoint status bytes, since those are a
symptom of the chip being unconfigured rather than the cause. PRs welcome.

## AI assistance disclosure

This fix was developed with heavy use of an LLM (Claude, by Anthropic), and it seems only fair to say
so plainly — particularly if you are reviewing the diff or weighing it for upstream.

**What the model did:** read the vendor source and traced the `NO-CARRIER` symptom to the
interrupt-endpoint link bit; noticed that the vendor's own BMSR polling path already existed but was
unreachable on hardware that *has* an interrupt endpoint; wrote the patch, `dkms.conf`, the udev rule,
and this README.

**What a human did:** all of the testing. Every claim on this page — the negotiated gigabit link, the
stable `carrier_changes`, DHCP, the ping results, the switch-flap conclusion — comes from observed
behaviour on real hardware, not from the model's say-so.

**The model was wrong twice**, and both errors are documented above rather than quietly dropped, because
they're the two traps most likely to catch someone else:

1. It blamed the driver for a link flap that turned out to be a managed switch shutting the port.
2. It concluded from a post-reboot observation that a kernel upgrade had fixed the bug upstream. It had
   not — the chip was carrying leftover configuration across a warm reboot. The correction came from the
   hardware owner recalling that plugging into Windows and rebooting had produced the same illusion for
   years, which is what prompted the physical replug that disproved it.

Point 2 is worth dwelling on if you're using an LLM on hardware problems: the model reasoned correctly
from the evidence it had, and was still wrong, because the evidence was contaminated by state it
couldn't observe.

**What this means for you:** judge the diff on its merits. It is two hunks; it is short enough to read
in full, and you should. It has been verified on exactly one adapter and one kernel (see "Tested on"),
so treat it as a working fix for a specific problem rather than a broadly validated patch.

## License

GPL-2.0, inherited from ASIX's GPL driver release. Copyright © 2022 ASIX Electronics Corporation for
the original source; the patch is offered under the same terms. Vendor source: ASIX
[AX88179B product page](https://www.asix.com.tw/en/product/USBEthernet/Super-Speed_USB_Ethernet/AX88179B).

This is an unofficial patch. It is not affiliated with or endorsed by ASIX.
