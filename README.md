# NUT Server + Client Setup Guide

**Setup this guide is written for:**

| Role | Machine | Package | `nut.conf` mode |
|---|---|---|---|
| NUT server (primary) | Ubuntu 24.04 VM (`ubu`), UPS passed through over USB | `nut` (NUT 2.8.1) | `netserver` |
| NUT client (secondary) | Proxmox VE host | `nut-client` | `netclient` |

**UPS:** APC Back-UPS NS 1500M2, USB ID `051d:0002`, serial `0B2548L00791`, NUT name `apc`

Replace anything in `<angle brackets>` with your own values. Each step says *why* as well as *what*, so you can adapt it to other setups.

---

## How the pieces fit

```
APC UPS ──USB──▶ Ubuntu VM (NUT server)
                  ├─ driver (usbhid-ups)  reads the UPS over USB
                  ├─ upsd                 serves UPS data on TCP 3493
                  ├─ upsmon (primary)     decides when to shut down
                  └─ upssched             timers ("still on battery after N sec → act")
                         │  TCP 3493
                         ▼
                 Proxmox host (NUT client)
                  └─ upsmon (secondary)   shuts down when the primary says so
```

- **Driver → upsd → upsmon** is the chain. If `upsc` says *"Driver not connected"*, upsd is running but the driver is not.
- The **primary** waits for **secondaries** to shut down before it shuts down itself.
- **Design choice:** the timers live on the **server only**. When a timer fires, the server raises FSD (forced shutdown) and every client follows. Clients don't need their own timers, so there's only one place to configure them.

---

# Part 1 — Prep: pass the UPS through to the VM (Proxmox host)

Skip this part if the NUT server is bare metal.

1. In the Proxmox GUI, **shut the VM down completely**.
2. Go to VM → Hardware → Add → USB Device → **Use USB Vendor/Device ID** → pick `051d:0002`.
3. **Leave "Use USB3" unchecked.** This UPS is a full-speed (12 Mbit) USB 1.1 device. On a virtual USB 3 (xhci) controller it fails to set up, with `can't set config #1, error -32` in `dmesg`.
4. Start the VM.

The same thing from the Proxmox shell:

```bash
qm shutdown <vmid> && qm wait <vmid>
qm set <vmid> -usb0 host=051d:0002        # no usb3=1
qm start <vmid>
```

**Why vendor/device ID instead of port:** the passthrough follows the UPS even if you move it to a different USB port.

**Why a cold start:** the VM's virtual USB controller is only rebuilt when the VM starts, so a reboot from inside the VM doesn't apply the change.

---

# Part 2 — NUT server (Ubuntu VM)

## 2.1 Confirm the VM sees the UPS

```bash
lsusb | grep -i 051d
lsusb -t
sudo dmesg | grep -iE "051d|set config"
```

What you should see:
- `lsusb` lists `ID 051d:0002 American Power Conversion`.
- `lsusb -t` shows the UPS with `If 0, Class=Human Interface Device, Driver=usbhid`. If there's no `If 0` or `Driver=`, the device never finished setting up. Go back to Part 1.
- `dmesg` has **no** `can't set config` line.

## 2.2 Install

```bash
sudo apt update
sudo apt install nut
```

`nut` pulls in both `nut-server` and `nut-client`. The server machine needs both, because it also runs its own upsmon.

Optional — make `nut-scanner` work. NUT 2.8.1 on Ubuntu looks for the plain `libusb-1.0.so` file, which only comes with the -dev package:

```bash
sudo apt install libusb-1.0-0-dev
sudo nut-scanner -U        # prints a ready-made ups.conf block, including the serial
```

## 2.3 Disable upower

```bash
sudo systemctl disable --now upower
sudo systemctl mask upower
```

**Why:** upower is a desktop battery service that grabs HID UPS devices. If it holds the UPS, the NUT driver can't claim it.

## 2.4 udev rule (lets the `nut` user access the UPS)

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="051d", ATTR{idProduct}=="0002", MODE="0664", GROUP="nut"' \
  | sudo tee /etc/udev/rules.d/99-nut-ups.rules
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=usb
ls -l /dev/bus/usb/<bus>/<device>     # from lsusb; group should now be: nut
```

**Why:** the driver gives up root privileges and runs as the `nut` user, so a bug in it can't take over the system. That means the USB device file must belong to the `nut` group. Without this, you get `insufficient permissions on everything`.

## 2.5 `/etc/nut/nut.conf`

```ini
MODE=netserver
```

`netserver` tells the service scripts to start the driver, upsd **and** upsmon, and to accept clients over the network.

## 2.6 `/etc/nut/ups.conf`

```ini
maxretry = 3

[apc]
    driver = usbhid-ups
    port = auto
    desc = "APC Back-UPS NS 1500M2"
    vendorid = 051d
    productid = 0002
    serial = 0B2548L00791
```

- `[apc]` is the UPS name used everywhere else (`apc@localhost`, `apc@<server-ip>`).
- `vendorid`, `productid` and `serial` pin the driver to this exact UPS. That matters if you ever add a second APC unit, because the IDs will be identical.
- `maxretry = 3` makes the driver retry at startup if the USB device isn't ready yet.

## 2.7 `/etc/nut/upsd.conf`

```ini
LISTEN 127.0.0.1 3493
LISTEN <server-ip> 3493
```

upsd only listens on localhost by default. Adding the VM's LAN IP lets clients connect. This is tighter than `LISTEN 0.0.0.0`, which listens on every interface.

**Firewall:** allow TCP 3493 only from your NUT clients (Proxmox firewall or UFW). For example:

```bash
sudo ufw allow from <proxmox-host-ip> to any port 3493 proto tcp
```

## 2.8 `/etc/nut/upsd.users`

```ini
[upsmon_local]
    password = <strong-password-1>
    upsmon primary

[upsmon_remote]
    password = <strong-password-2>
    upsmon secondary

[admin]
    password = <strong-password-3>
    actions = SET
    instcmds = ALL
```

| User | Used by | Why |
|---|---|---|
| `upsmon_local` | the server's own upsmon | `primary` = the machine wired to the UPS. It controls the shutdown order. |
| `upsmon_remote` | every client | `secondary` = follows the primary's lead |
| `admin` | you, with `upscmd` / `upsrw` | runs battery tests and changes UPS settings |

`primary`/`secondary` replaced `master`/`slave` in NUT 2.8. The old words still work.

```bash
sudo chmod 640 /etc/nut/upsd.users
sudo chown root:nut /etc/nut/upsd.users
```

This file holds passwords, so only root and the nut group should be able to read it.

## 2.9 `/etc/nut/upsmon.conf` (server)

```ini
MONITOR apc@localhost 1 upsmon_local <strong-password-1> primary

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
NOTIFYCMD /usr/sbin/upssched
POLLFREQ 5
POLLFREQALERT 2
HOSTSYNC 15
DEADTIME 15
FINALDELAY 5
POWERDOWNFLAG /etc/killpower

NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL+EXEC
NOTIFYFLAG FSD      SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK   SYSLOG+WALL+EXEC
NOTIFYFLAG COMMBAD  SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT SYSLOG+WALL
NOTIFYFLAG NOCOMM   SYSLOG+WALL+EXEC
NOTIFYFLAG CAL      SYSLOG+WALL+EXEC
NOTIFYFLAG NOTCAL   SYSLOG+WALL+EXEC
```

What the key settings do:
- **`MONITOR … 1 …`**: `1` is the number of power supplies this UPS feeds on this machine.
- **`EXEC`** passes the event to `NOTIFYCMD`, which is upssched. **No EXEC means no timers.** Every event you reference in `upssched.conf` needs EXEC here.
- **`CAL` / `NOTCAL`** are the calibration events (NUT 2.8+). They're needed so a battery test doesn't trigger a shutdown (see 2.10). Check that your version supports them with `man upsmon.conf | grep -i -A2 "CAL"`.
- **`HOSTSYNC 15`**: once FSD is raised, the primary waits up to 15 s for secondaries to log off before it shuts down.
- **`FINALDELAY 5`**: an extra pause before the primary runs `SHUTDOWNCMD`.
- **`POWERDOWNFLAG`** marks that the UPS should cut its outlets at the end of the shutdown. **See the warning in Part 5** before relying on it in a VM setup.
- **`NOTIFYCMD` path:** on Ubuntu 24.04, `/sbin` and `/usr/sbin` are the same directory, so either works. Confirm with `which upssched`.

## 2.10 `/etc/nut/upssched.conf` (timers)

```ini
CMDSCRIPT /etc/nut/upssched-cmd
PIPEFN /run/nut/upssched.pipe
LOCKFN /run/nut/upssched.lock

# Power lost: start the timers
AT ONBATT * START-TIMER onbatt 15
AT ONBATT * START-TIMER earlyshutdown <seconds>

# Power back: cancel EVERY timer ONBATT started
AT ONLINE * CANCEL-TIMER onbatt online
AT ONLINE * CANCEL-TIMER earlyshutdown

# Battery test or calibration: never shut down
AT CAL * CANCEL-TIMER earlyshutdown

# Low battery reported by the UPS: shut down now (backstop)
AT LOWBATT * EXECUTE shutdowncritical

# Lost contact with the UPS
AT COMMBAD * START-TIMER commbad 30
AT COMMOK  * CANCEL-TIMER commbad commok
AT NOCOMM  * EXECUTE commbad
```

The rules that matter:

1. **Every `START-TIMER` needs a `CANCEL-TIMER` on the event that undoes it.** A timer is a lit fuse. If nothing cancels it on `ONLINE`, any power blip, even one second, ends in a shutdown.
2. **`CAL` must cancel the shutdown timer.** A deep test or calibration runs the UPS on battery *on purpose*, so `ONBATT` fires. Without this line, the test shuts the server down.
3. **`CANCEL-TIMER <name> <cmd>`** cancels the timer, and if no timer was running it runs `<cmd>` instead. That's how `online` gets logged.
4. `/run/nut` is created at boot by NUT's tmpfiles config, so the pipe and lock paths always exist.

**Choosing `<seconds>` for `earlyshutdown`:**

```
timer = runtime at your normal load − time for a full clean shutdown − buffer (≥30 s)
```

- Runtime: `upsc apc@localhost battery.runtime`. Check it after a calibration, at your normal load.
- Shutdown time: measure it in Part 4, Test 3.
- Example: 600 s runtime − 240 s shutdown − 60 s buffer = **300 s**.
- On a weak battery (~141 s of runtime), a 120 s timer leaves only ~20 s to shut down, which is not enough. Fix the battery first, or use a much shorter timer.

## 2.11 `/etc/nut/upssched-cmd`

```bash
sudo nano /etc/nut/upssched-cmd
```

```bash
#!/bin/sh
UPS="apc@localhost"

case $1 in
  online)
    logger -t upssched-cmd "UPS back on line power" ;;
  onbatt)
    logger -t upssched-cmd "UPS on battery" ;;
  earlyshutdown)
    # Safety net: never shut down during a calibration or battery test
    if upsc "$UPS" ups.status 2>/dev/null | grep -q CAL; then
      logger -t upssched-cmd "Calibration in progress - skipping early shutdown"
      exit 0
    fi
    logger -t upssched-cmd "On battery too long - forcing shutdown (FSD)"
    /usr/sbin/upsmon -c fsd ;;
  shutdowncritical)
    logger -t upssched-cmd "Battery critical - forcing shutdown (FSD)"
    /usr/sbin/upsmon -c fsd ;;
  commbad)
    logger -t upssched-cmd "Lost communication with UPS" ;;
  commok)
    logger -t upssched-cmd "Communication with UPS restored" ;;
  *)
    logger -t upssched-cmd "Unrecognized command: $1" ;;
esac
```

```bash
sudo chmod 755 /etc/nut/upssched-cmd
sudo chown root:nut /etc/nut/upssched-cmd
```

`upsmon -c fsd` doesn't power anything off directly. It raises the **forced shutdown** flag, and every machine monitoring the UPS then shuts down in the right order (Part 5).

## 2.12 Start and verify the server

```bash
sudo systemctl restart nut-driver-enumerator
sudo systemctl restart nut-driver@apc nut-server nut-monitor
sudo systemctl enable nut-server nut-monitor

upsc apc@localhost ups.status        # expect: OL
upsc apc@localhost                   # full variable list
upsc apc@localhost ups.serial        # 0B2548L00791
```

- **`nut-driver-enumerator`** reads `ups.conf` and creates one `nut-driver@<name>` service per UPS. Re-run it whenever you change `ups.conf`.
- **"Init SSL without certificate database"** is harmless. It only means SSL isn't set up.

---

# Part 3 — NUT client (Proxmox host)

## 3.1 Install

```bash
apt update
apt install nut-client
```

The client doesn't talk to the UPS directly, so it needs only upsmon: no driver, no upsd. **Don't install the full `nut` package here.** A second driver fighting over the same UPS causes `Can't claim USB device`.

## 3.2 Test the connection before configuring

```bash
upsc apc@<server-ip> ups.status      # expect: OL
```

If this fails, fix it before going further. Check the server's `LISTEN` line, the firewall on TCP 3493, and that the server is up.

## 3.3 `/etc/nut/nut.conf`

```ini
MODE=netclient
```

## 3.4 `/etc/nut/upsmon.conf` (client)

```ini
MONITOR apc@<server-ip> 1 upsmon_remote <strong-password-2> secondary

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POLLFREQ 5
POLLFREQALERT 2
DEADTIME 15

NOTIFYFLAG ONLINE   SYSLOG+WALL
NOTIFYFLAG ONBATT   SYSLOG+WALL
NOTIFYFLAG LOWBATT  SYSLOG+WALL
NOTIFYFLAG FSD      SYSLOG+WALL
NOTIFYFLAG COMMOK   SYSLOG+WALL
NOTIFYFLAG COMMBAD  SYSLOG+WALL
NOTIFYFLAG SHUTDOWN SYSLOG+WALL
NOTIFYFLAG NOCOMM   SYSLOG+WALL
```

- **The username and password must exactly match `[upsmon_remote]` in the server's `upsd.users`.** That's the #1 reason a client won't connect.
- **No `EXEC` and no upssched** on the client, because the server owns the timers. When the server raises FSD, this client sees it and runs `SHUTDOWNCMD`.
- If you'd rather a client ran its own early-shutdown timer, copy the server's NOTIFYCMD/EXEC/upssched setup, but have its `earlyshutdown` run `/sbin/shutdown -h +0` instead of `upsmon -c fsd`. A secondary shouldn't force the whole UPS group down.

```bash
chmod 640 /etc/nut/upsmon.conf
chown root:nut /etc/nut/upsmon.conf
```

## 3.5 Start and verify the client

```bash
systemctl restart nut-monitor
systemctl enable nut-monitor
systemctl status nut-monitor --no-pager
journalctl -u nut-monitor -n 20 --no-pager
```

What you should see: `Communications with UPS apc@<server-ip> established`, and no login or access errors.

On the server, check that the client is logged in:

```bash
upsc -c apc@localhost          # lists connected clients: localhost + the Proxmox host IP
```

## 3.6 Proxmox guest shutdown settings

When the host gets `shutdown`, Proxmox stops every guest first. How cleanly that happens depends on a few settings:

- **Start/Shutdown order:** guests stop in the reverse of their startup order. Give the **NUT server VM order 1**, so it starts first and stops last.
- **Shutdown timeout:** Proxmox waits this long per guest before forcing it off (the default is 180 s). Every hung guest eats into your battery time.
- **QEMU guest agent:** install it in each VM and enable *QEMU Guest Agent* in VM → Options. Shutdown then becomes a reliable request instead of an ACPI button press the guest might ignore. This matters most for TrueNAS.

---

# Part 4 — Testing (from no risk up to a real shutdown)

**Test 1: watch only (no risk).** On the server:

```bash
journalctl -f -t upssched-cmd -t upsmon -u nut-monitor
```

Pull the UPS's wall plug for about 10 s, then plug it back in. You should see: `ONBATT` → timers started → `ONLINE` → timers cancelled → `online` logged. **Nothing shuts down.**

**Test 2: dry run of the timer (no shutdown).** Temporarily comment out both `/usr/sbin/upsmon -c fsd` lines in `upssched-cmd`, and set `earlyshutdown` to 20 s in `upssched.conf`. Then:

```bash
sudo systemctl restart nut-monitor
```

Unplug the UPS until the "forcing shutdown" log line appears, then plug it back in. **Put both changes back afterwards.**

**Test 3: real FSD (planned).** Pick a time when downtime is OK, and power off anything you can't afford to lose. Then:

```bash
sudo upsmon -c fsd
```

Watch Proxmox stop its guests, the client shut down, and the server shut down last. **Time it.** That number goes into the timer formula in 2.10.

**Battery tests** (use the `admin` user):

```bash
upscmd -l apc@localhost                                           # see what your UPS supports
upscmd -u admin -p <strong-password-3> apc@localhost test.battery.start.quick
```

Only run a deep test or calibration **after** the `CAL` cancel rule is in place.

**Check the battery's health:**

```bash
upsc apc@localhost | grep -E 'battery\.(charge|runtime|date|mfr\.date)'
```

Passing the self-test doesn't mean the runtime is healthy. Compare the battery date with when you installed it.

---

# Part 5 — How a safe shutdown works (and the VM gotcha)

The order of events:

1. The timer runs out, or the UPS reports `LB`. The primary sets **FSD**.
2. Each secondary sees FSD, runs `SHUTDOWNCMD`, and logs off.
3. The primary waits up to `HOSTSYNC` for the secondaries to log off.
4. After `FINALDELAY`, the primary runs its own `SHUTDOWNCMD`.
5. Optional: late in the primary's shutdown, the UPS is told to cut its outlets (`POWERDOWNFLAG`). The UPS turns them back on when wall power returns, and machines with BIOS "power on after AC loss" boot back up.

**⚠ If the NUT server is a VM on the host it protects:**

- **Don't let the UPS cut power.** The NUT VM can finish shutting down while the host is still stopping other guests, such as TrueNAS. If the UPS cuts power at that point, those guests lose power partway through shutting down. Either remove `POWERDOWNFLAG` from the server's `upsmon.conf`, or make sure the UPS can't be powered off from here.
- **Power returning during a shutdown:** the host finishes shutting down, and because the UPS outlets never went off, nothing turns it back on. The BIOS setting doesn't help, because the power never actually dropped. Bring the host back with IPMI instead:
  ```bash
  ipmitool -I lanplus -H <ipmi-ip> -U <user> -P <pass> power on
  ```
  You can also use the IPMI web page.

---

# Part 6 — Troubleshooting (errors we actually hit)

| Error | Meaning | Fix |
|---|---|---|
| `Error: Driver not connected` (from `upsc`) | upsd is running but the driver isn't | Run `sudo upsdrvctl -D start` to see the real error |
| `Could not open any HID devices: insufficient permissions on everything` | The `nut` user can't access the USB device file | udev rule (2.4), then reload and trigger |
| `Can't claim USB device [051d:0002]: Other error` | Something else holds the device, or it never finished setting up | Check `pgrep -a upower`, `ps aux \| grep [u]sbhid-ups` (on the host too), `dmesg` |
| `can't set config #1, error -32` (in `dmesg`) | The USB device never finished setting up in the VM | Cold-stop the VM, uncheck **Use USB3**, start it again; if needed, replug the UPS's USB cable |
| `lsusb -t` shows the UPS with no `If 0` / `Driver=` | Same as above | Same as above |
| `nut-scanner: Cannot load USB library (libusb-1.0.so)` | Ubuntu packaging quirk in nut-scanner only | `sudo apt install libusb-1.0-0-dev`. The driver itself is unaffected. |
| `upsnotify: failed to notify about state 4` | The driver was started by hand instead of by systemd | Harmless |
| `Init SSL without certificate database` | SSL isn't configured | Harmless on a LAN |
| Client: `Login on UPS failed` / `ACCESS-DENIED` | The username or password doesn't match `upsd.users` | Make the two files match exactly, then restart upsd and upsmon |
| Client: `Connection refused` | upsd isn't listening on the LAN IP, or a firewall is blocking it | Check the `LISTEN` line and TCP 3493 |
| Server shuts down after a short blip | A `START-TIMER` with no `CANCEL-TIMER` | Add `AT ONLINE * CANCEL-TIMER <name>` |
| Server shuts down during a battery test | `ONBATT` fired during the test | Add `AT CAL * CANCEL-TIMER earlyshutdown` + `NOTIFYFLAG CAL … EXEC` |
| Timers never fire | `EXEC` is missing on the NOTIFYFLAG, or `NOTIFYCMD` isn't set | Add `EXEC` and check `which upssched` |

**Useful commands:**

```bash
upsc apc@localhost                        # all UPS data
upsc -c apc@localhost                     # connected clients
upscmd -l apc@localhost                   # supported commands
upsrw apc@localhost                       # writable settings
sudo upsdrvctl -D start                   # driver in the foreground, verbose
journalctl -u nut-driver@apc -u nut-server -u nut-monitor -f
```

---

# Appendix — Differences from the Techno Tim guide

This guide was checked against Techno Tim's NUT guide. These are the deliberate differences:

| Techno Tim | This guide | Why |
|---|---|---|
| `upssched.conf` starts `earlyshutdown` on ONBATT with **no cancel** | Adds `AT ONLINE * CANCEL-TIMER earlyshutdown` | Without it, any blip causes a full shutdown. This is the exact bug you hit. |
| No calibration handling | `CAL` cancel rule + a `CAL` check in the script | Otherwise battery tests trigger a shutdown |
| One user (`admin`, `master`) shared by the server and clients | Separate `primary`, `secondary` and `admin` users | Least privilege: clients can't run commands on the UPS |
| `LISTEN 0.0.0.0 3493` | Listens on localhost + the LAN IP, with a firewall rule | Smaller attack surface |
| The udev rule only covers Tripp Lite `09ae:2012`, and restarts the driver via `RUN+=` | An APC `051d:0002` rule, permissions only | Matches this UPS; systemd already restarts the driver |
| Timers on the **client** | Timers on the **server**; clients follow FSD | One place to configure, and the shutdown order is controlled by the primary |
| Client `RUN_AS_USER root` | Not set (the default `nut` user) | Not needed, because the client only reads status |
| `PIPEFN`/`LOCKFN` in `/etc/nut` | `/run/nut` | `/run/nut` always exists and is the correct place for runtime files |
| Install `nut nut-client nut-server` | `nut` on the server, `nut-client` only on clients | A driver on the client can fight over the USB device |
| Not covered | VM USB passthrough (no USB3), upower, the nut-scanner lib fix, Proxmox guest shutdown order, the VM + power-cut gotcha | Specific to this setup |

## Sources

- [Techno Tim — Network UPS Tools (NUT) Ultimate Guide](https://technotim.com/posts/NUT-server-guide/)
- NUT man pages on your system: `man ups.conf`, `man upsd.users`, `man upsmon.conf`, `man upssched.conf`
