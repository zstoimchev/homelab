# pfSense Setup

This document describes the current pfSense setup in my homelab.

## Purpose

pfSense is used as the main router and firewall for the homelab network. It provides LAN routing, DHCP, DNS, firewall rules, and remote access through Tailscale.

## Current Setup

The current network is simple and does not use VLANs yet.

Current main LAN: `10.12.137.0/24`

Main devices:
- pfSense: `10.12.137.1`
- Switch:  `10.12.137.2`
- Proxmox: `10.12.137.3`

## What I Did

### 1. Installed pfSense

I installed pfSense on a small mini PC and connected a monitor/keyboard for the initial setup.

The WAN interface is connected to the upstream internet connection, and the LAN interface is connected to the homelab switch.

### 2. Configured Interfaces

After installation, I configured the interfaces from the pfSense console.

WAN was configured to receive an address from the upstream network.

LAN was configured manually: `LAN IP: 10.12.137.1/24`

No upstream gateway was configured on LAN.

### 3. Enabled DHCP on LAN

I enabled the DHCP server on the LAN interface so devices connected to the switch can automatically receive IP addresses.

DHCP range: `10.12.137.64 - 10.12.137.127`

Important infrastructure devices are kept outside the DHCP range and use static IPs.

### 4. Configured Web Access

After LAN was working, I accessed the pfSense web interface from my laptop using: `https://10.12.137.1`

From there, I continued the remaining configuration through the web UI.

### 5. Configured DNS

pfSense is used as the DNS server for LAN clients.

I prefer Cloudflare DNS instead of Google DNS, so the upstream DNS servers are: `1.1.1.1` and `1.0.0.1`.

### 6. Configured Firewall Rules

For now, the LAN is treated as a trusted network.

LAN clients are allowed to access the internet and other local services.

Tailscale rules are currently permissive while the setup is still being tested. These rules should be restricted later.

### 7. Added Tailscale

I installed and configured Tailscale on pfSense.

pfSense is used as a Tailscale subnet router so I can access the homelab LAN remotely from my laptop.

The exposed LAN route is: `10.12.137.0/24`

On my laptop, I enabled accepting Tailscale routes so I can access LAN IPs wirelessly.

### 8. Connected the Switch

I connected a TP-Link TL-SG108E managed switch to the pfSense LAN interface.

The switch management IP is: `10.12.137.2`

At the moment, VLANs are not configured. All ports are part of the same normal LAN.

### 9. Connected Proxmox

After the network was working, I connected the Proxmox host to the switch.

The Proxmox host is available at: `https://10.12.137.3:8006`

## Current Status

#### Actually done:

- pfSense has internet access
- LAN clients receive IP addresses
- Laptop can access pfSense web UI
- Switch is reachable on the LAN
- Proxmox is reachable on the LAN
- Tailscale can access LAN IPs remotely

#### Yes to be done:

- Add VLANs for management, servers, and pentest lab
- Add a dedicated server/services network
- Add a red-team / pentest lab network
- Add monitoring with Zabbix and Grafana
- Add Pi-hole or pfBlockerNG for DNS filtering
- Restrict Tailscale firewall rules
- Export and securely back up the pfSense configuration
