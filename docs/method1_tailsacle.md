# Option 1 — Tailscale (Mesh VPN)

> Quickest path to remote home access. No port forwarding, no DDNS, works behind any ISP router.

**Trade-off:** Tailscale's coordination server brokers your connections. You trust them, not just your own hardware.

---

## How It Works

```
[Phone / Laptop — anywhere]
        │
  Tailscale coordination server (NAT traversal)
        │
  [RPi4 — subnet router]
        │
  [Home LAN — all devices accessible by IP]
```

Your RPi acts as a **subnet router** — it advertises your home LAN (`192.168.1.0/24`) to your Tailscale network, so all home devices become reachable without installing Tailscale on each one.

---

## Requirements

- Raspberry Pi 4 (always-on, connected to home LAN)
- Tailscale account — [tailscale.com](https://tailscale.com) (free tier: 3 users, 100 devices)
- Tailscale app on client devices (Android / iOS / Windows / Linux / macOS)

---

## Setup

### 1. Install Tailscale on RPi

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

### 2. Approve Subnet Route

- Go to [Tailscale admin panel](https://login.tailscale.com/admin/machines)
- Find your RPi → Edit route settings → Enable `192.168.1.0/24`

### 3. Enable IP Forwarding on RPi

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 4. Install Tailscale on Client Devices

Download app → sign in with same account → you're on the same tailnet.

### 5. Access Home Devices

```bash
# By LAN IP directly (subnet routing)
ssh user@192.168.1.50
ping 192.168.1.120        # ESP32

# Or by Tailscale-assigned name
ssh user@raspberrypi
```

---

## Optional — Pi-hole as DNS over Tailscale

Install Pi-hole on RPi, then in Tailscale admin panel set your RPi's Tailscale IP as the global DNS server. All devices on your tailnet get ad blocking automatically.

> **Keywords to look up:** Tailscale MagicDNS, Tailscale exit node, Tailscale ACLs

---

## Checklist

- [ ] Tailscale account created
- [ ] Tailscale installed on RPi
- [ ] Subnet route `192.168.1.0/24` approved in admin panel
- [ ] IP forwarding enabled on RPi
- [ ] Tailscale app installed on client devices
- [ ] Test: `ping 192.168.1.1` from outside home network