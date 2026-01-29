# Laptop-as-Router with Cloudflare WARP (Step-by-Step Linux Guide)

This is a guide to turn any Linux laptop into:

- a router / gateway for other devices
- a DHCP + DNS server
- a NAT device
- a Cloudflare WARP tunnel endpoint

All traffic from LAN clients is NATed into the Cloudflare WARP tunnel, making the laptop a full VPN gateway.

This guide was designed to work on any modern Linux distro that uses **NetworkManager** (Arch, Ubuntu, Fedora, Debian, EndeavourOS, etc.).

---

## 0. What This Setup Does

```
Upstream Network (Campus / Hostel / Public Wi-Fi)
            │
         wlan0 (Wi-Fi)
            │
         Laptop  ← Cloudflare WARP (always ON)
            │
      enpX (Ethernet, Shared)
            │
         Router (AP only)
            │
     Phones / Tablets / Laptops
```

- Laptop is the **only device** connected to the upstream network
- Router is used only as a **Wi-Fi access point**
- All LAN clients get IPs from the laptop
- Internet access is tunneled through Cloudflare WARP

---

## 1. Prerequisites

### 1.1 Hardware

- Linux laptop with Wi-Fi + Ethernet
- Wifi Router
- Ethernet cable

### 1.2 Required Packages

Install the following packages:

```bash
# Arch / EndeavourOS
sudo pacman -S networkmanager iptables cloudflare-warp

# Ubuntu / Debian
sudo apt install network-manager iptables cloudflare-warp

# Fedora
sudo dnf install NetworkManager iptables cloudflare-warp
```

Ensure NetworkManager is running:

```bash
sudo systemctl enable --now NetworkManager
```

---

## 2. Router Configuration (One Time)

1. Factory reset the router
2. Log into router admin page
3. Disable:
    - DHCP server
    - Firewall / parental controls (if configurable)
4. Set Internet / WAN type to **Automatic (DHCP)**
5. Do **not** configure PPPoE or static IP
6. Save and reboot router

The router is now a **dumb AP + switch**.

---

## 3. Laptop Network Setup

### 3.1 Connect to Upstream Wi-Fi

Connect normally to the upstream network (campus / hostel Wi-Fi).

Verify:

```bash
ip a show wlan0
```

You should see an IP assigned.

---

### 3.2 Share Ethernet to Other Computers

Identify the Ethernet connection name:

```bash
nmcli connection show
```

Modify it to shared mode (This can also be done via the NetworkManager GUI):

```bash
sudo nmcli connection modify <ethernet-connection-name> ipv4.method shared
sudo nmcli connection down <ethernet-connection-name>
sudo nmcli connection up <ethernet-connection-name>
```

Expected result:

- Ethernet interface gets `10.42.0.1/24`

Verify:

```bash
ip a show <ethernet-interface>
```

---

## 4. DHCP & DNS (Automatic)

When Ethernet sharing is enabled, NetworkManager automatically:

- starts `dnsmasq`
- assigns IPs `10.42.0.10 – 10.42.0.254`
- sets gateway and DNS to `10.42.0.1`

Verify:

```bash
ps aux | grep dnsmasq
```

---

## 5. Enable IPv4 Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Persist across reboot:

```bash
sudo nano /etc/sysctl.d/99-forwarding.conf
```

```conf
net.ipv4.ip_forward=1
```

Apply:

```bash
sudo sysctl --system
```

---

## 6. NAT

Masquerade traffic leaving Wi-Fi:

```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
```

Masquerade traffic leaving Cloudflare WARP:

```bash
sudo iptables -t nat -A POSTROUTING -o CloudflareWARP -j MASQUERADE
```

---

## 7. Cloudflare WARP (Core Step)

Register and connect:

```bash
warp-cli register
warp-cli connect
```

Verify:

```bash
warp-cli status
```

WARP must remain **ON** for laptop and downstream devices.

---

## 8. Critical Kernel Fix (VPN + NAT Stability)

Linux drops packets by default in multi-interface setups.  
This **must be fixed**.

Apply immediately:

```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=2
```

Persist:

```bash
sudo nano /etc/sysctl.d/99-multihomed.conf
```

```conf
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=2
```

Apply:

```bash
sudo sysctl --system
```

---

## 9. Firewall (Minimal Required Rules)

Allow LAN traffic:

```bash
sudo iptables -I INPUT -s 10.42.0.0/24 -j ACCEPT
sudo iptables -I OUTPUT -d 10.42.0.0/24 -j ACCEPT
```

---

## 10. Verification

### Laptop

```bash
ip a
ip route
warp-cli status
ps aux | grep dnsmasq
```

### Client Devices

Look for `warp=on` in the following link
```
https://cloudflare.com/cdn-cgi/trace
```

- IP: `10.42.0.x`
- Gateway: `10.42.0.1`
- DNS: `10.42.0.1`

Internet should now work without manual proxy configuration.

---

## 11. Optional: Discord Voice Issues

If Discord voice hangs (DTLS / checking routes):

- Ensure Section 8 (rp_filter) is applied
- Restart Discord

No Discord-specific config is normally required.

---

## 12. Optional: KDE Connect

KDE Connect works **only on real LANs**.

Manual pairing IP:

```text
10.42.0.1
```

If discovery fails, allow ports:

```bash
sudo iptables -I INPUT -p udp --dport 1716 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 1716:1764 -j ACCEPT
```

Bluetooth warnings can be ignored or disabled.

---

## 13. Optional: Network-Wide Ad Blocking (DNS-Level)

This section adds **network-wide ad and tracker blocking** for **all LAN clients**, similar to Pi-hole, using the **existing dnsmasq instance started by NetworkManager**.

---

### 16.1 How This Works

- NetworkManager already runs `dnsmasq` for the shared Ethernet interface
- We inject a DNS blocklist into **NetworkManager’s dnsmasq configuration**
- All LAN clients using `10.42.0.1` as DNS are filtered automatically
### 16.2 Create dnsmasq Override Directory

NetworkManager reads additional dnsmasq config from:

```bash
/etc/NetworkManager/dnsmasq-shared.d/
```

Create it if missing:

```bash
sudo mkdir -p /etc/NetworkManager/dnsmasq-shared.d
```

### 16.3 Add an Ad Blocklist

Create a blocklist file:

```bash
sudo nano /etc/NetworkManager/dnsmasq-shared.d/blocklist.conf
```

Add entries in this format:

```conf
address=/doubleclick.net/0.0.0.0
address=/googlesyndication.com/0.0.0.0
address=/ads.mopub.com/0.0.0.0
address=/ads.twitter.com/0.0.0.0
address=/ads.facebook.com/0.0.0.0
```

Save and exit.

### 16.4 Restart NetworkManager

Apply the changes:

```bash
sudo systemctl restart NetworkManager
```

This automatically restarts dnsmasq with the new rules.

### 16.5 Use a Maintained Blocklist (Recommended)

Instead of maintaining entries manually, use a curated list.

Example: **OISD basic list** (lightweight, low breakage)

```bash
sudo curl -s https://small.oisd.nl/domainswild \
| sed 's/^/address=\/\//' \
| sed 's/$/\/0.0.0.0/' \
| sudo tee /etc/NetworkManager/dnsmasq-shared.d/blocklist.conf
```

Restart:

```bash
sudo systemctl restart NetworkManager
```

### 16.6 Temporarily Disable Ad Blocking

To disable blocking without deleting configuration:

```bash
sudo mv /etc/NetworkManager/dnsmasq-shared.d/blocklist.conf /tmp/blocklist.conf
sudo systemctl restart NetworkManager
```

Re-enable:

```bash
sudo mv /tmp/blocklist.conf /etc/NetworkManager/dnsmasq-shared.d/
sudo systemctl restart NetworkManager
```

### 16.7 Whitelist Specific Domains

Create a whitelist file:

```bash
sudo nano /etc/NetworkManager/dnsmasq-shared.d/whitelist.conf
```

Example:

```conf
address=/accounts.google.com/#
address=/login.microsoftonline.com/#
address=/api.spotify.com/#
```

Restart NetworkManager.

### 16.9 Limitations

DNS-based blocking **cannot**:

- remove in-app sponsored content
- block ads served from first-party domains
- hide embedded social media ads

For browsers, combine with a client-side blocker (e.g. uBlock Origin).

---
## 14. Automation Script (Recommended)

All commands above can be automated.

This guide assumes an **automation script** (to be attached separately) that:

- enables forwarding
- applies NAT rules
- applies rp_filter settings
- connects WARP

When using the script:

```bash
sudo ./warp-router-on.sh
sudo ./warp-router-off.sh
```

The manual steps in this guide describe **exactly what the script does internally**.

---

## 15. Common Failures

### Clients cannot get IP

- Ethernet not in shared mode
- dnsmasq not running
- Router needs reboot

### Internet works on laptop but not clients

- NAT rules missing
- WARP not connected

### UDP / voice issues

- rp_filter not applied

---