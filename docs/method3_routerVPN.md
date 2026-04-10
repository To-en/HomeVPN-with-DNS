# Option 3 — use Router as VPN server (Production Setup)

---

## How It Works

```
[Phone / Laptop - anywhere]
          │
    WireGuard tunnel (UDP 51820)
          │
  [DuckDNS → home public IP]
          │
  [Home ONU Router - set Bridge mode]
          │
  [Mercusys MR60X Router - 192.168.1.1]
   - WireGuard VPN server 
   - Firewall + NAT
   - DHCP server
          │
  [Home LAN]
   -  RPi4.home 
   - mypc.home
   ...
```

Once tunneled into Mercusys VPN Server, 
RPi act as dns server that provide domain name to each device which has been binded to specific DHCP ip address  

**For Wireguard clinet, do remember to save only one config per machineone client profile, no switching.**

---

## Requirements

- Mercysys MR60X 
(Any router that has native Wireguard support)
- Raspberry Pi 4 (DNS or Pi-hole, depends on user)
- DuckDNS account — [duckdns.org](https://www.duckdns.org) (free)
- WireGuard app on client devices

> **Keywords to look up:** RouterOS bridge mode, MikroTik DMZ vs bridge, Winbox download

---

## Setup

### 1. Put ONU Router in Bridge mode
to make it passes the public IP directly to Mercusys, so our custom router owns the WAN.

Two approaches
- **Bridge mode** — full pass-through, preferred
- **DMZ host** — forwards all unsolicited traffic to Mercusys, simpler to set up but it is not a direct solution. 

### 2. MikroTik — Basic WAN Setup

After ZTE is in bridge mode, MikroTik's WAN port gets the public IP via DHCP from AIS.

### 3. DuckDNS

Same as Option 2 — run the update script on RPi or any always-on device:

```bash
crontab -e
# Add:
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=YOUR_SUBDOMAIN&token=YOUR_TOKEN&ip=" > ~/duckdns.log
```

### 4. WireGuard on Mercusys MR60X

After setting up the router, read this guide on how to set Wireguard VPN on Mercusys 
https://www.mercusys.co.th/faq-1081/
it is very simple
1. Config Tunnel address and specified port 51820 UDP, in this case we use 10.0.0.1/24
2. Generate private and public key, don't forget to note
![image](/asset/wireguardMR60x.png)
3. 
![image](/asset/wireguardMR60x1.png)

![image](/asset/exportClientKey.png)

```
in our case we will use
```

> **Keyword to look up:** MikroTik WireGuard setup RouterOS 7, MikroTik WireGuard peer config

### 5. Add Client Peer on client machine

Generate keypair on any Linux machine (or RPi):
```bash
wg genkey | tee client.key | wg pubkey > client.pub
```

In Winbox:
```
WireGuard → Peers → Add
  Interface: wg0
  Public Key: <contents of client.pub>
  Allowed Address: 10.0.0.2/32
```

Client config (import into WireGuard app):
```ini
[Interface]
PrivateKey = <contents of client.key>
Address = 10.0.0.2/24
DNS = 192.168.1.x        # RPi's LAN IP

[Peer]
PublicKey = <Copy WireGuard public key here>
Endpoint = <YOUR_SUBDOMAIN>.duckdns.org:51820
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
# 192.168.1.0/24 is common subnet in Thai network
PersistentKeepalive = 25
```

### In case you want to SSH easily

### 1. Do DHCP reservation for your frequent used device
This will depend on each router setting page

### 2. Make Rasberyy Pi a DNS server 

We need to make pi has static ip first 
```bash
sudo nmcli con mod "Wired connection 1" ipv4.address 192.168.1.253/24

sudo nmcli con mode "Wired connection 1" ipv4.gateway 192.168.1.1

sudo nmcli con mode "Wired connection 1" ipv4.dns 127.0.0.1
# Pi itself act as DNS so we fill loopback ip
sudo reboot
```


Add local DNS record by mapping the DHCP reserved ip to domain name
```
abc
```

---

## Done, to use the VPN

- Open wireguard app
- Connect WireGuard profile → inside home LAN
- `ssh user@mypc.home` works
- Pi-hole filters ads for all devices including VPN clients
- MikroTik handles all routing — RPi only does DNS

#### In case the client have no GUI
- [Install Wireguard in Terminal & Quickstart](https://www.wireguard.com/quickstart/#key-generation)
- Write a client config file by yourself, here's the standard way to write it: [Samples client.conf](https://gist.github.com/lanceliao/5d2977f417f34dda0e3d63ac7e217fd6)

---

## Checklist

- [ ] Your ONU router set in bridge mode — Mercusys MR60X gets public IP on WAN and becomes the main router
- [ ] Register DuckDNS, and create Duckdns script to run automatically on RPi using cron
- [ ] Create Mercusys WireGuard interface created ( port `51820`, ip subnet = up to you)
- [ ] Make sure Firewall inbound rule doesn't block UDP 51820
- [ ] Client 
- [ ] Client config imported into WireGuard app
- [ ] Pi-hole installed on RPi (DHCP off)
- [ ] MikroTik DHCP points to RPi as DNS
- [ ] Test: `ssh user@mypc.home` from outside network