# Kali Linux VM — Network Troubleshooting Checklist

> Personal reference + community resource. Covers VirtualBox and VMware.  
> Most "WiFi not working" issues in a Kali VM are actually **ethernet/bridged adapter misconfigurations** — not WiFi at all.
> Also note that sometimes changing from "bridged adapter" to "NAT" on the virtual box network settings also helps. 
> You can also check whether your host OS is correctly configured to network and dhcp was successful or failed using "ipconfig" for windows and "ifconfig" for mac. If dhcp has failed it shows a self assigned ip address===>>> 169.254.x.x and Default Gateway: (blank) BUT if it was successful it gives a real private ip + a gateway ==>  192.168.x.x  (or 10.x.x.x / 172.16-31.x.x) and Default Gateway: 192.168.x.1

---

## Quick Orientation

| Symptom | Likely Layer |
|---|---|
| No interface except `lo` | VM adapter not attached / driver missing |
| Interface exists, no IP | DHCP failure / NM not managing it |
| Has IP, no internet | Wrong gateway / DNS broken |
| Ping works, browser doesn't | DNS issue |
| Works on NAT, not Bridged | Host adapter mismatch in VM settings |

---

## Step 0 — Understand What You're Working With

In a VM, there's no "WiFi" unless you've passed through a physical USB WiFi adapter.  
What you have is a **virtual ethernet interface** (`eth0`) bridged or NAT'd to your host's network.

```bash
ip link show          # list all interfaces
ip addr show          # interfaces + IPs
nmcli device status   # NM view of all devices
```

---

## Step 1 — Check Interface State

```bash
ip addr show eth0
```

**Read the flags:**

| Flag | Meaning |
|---|---|
| `UP` | Interface is administratively up |
| `LOWER_UP` | Physical/link layer is connected |
| `inet x.x.x.x` | Has an IP — DHCP worked |
| No `inet` line | No IP assigned |

**If interface is DOWN:**
```bash
sudo ip link set eth0 up
```

---

## Step 2 — Check NetworkManager

```bash
nmcli device status
```

**States to know:**

| State | Meaning |
|---|---|
| `connected` | NM managing it, has IP |
| `disconnected` | NM sees it but isn't connecting |
| `unmanaged` | NM is ignoring this interface |
| `unavailable` | Link is down |

**If `unmanaged`:**
```bash
nmcli device set eth0 managed yes
sudo systemctl restart NetworkManager
```

**If `disconnected`:**
```bash
nmcli connection show        # list profiles
```

---

## Step 3 — Fix NetworkManager Connection Profile

**Profile exists but DEVICE column is `--` (not bound to eth0):**
```bash
nmcli connection modify "Wired connection 1" connection.interface-name eth0
nmcli connection up "Wired connection 1"
```

**No profile at all:**
```bash
nmcli connection add type ethernet ifname eth0 con-name eth0-conn
nmcli connection up eth0-conn
```

**Nuclear restart:**
```bash
sudo systemctl restart NetworkManager
nmcli connection up "Wired connection 1"
```

---

## Step 4 — Force DHCP Manually

```bash
sudo dhclient -r eth0          # release existing lease
sudo dhclient -v eth0          # request new IP (-v shows DISCOVER/OFFER exchange)
```

**Read the `-v` output:**
- `DHCPDISCOVER` sent but no `DHCPOFFER` → DHCP server unreachable → bridged adapter pointing to wrong host interface
- `DHCPOFFER` received but fails → lease conflict or firewall

---

## Step 5 — Check Routing

```bash
ip route
```

**Expected output (example):**
```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.x
```

**If no default route:**
```bash
sudo ip route add default via <gateway-ip> dev eth0
# Find gateway: check your host OS or router (usually 192.168.1.1 or 192.168.0.1)
```

---

## Step 6 — Check DNS

```bash
ping 8.8.8.8          # tests routing (no DNS)
ping google.com        # tests DNS
```

**Can ping IP but not domain → DNS broken:**
```bash
cat /etc/resolv.conf
```

**Fix — add nameservers:**
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf
```

**Permanent fix (stops NM from overwriting resolv.conf):**
```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
# Under [main] add:
# dns=none
sudo systemctl restart NetworkManager
```

---

## Step 7 — VirtualBox Adapter Settings (Most Common Root Cause)

**Shut down the VM first.**

Go to: `Settings → Network → Adapter 1`

| Mode | Use Case |
|---|---|
| **NAT** | Simple internet access, VM can't be reached from host |
| **Bridged** | VM gets its own IP on your LAN, best for pentesting |
| **Host-only** | VM ↔ Host only, no internet |
| **NAT Network** | Like NAT but multiple VMs can talk to each other |

**Bridged mode gotcha:** The "Name" dropdown must match your **currently active host adapter**.

- Host on WiFi → bridge to your WiFi adapter (e.g., `Intel Wireless-AC 9560`)
- Host on ethernet → bridge to your ethernet adapter
- Switching between WiFi/ethernet on host = VM loses connectivity

**Quick sanity check:** Switch to NAT and boot. If internet works → bridged adapter was misconfigured.

---

## Step 8 — VMware Specific

```bash
# If using VMware and network breaks after host sleep/resume:
sudo systemctl restart vmware-tools   # or open-vm-tools
sudo systemctl restart NetworkManager
```

VMware NAT service sometimes dies:
- Windows host: restart `VMware NAT Service` in services.msc
- Linux host: `sudo /etc/init.d/vmware restart`

---

## Step 9 — Physical WiFi Adapter Passthrough (USB)

If you actually want a **real WiFi interface (`wlan0`)** in your VM:

1. Plug in a USB WiFi adapter (e.g., Alfa AWUS036ACH)
2. VirtualBox: `Settings → USB → Add filter` for your adapter
3. Boot VM, verify with `ip link show`
4. Connect:
```bash
nmcli device wifi list
nmcli device wifi connect "SSID" password "yourpassword"
```

**For monitor mode (needed for wireless pentesting):**
```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan0
iwconfig        # confirm wlan0mon exists
```

---

## Full Reset Sequence (When Nothing Works)

```bash
sudo ip link set eth0 down
sudo ip link set eth0 up
sudo dhclient -r eth0
sudo dhclient -v eth0
sudo systemctl restart NetworkManager
nmcli connection up "Wired connection 1"
ip addr show eth0
ping 8.8.8.8
```

---

## Useful Commands — Quick Reference

```bash
ip link show                          # all interfaces + state
ip addr show                          # interfaces + IPs
ip route                              # routing table
nmcli device status                   # NM device overview
nmcli connection show                 # NM connection profiles
nmcli device wifi list                # scan WiFi (if wlan0 exists)
sudo dhclient -v eth0                 # verbose DHCP request
sudo systemctl status NetworkManager  # NM service status
sudo systemctl restart NetworkManager
cat /etc/resolv.conf                  # DNS config
resolvectl status                     # detailed DNS status (systemd-resolved)
dmesg | grep -i eth                   # kernel messages for ethernet
dmesg | grep -i wlan                  # kernel messages for WiFi
```

---

## Common Scenarios → Fix Map

| Situation | Fix |
|---|---|
| `eth0` not listed at all | Check VM adapter is enabled in settings |
| `eth0` up, no IP, `ip route` empty | Run `dhclient -v eth0`, fix bridged adapter |
| NM shows `disconnected` | `nmcli connection up "Wired connection 1"` |
| NM shows `unmanaged` | `nmcli device set eth0 managed yes` |
| Works on NAT, not Bridged | Change bridged adapter Name to active host interface |
| Ping IP works, domain doesn't | Fix `/etc/resolv.conf` |
| Worked yesterday, broke today | Host switched WiFi ↔ ethernet, update bridged adapter target |

---

*Tested on Kali Linux 2024.x — VirtualBox 7.x*
