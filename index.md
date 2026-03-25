---
title: Home
layout: home
nav_order: 1
---

# Running OpenWrt in an Unprivileged LXC Container on Arch Linux
{: .no_toc }

This guide walks through running OpenWrt as an unprivileged LXC container on Arch Linux, with a LAN bridge interface giving the container a presence on LAN (it's own IP), with LuCI accessible from any device on the LAN.

**Target setup:**
- Host: Arch Linux, NetworkManager, firewalld
- Container: OpenWrt (x86/64 snapshot) managed by a dedicated unprivileged user
- Networking: `lxcbr0` for container management/outbound, `br0` (bridged physical NIC) for LAN access
- LuCI accessible at a static LAN IP from any device on the network

---

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

- Arch Linux installed with NetworkManager and firewalld
- `sudo` access
- Physical access to the machine (some network steps will briefly drop connectivity)

Install LXC:

```bash
sudo pacman -S lxc lxcfs
```

Run the sanity check and verify all items are `enabled`:

```bash
lxc-checkconfig
```

Fix `newuidmap` and `newgidmap` if flagged as not setuid-root:

```bash
sudo chmod u+s /usr/bin/newuidmap
sudo chmod u+s /usr/bin/newgidmap
```

---

## 1. Create a Dedicated User

Create a dedicated user to own and manage the container:

```bash
sudo useradd -m -s /bin/bash lxwrt
sudo passwd lxwrt
sudo loginctl enable-linger lxwrt
```

`enable-linger` ensures systemd maintains a session for `lxwrt` at boot, creating `/run/user/<uid>` automatically without requiring an interactive login.

---

## 2. Configure Unprivileged LXC

Add subuid and subgid mappings for `lxwrt`:

```bash
echo "lxwrt:100000:65536" | sudo tee -a /etc/subuid
echo "lxwrt:100000:65536" | sudo tee -a /etc/subgid
```

Allow `lxwrt` to create veth interfaces on both bridges (more on `br0` later):

```bash
echo "lxwrt veth lxcbr0 10" | sudo tee -a /etc/lxc/lxc-usernet
echo "lxwrt veth br0 10" | sudo tee -a /etc/lxc/lxc-usernet
```

Switch to `lxwrt` and create the default container config:

```bash
su - lxwrt

mkdir -p ~/.config/lxc
cat > ~/.config/lxc/default.conf << 'EOF'
lxc.idmap = u 0 100000 65536
lxc.idmap = g 0 100000 65536
lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
EOF
```

Add the required environment variables to `.bashrc` so they're always set when managing containers interactively:

```bash
cat >> ~/.bashrc << 'EOF'

# LXC unprivileged container requirements
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus
EOF

exit
```

---

## 3. Set Up the Host Bridge (`lxcbr0`)

`lxcbr0` is a NAT bridge managed by `lxc-net` that provides the container with outbound internet access and a management IP. Create its config:

```bash
sudo tee /etc/default/lxc-net << 'EOF'
USE_LXC_BRIDGE="true"
LXC_BRIDGE="lxcbr0"
LXC_ADDR="10.0.3.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.3.0/24"
LXC_DHCP_RANGE="10.0.3.2,10.0.3.254"
LXC_DHCP_MAX="253"
EOF

sudo systemctl enable --now lxc-net
```

Verify the bridge is up:

```bash
ip link show lxcbr0
```

Add `lxcbr0` to firewalld's trusted zone so DHCP traffic flows freely:

```bash
sudo firewall-cmd --zone=trusted --add-interface=lxcbr0 --permanent
sudo firewall-cmd --reload
```

---

## 4. Create the LAN Bridge (`br0`)

This bridges your physical NIC so the container can appear as a real device on your LAN.

{: .warning }
Do this from a **local terminal**, not SSH — your connection will briefly drop when the IP moves from the physical NIC to the bridge.

First check your current wired connection name and physical NIC:

```bash
nmcli connection show
ip addr show
```

Substitute `enp1s0f0` with your actual NIC name and `192.168.2.53` with your host's current LAN IP throughout the following commands.

Create the bridge and configure the static IP:

```bash
sudo nmcli connection add type bridge ifname br0 con-name br0
sudo nmcli connection modify br0 ipv4.addresses '192.168.2.53/24'
sudo nmcli connection modify br0 ipv4.gateway '192.168.2.1'
sudo nmcli connection modify br0 ipv4.dns '192.168.2.53'
sudo nmcli connection modify br0 ipv4.method manual
sudo nmcli connection modify br0 bridge.stp no
```

Add the physical NIC as a bridge slave:

```bash
sudo nmcli connection add type bridge-slave ifname enp1s0f0 master br0 con-name br0-slave-enp1s0f0
```

Cut over (run as a single chained command to minimise downtime):

```bash
sudo nmcli connection down "Wired connection 1" && \
sudo nmcli connection up br0-slave-enp1s0f0 && \
sudo nmcli connection up br0
```

Verify:

```bash
ip addr show br0
ping -c3 1.1.1.1
```

---

## 5. Create the Container

Export the required environment variables, then create the container using the LXC download template:

```bash
su - lxwrt
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus

lxc-create -n openwrt -t download -- -d openwrt -a amd64
```

This will present an interactive list of available OpenWrt releases. Select the snapshot or stable release you prefer.

Fix directory ACLs so the container's mapped root can traverse `lxwrt`'s home (run as your personal user with sudo):

```bash
exit

sudo setfacl -m u:100000:x /home/lxwrt
sudo setfacl -m u:100000:x /home/lxwrt/.local
sudo setfacl -m u:100000:x /home/lxwrt/.local/share
sudo setfacl -m u:100000:x /home/lxwrt/.local/share/lxc
sudo setfacl -m u:100000:x /home/lxwrt/.local/share/lxc/openwrt
```

---

## 6. Configure the Container

Add the LAN veth interface and autostart flag to the container config:

```bash
su - lxwrt

cat >> ~/.local/share/lxc/openwrt/config << 'EOF'
lxc.net.1.type = veth
lxc.net.1.link = br0
lxc.net.1.flags = up
lxc.net.1.name = eth1
lxc.start.auto = 1
EOF
```

---

## 7. Configure Container Autostart

Create a systemd user service that starts LXC containers marked with `lxc.start.auto = 1` at boot:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/lxc-autostart.service << 'EOF'
[Unit]
Description=LXC autostart containers
After=network.target lxc-net.service

[Service]
Type=oneshot
RemainAfterExit=yes
Environment=XDG_RUNTIME_DIR=/run/user/%U
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/%U/bus
ExecStart=/usr/bin/lxc-autostart
ExecStop=/usr/bin/lxc-autostart -s

[Install]
WantedBy=default.target
EOF

systemctl --user enable --now lxc-autostart.service
```

---

## 8. Start the Container and Configure Networking

Start the container:

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus

lxc-start -n openwrt
```

Attach to it:

```bash
lxc-attach -n openwrt -- /bin/ash -l
```

Inside the container, assign a static LAN IP to `eth1` (substitute `192.168.2.59` with your desired LAN IP):

```bash
uci set network.lan=interface
uci set network.lan.ifname='eth1'
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.2.59'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.2.1'
uci commit network
/etc/init.d/network restart
```

Remove the WAN interfaces since the Mac mini is not directly connected to WAN in this setup:

```bash
uci delete network.wan
uci delete network.wan6
uci commit network
/etc/init.d/network restart
```

Verify `eth1` has the static IP:

```bash
ip addr show eth1
```

---

## 9. Install LuCI

Still inside the container, install LuCI with uhttpd:

```bash
apk update
apk add luci
```

{: .note }
OpenWrt snapshots use `apk` instead of `opkg`. If you are on a stable release prior to 24.10, use `opkg install luci` instead.

Enable and start the web server:

```bash
/etc/init.d/uhttpd enable
/etc/init.d/uhttpd start
```

Configure uhttpd to listen on the LAN IP:

```bash
uci set uhttpd.main.listen_http='192.168.2.59:80'
uci commit uhttpd
/etc/init.d/uhttpd restart
```

LuCI is now accessible at `http://192.168.2.59` from any device on your LAN.

---

## 10. Verify Everything Survives a Reboot

Reboot the host and confirm:

```bash
# Bridge has the correct IP
ip addr show br0

# Container is running
sudo -u lxwrt lxc-ls --fancy

# Container has internet access
lxc-attach -n openwrt -- /bin/ash -l
ping -c3 1.1.1.1
```

And verify LuCI is reachable at `http://192.168.2.59` from another LAN device.

---

## Network Architecture Summary

```
                    ┌─────────────────────────────────────┐
                    │           Arch Linux Host            │
                    │                                      │
  LAN 192.168.2.x ──┤ br0 (192.168.2.53)                  │
                    │  └── enp1s0f0 (physical NIC)         │
                    │                                      │
                    │ lxcbr0 (10.0.3.1, NAT)               │
                    │                                      │
                    │  ┌──────────────────────────────┐    │
                    │  │     OpenWrt LXC Container    │    │
                    │  │                              │    │
                    │  │  eth0 ── lxcbr0 (10.0.3.x)  │    │
                    │  │  eth1 ── br0 (192.168.2.59)  │    │
                    │  │                              │    │
                    │  │  LuCI, DHCP server, WireGuard│    │
                    │  └──────────────────────────────┘    │
                    └─────────────────────────────────────┘
```

- `eth0` — management interface, outbound internet for the container (via NAT on `lxcbr0`)
- `eth1` — LAN interface, reachable by all devices on `192.168.2.x`
- The host machine reaches the container via `10.0.3.x` (bridge isolation prevents host↔container traffic over `br0`)
