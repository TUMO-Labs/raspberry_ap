# Raspberry Pi 3B+ — Wi-Fi Hotspot Setup 

Configure Raspberry Pi as a Wi-Fi Access Point (AP) while simultaneously connected to a home router.

- **AP interface:** `uap0` (virtual, based on `wlan0`)
- **Client interface:** `wlan0` → home router `xxx`
- **AP SSID:** `raspberry_kr`
- **AP subnet:** `192.168.50.0/24`
- **DHCP range:** `192.168.50.10 – 192.168.50.100`
- **Network Manager:** manages `wlan0` and `uap0`
- **dnsmasq:** system instance, handles DHCP + DNS for AP clients
- **NAT:** iptables MASQUERADE — AP clients reach the internet via `wlan0`

---

## Boot Order

```
wlan0 up
  → create_uap0.service   (creates uap0 virtual interface)
    → NetworkManager      (PiHotspot manual, assigns 192.168.50.1/24)
      → uap0-nat.service  (iptables NAT + ip_forward)
        → dnsmasq.service (DHCP + DNS for AP clients)
```

---

## 1. Install Packages

```bash
sudo apt update
sudo apt install -y network-manager #  may already be installed on some Raspberry Pi OS images
```

---

## 2. File: `/etc/systemd/system/create_uap0.service`

Creates the virtual AP interface `uap0` before NetworkManager starts.

```bash
sudo nano /etc/systemd/system/create_uap0.service
```

```ini
[Unit]
Description=Create uap0 interface
After=sys-subsystem-net-devices-wlan0.device

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/sbin/iw phy phy0 interface add uap0 type __ap
ExecStartPost=/sbin/ip link set uap0 up
ExecStop=/sbin/iw dev uap0 del

[Install]
WantedBy=multi-user.target
```

---

## 3. File: `/etc/systemd/system/uap0-nat.service`

Configures IP forwarding and iptables NAT so AP clients can reach the internet.

```bash
sudo nano /etc/systemd/system/uap0-nat.service
```

```ini
[Unit]
Description=uap0 NAT and forwarding
After=NetworkManager.service create_uap0.service network-online.target
Requires=create_uap0.service
Before=dnsmasq.service

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/sbin/sysctl -w net.ipv4.ip_forward=1
ExecStart=/sbin/iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
ExecStart=/sbin/iptables -A FORWARD -i uap0 -o wlan0 -j ACCEPT
ExecStart=/sbin/iptables -A FORWARD -i wlan0 -o uap0 -m state --state RELATED,ESTABLISHED -j ACCEPT
ExecStop=/sbin/iptables -t nat -D POSTROUTING -o wlan0 -j MASQUERADE
ExecStop=/sbin/iptables -D FORWARD -i uap0 -o wlan0 -j ACCEPT
ExecStop=/sbin/iptables -D FORWARD -i wlan0 -o uap0 -m state --state RELATED,ESTABLISHED -j ACCEPT

[Install]
WantedBy=multi-user.target
```

---

## 4. File: `/etc/NetworkManager/NetworkManager.conf`

```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```

```ini
[main]
plugins=ifupdown,keyfile

[ifupdown]
managed=false

[device]
wifi.scan-rand-mac-address=no
```

> ⚠️ Do **not** add `dns=dnsmasq` — DNS is handled by the system dnsmasq, not NM.

---

## 5. File: `/etc/dnsmasq.conf`

Install `dnsmasq` only after the AP interface and services are defined, then stop and disable its default startup until the custom config is ready.

```bash
sudo apt install -y dnsmasq
sudo systemctl stop dnsmasq
sudo systemctl disable dnsmasq
```

```bash
sudo nano /etc/dnsmasq.conf
```

```
interface=uap0
bind-dynamic
listen-address=192.168.50.1

dhcp-authoritative
dhcp-range=192.168.50.10,192.168.50.100,12h
dhcp-option=option:router,192.168.50.1
dhcp-option=option:dns-server,192.168.50.1

domain=iot.local
address=/iot.local/192.168.50.1

no-resolv
server=1.1.1.1
server=8.8.8.8
```

---

## 6. Enable Services

```bash
sudo systemctl daemon-reload
sudo systemctl enable create_uap0.service uap0-nat.service dnsmasq.service
sudo systemctl restart NetworkManager
```

---

## 7. Create AP Connection in NetworkManager

```bash
# Start uap0 first
sudo systemctl start create_uap0.service

# Create connection
sudo nmcli connection add \
    type wifi \
    ifname uap0 \
    con-name PiHotspot \
    ssid raspberry_kr \
    -- \
    wifi.mode ap \
    wifi.band bg \
    wifi.channel 8 \
    wifi-sec.key-mgmt wpa-psk \
    wifi-sec.proto rsn \
    wifi-sec.group ccmp \
    wifi-sec.pairwise ccmp \
    wifi-sec.pmf 1 \
    wifi-sec.psk "1234567890" \
    ipv4.method manual \
    ipv4.addresses "192.168.50.1/24" \
    ipv6.method disabled

# Bring up the AP
sudo nmcli connection up PiHotspot
```

---

## 8. Start NAT and dnsmasq

```bash
sudo systemctl start uap0-nat.service
sudo systemctl start dnsmasq
```

---

## 9. Verification

```bash
# Status of all services
sudo systemctl status create_uap0 uap0-nat dnsmasq NetworkManager

# IP addresses on interfaces
ip addr show uap0    # should be 192.168.50.1/24
ip addr show wlan0   # should be IP from home router

# DHCP listening on port 67
sudo ss -ulnp | grep :67

# NAT rules
sudo iptables -t nat -L POSTROUTING -n -v

# Active connections
nmcli connection show --active

# Leases after connecting a phone
cat /var/lib/misc/dnsmasq.leases

# dnsmasq logs in real time
sudo journalctl -u dnsmasq -f
```

---

## 10. Verify After Reboot

```bash
sudo reboot

# After reboot:
nmcli connection show --active
sudo systemctl status create_uap0 uap0-nat dnsmasq
ip addr show uap0
```

---

## Troubleshooting

| Problem | Diagnostic Command |
|---|---|
| uap0 not created | `sudo systemctl status create_uap0` |
| AP not visible in Wi-Fi list | `sudo iw dev` — verify uap0 is in AP mode |
| Client not getting IP | `sudo journalctl -u dnsmasq -n 50` |
| No internet on client | `sudo iptables -t nat -L -n -v` |
| dnsmasq fails to start | `sudo dnsmasq --test` — validate config |
| Port 67 conflict | `sudo ss -ulnp \| grep :67` — kill conflicting process |

---

## File Structure

```
/etc/
├── dnsmasq.conf                              # DHCP + DNS config
├── NetworkManager/
│   ├── NetworkManager.conf                   # NM config (no dns=dnsmasq)
│   └── system-connections/
│       └── PiHotspot.nmconnection            # AP connection (auto-created by nmcli)
└── systemd/system/
    ├── create_uap0.service                   # Creates virtual uap0 interface
    └── uap0-nat.service                      # iptables NAT
```
