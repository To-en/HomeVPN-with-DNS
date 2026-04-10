# Option 2 — Pure RPi WireGuard (Self-Hosted VPN)

---

## How It Works

```
[Phone / Laptop — anywhere]
          │
    WireGuard tunnel (UDP 51820)
          │
  [DuckDNS → home public IP]
          │
  [ZTE Router — port forward 51820 → RPi]
          │
  [RPi4 — 192.168.1.1]
   ├── WireGuard server  (wg0: 10.0.0.1)
   ├── Pi-hole           (DNS + ad blocking)
   └── dnsmasq           (DHCP, built into Pi-hole)
          │
  [Home LAN]
   ├── mypc.home
   └── esp32.home
```

---

## Requirements

- Raspberry Pi 4 (always-on, wired ethernet)
- DuckDNS account — [duckdns.org](https://www.duckdns.org) (free)
- Access to ZTE router admin page (port forwarding)
- WireGuard app on client devices

---

## Setup

### 1. RPi Static IP

Type in terminal

```bash
sudo nmcli con mod "Wired connection 1" ipv4.address 192.168.1.253/24

sudo nmcli con mode "Wired connection 1" ipv4.gateway 192.168.1.1

sudo nmcli con mode "Wired connection 1" ipv4.dns 127.0.0.1
# Pi itself act as DNS so we fill loopback ip

sudo reboot
```

### 2. DuckDNS
- Register Duckdns account
- Create subdomain at duckdns.org → note your token
- Add cron job script to keep IP updated:

```bash
crontab -e
# Add:
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=YOUR_SUBDOMAIN&token=YOUR_TOKEN&ip=" > ~/duckdns.log
```

### 3. Port Forwarding on your Router
Forward `51820/UDP → 192.168.1.1` in your ZTE admin panel.





---

## Checklist

- [x] RPi static IP set and rebooted
- [x] DuckDNS subdomain created + cron update running
- [x] Port `51820/UDP` forwarded on ZTE router
- [ ] WireGuard installed, `wg0` service running
- [ ] IP forwarding enabled
- [ ] Pi-hole installed, DHCP active
- [ ] ZTE router DHCP disabled
- [ ] Client config imported into WireGuard app
- [ ] Test: `ssh user@mypc.home` from outside network