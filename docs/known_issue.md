## Known Issues & Fixes

### Internet dies when WireGuard activates on client

**Symptom:** Client connects to WireGuard, handshake completes, LAN devices are reachable, but all internet/DNS stops working.

**Root cause:** Client config has `DNS = 10.0.0.1` pointing to RPi's dnsmasq. But dnsmasq started at boot *before* WireGuard created the `wg0` interface — so dnsmasq never bound to `wg0`. DNS queries from VPN clients go into a black hole.

**Quick fix:** After bringing up WireGuard on RPi, restart dnsmasq:
```bash
sudo systemctl restart dnsmasq
```

**Permanent fix:** Add a systemd override so dnsmasq always starts after WireGuard:
```bash
sudo systemctl edit dnsmasq
```
Add:
```ini
[Unit]
After=wg-quick@wg0.service
Requires=wg-quick@wg0.service
```

> **Keyword to look up:** systemd service After= Requires=

### SSH to RPi breaks when WireGuard activates on same machine

**Symptom:** You're SSH'd into RPi from your desktop. You bring up WireGuard on that same desktop. SSH session freezes.

**Root cause:** `AllowedIPs = 10.0.0.0/24, 192.168.1.0/24` in the client config routes the entire `192.168.1.0/24` subnet through the tunnel. Your existing SSH session (direct LAN → RPi at `192.168.1.x`) gets hijacked into the tunnel and breaks.

**Fix — add a static route before activating WireGuard:**
```bash
sudo ip route add 192.168.1.1/32 via <your-gateway> dev eth0
sudo wg-quick up wg0
```

**Better fix:** Test WireGuard from a *separate* device (phone) so RPi SSH stays unaffected.

### `wg show` is empty on client

**Symptom:** WireGuard config file is in `/etc/wireguard/wg0.conf` but `sudo wg show` shows nothing.

**Fix:** The interface isn't up yet. Explicitly start it:
```bash
sudo wg-quick up wg0
```

### Handshake stuck — `0B received, 296B sent`

**Checklist:**
- Is port `51820/UDP` actually forwarded? Test with `sudo nc -ul 51820` on RPi + `echo test | nc -u <public-ip> 51820` from outside
- Is DuckDNS resolving to the right IP? `nslookup YOUR_SUBDOMAIN.duckdns.org`
- Are server and client using each other's *public* keys (not private)?
- Is RPi's firewall allowing UDP 51820? `sudo ufw status` or `sudo iptables -L INPUT`

---

## Checklist

- [ ] RPi static IP set and rebooted
- [ ] DuckDNS subdomain created + cron update running
- [ ] Port `51820/UDP` forwarded on router
- [ ] WireGuard installed, `wg0` service running
- [ ] IP forwarding enabled
- [ ] DNS approach chosen (A or B) and dnsmasq configured
- [ ] If Approach A: router DHCP disabled
- [ ] If Approach B: DHCP reservations set on router, DNS pointed to RPi
- [ ] dnsmasq restart ordering fixed (systemd dependency)
- [ ] Client config imported into WireGuard app
- [ ] Test: `ssh user@mypc.home` from outside network