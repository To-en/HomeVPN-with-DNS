# Home VPN Gateway

> How I setup my own VPN Server to access home network.<br>
> The original need is for me to SSH from my university.

Instead of using Tailscale I decided to built on a **Raspberry Pi 4 (Rpi)** <br>
as WireGuard server & DNS server.<br> 
Once connected, just `ssh user@mypc.home` from anywhere.

---

## How I set it up

```
[Your phone / laptop from anywhere]
	            │
    WireGuard tunnel (UDP 51820)
        	    │
  [DuckDNS -> home public IP]
        	    │
  [Router: port forward 51820]
		- DHCP (192.168.1.100 – 200)
        │
	[Home LAN]
   - Other devices
   - RasberryPi (Static IP  192.168.1.253) 
      - WireGuard server  (10.0.0.1) 
      - DNS server        (*Static Mapping)
	 ...
```

### For someone with CGNAT Router
CGNAT prevent the home router to have permanent public IP which Wireguard client needs.<br>
But there are a good reason why it has to be that way [here](https://www.a10networks.com/glossary/what-is-carrier-grade-nat-cgn-cgnat/)<br>

To bypass CGNAT: 
- Option A: &nbsp; Buy Static IP and use DuckDNS
- Option B: &nbsp; If they offer free DDNS service
  1. Call ISP to enable it
  2. Receive Domain name and ports that should map directly to router ip  
  3. Use that instead of DuckDNS <br>
  
  In my case the service is called `THDDNS`
    ```
        WireGuard tunnel (UDP 51820)
              │
      [AIS THDDNS -> home public IP]
              │
      [Router: port forward 5540]
    ```

## Setup Guide

### 1. Set RPi Static IP
I set it as 192.168.253/24, you set what is needed for
```bash
sudo nmcli con mod "Wired connection 1" ipv4.address 192.168.1.253/24

sudo nmcli con mode "Wired connection 1" ipv4.gateway 192.168.1.1

sudo nmcli con mode "Wired connection 1" ipv4.dns 127.0.0.1
# Pi itself act as DNS so we fill loopback ip

sudo reboot
```

### 2. If home router IP has no domain name yet

1. Sign in at [duckdns.org](https://www.duckdns.org) 
2. Create a subdomain (e.g., `myhome.duckdns.org`)
3. Keep your token from the dashboard
4. Add cron job script to keep IP updated:

```bash
crontab -e
# Add to cron — runs every 5 minutes
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=YOUR_SUBDOMAIN&token=YOUR_TOKEN&ip=" > ~/duckdns.log
```

---

### 3. Router setting 
#### Port Forwarding

In your router admin panel, forward:

| Protocol | External Port | Internal IP | Internal Port |
|----------|--------------|-------------|---------------|
| UDP | 51820 | 192.168.1.253 | 51820 |

In case of CGNAT Router

| Protocol | External Port | Internal IP | Internal Port |
|----------|--------------|-------------|---------------|
| UDP | 5540 (ISP given port) | 192.168.1.253 | 51820 |

#### DHCP Reservation
Usually this setting is in -> Network -> LAN -> DHCP , there should be setting panel for DHCP reservation or binding
- Set mypcEthernet -> 192.168.1.120
- Set mylaptopWifi -> 192.168.1.111
<br>....

---

### 4. WireGuard -- Server (RPi)
Install and setup Wireguard
```bash
sudo apt update && sudo apt install wireguard -y

# add wireguard network interface first
ip link add dev wg0 type wireguard

# Generate server keypair
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Create config file of this name
sudo nano /etc/wireguard/wg0.conf
```

Write the following in `wg0.conf`
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
Automate wireguard process 
```bash
# Start wireguard as systemd service
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### 5. WireGuard -- Client

Generate keypair for each client device:
```bash
wg genkey | tee client.key | wg pubkey > client.pub
```


Write client config
(Can be copy and import into WireGuard app):
```ini
[Interface]
PrivateKey = <contents of client.key>
Address = 10.0.0.2/24
DNS = 10.0.0.1 # tunnel ip for Rpi
[Peer]
PublicKey = <contents of server.pub>
Endpoint = YOUR_SUBDOMAIN.duckdns.org:51820
# For anyone using ISP Local DDNS, please use their domain name instead.
  # For my case:
  # Endpoint = YOUR_SUBDOMAIN.thddns.net:5540
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

```
fill static DNS entries
```bash
interface=eth0
interface=wg0
# Example
address=/mypc.home/192.168.1.120
address=/laptop.home/192.168.1.110
address=/esp32.home/192.168.1.115
address=/rpi.home/192.168.1.253

# Forward everything else to upstream DNS
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

There is it adding more client.
## Adding More Clients

Repeat step 5 for each device (phone, laptop, etc.). Each gets a unique keypair and a unique VPN IP (`10.0.0.x`).


## Done, to use the VPN

- Open wireguard app
- Connect WireGuard profile → inside home LAN
- `ssh user@mypc.home` works

#### In case the client have no GUI
- [Install Wireguard in Terminal & Quickstart](https://www.wireguard.com/quickstart/#key-generation)
- Write a client config file by yourself, here's the standard way to write it: [Samples client.conf](https://gist.github.com/lanceliao/5d2977f417f34dda0e3d63ac7e217fd6)