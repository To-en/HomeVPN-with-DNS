# 🏠 Home VPN Gateway

> Access any device on your home network — by name — from anywhere in the world. Free, self-hosted, no third-party mesh services.

Built on a **Raspberry Pi 4** running WireGuard + dnsmasq. Once connected, `ssh user@mypc.home` just works, whether you're across the street or across the world.

---

## How It Works

```
[Your phone / laptop — anywhere]
			  	│
    WireGuard tunnel (UDP 51820)
        	│
  [DuckDNS -> home public IP]
        	│
  [Router: port forward 51820]
		- DHCP (192.168.1.100 – 200)
        	│
  [Raspberry Pi 4 — 192.168.1.1]
   - WireGuard server  (10.0.0.1) 
   - dnsmasq DNS       (*Static Mapping)
          │
	[Home LAN]
   - mypc.home
   - rpi.home
	 ...
```

New devices on your LAN register their own `.home` name automatically — no manual DNS entries needed.

### For some one with CGNAT Router
```
[Your phone / laptop — anywhere]
          │
    WireGuard tunnel (ISP DDNS provided UDP Port)
          │
  [ISP DDNS provided domain name → home public IP]
          │
  [Router: port forward 5540 (Some port that ISP gives)]
		- DHCP (192.168.1.100 – 200)
        	│
  [Raspberry Pi 4 — 192.168.1.1]
   - WireGuard server  (10.0.0.1) 
   - dnsmasq DNS       (*Static Mapping)
          │
	[Home LAN]
   - mypc.home
   - rpi.home
	 ...
```
CGNAT prevent the home router to have static public ip, then we need to call ISP provider to enable their own DDNS service that will map the changing home public ip to their domain name  
<br>
In this case Duckdns is not required we will use the DDNS service that the ISP provided, in my case it is AIS Fibre THDDNS

---

## Equipment

| Item | Notes |
|------|-------|
| Raspberry Pi 4 | 2GB+ recommended |
| Ethernet cable | RPi should be wired |
| Home router | Needs port forwarding support |
| DuckDNS account | Free at [duckdns.org](https://www.duckdns.org) |
| WireGuard client | App for Android / iOS / Windows / Linux / macOS |

---

## Setup Guide

### 1. Raspberry Pi — Static IP

Edit `/etc/dhcpcd.conf` (use the file at `configs/system/dhcpcd.conf` as reference):

```bash
sudo nano /etc/dhcpcd.conf
```

Reboot after saving:
```bash
sudo reboot
```

---

### 2. DuckDNS — Dynamic DNS

1. Sign in at [duckdns.org](https://www.duckdns.org) and create a subdomain (e.g., `myhome.duckdns.org`)
2. Copy your token from the dashboard
3. Install the update script:

```bash
cp scripts/duckdns-update.sh ~/duckdns-update.sh
chmod +x ~/duckdns-update.sh

# Edit with your subdomain and token
nano ~/duckdns-update.sh

# Add to cron — runs every 5 minutes
crontab -e
# Add this line:
# */5 * * * * ~/duckdns-update.sh
```

---

### 3. Router — Port Forwarding

In your router admin panel, forward:

| Protocol | External Port | Internal IP | Internal Port |
|----------|--------------|-------------|---------------|
| UDP | 51820 | 192.168.1.1 | 51820 |

In case of CGNAT Router

| Protocol | External Port | Internal IP | Internal Port |
|----------|--------------|-------------|---------------|
| UDP | 5540 (ISP given port) | 192.168.1.1 | 51820 |


---

### 4. WireGuard — Server (RPi)

```bash
sudo apt update && sudo apt install wireguard -y

# Generate server keypair
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Copy server config
sudo cp configs/wireguard/wg0.conf /etc/wireguard/wg0.conf

# Fill in your server private key
sudo nano /etc/wireguard/wg0.conf
```

Write the following config
```ini
[Interface]
PrivateKey = <contents of server.key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Add one block per client — see step 6
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### 5. WireGuard — Client

Generate keypair for each client device:

```bash
wg genkey | tee client.key | wg pubkey > client.pub
```

Client config (import into WireGuard app):
```ini
[Interface]
PrivateKey = <contents of client.key>
Address = 10.0.0.2/24
DNS = 10.0.0.1

[Peer]
PublicKey = <contents of server.pub>
Endpoint = YOUR_SUBDOMAIN.duckdns.org:51820
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
PersistentKeepalive = 25
```

Add the client's public key as a `[Peer]` block in `wg0.conf`, then reload:
```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

This outputs a `client.conf` ready to import into the WireGuard app. Add the generated `[Peer]` block to `/etc/wireguard/wg0.conf` on the RPi, then reload:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```
### 6. Make Rpi a DNS

Install dnsmasq:
```bash
sudo apt install dnsmasq -y
/etc/dnsmasq.conf:
confinterface=eth0
interface=wg0
```
fill static DNS entries
```conf
# Example
address=/mypc.home/192.168.1.120
address=/laptop.home/192.168.1.110
address=/esp32.home/192.168.1.115
address=/rpi.home/192.168.1.253

Forward everything else to upstream DNS
server=1.1.1.1		# Cloudflare DNS
server=8.8.8.8		# Google DNS
```

---

### 7. Test It

```bash
# On your phone/laptop — connect WireGuard, then:
ping mypc.home
ssh user@mypc.home

# Check VPN status on RPi
sudo wg show
```

---

## Adding More Clients

Repeat step 5 for each device (phone, laptop, etc.). Each gets a unique keypair and a unique VPN IP (`10.0.0.x`).

---

## Checklist for you

- [ ] Set static IP for your Single Board Computer
  - If use RasberyPi OS, the hostname can be set from its OS installation. Can SSH to its host name. 
  - Other OS might need to set static IP via `dhcpcd.conf`
- [ ] Create DNS server with RasberryPi(RPi), DuckDNS subdomain created + update script running
- [ ] Router port `51820/UDP` forwarded to RPi
- [ ] WireGuard installed and `wg0` service running
- [ ] dnsmasq installed and configured
- [ ] Router DHCP disabled
- [ ] Client config imported into WireGuard app
- [ ] `ssh hostname.home` works over VPN tunnel

---

## Troubleshooting

| Problem | Check |
|---------|-------|
| Can't connect to VPN | DuckDNS updated? Port 51820 forwarded? `sudo wg show` on RPi |
| `.home` names not resolving | Is dnsmasq running? Is DNS in client config set to `10.0.0.1`? |
| DHCP not working | Only one DHCP server active? Router's DHCP disabled? |
| RPi IP changed | Static IP set in `dhcpcd.conf`? |

---

## Security Notes

- WireGuard only responds to valid authenticated peers — unrecognized traffic is silently dropped
- Keep your private keys off version control — never commit `/etc/wireguard/*.key`
- Consider rotating client keys periodically if a device is lost