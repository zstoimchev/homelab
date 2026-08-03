# Homelab

A practical homelab for learning and applying system administration, networking, virtualization, self-hosting, and cybersecurity concepts.

The environment is built around **pfSense**, a smart managed switch, and a **Proxmox VE** server. Services are separated from the main LAN using VLANs, private remote access is provided through **Tailscale**, and selected applications are published through a centralized **Cloudflare Tunnel** and **Caddy** reverse proxy.

> [!IMPORTANT]
> This repository documents the architecture and configuration decisions without publishing passwords, API tokens, tunnel credentials, private keys, application data, or backup archives.

## Goals

The homelab is used to:

- practise Linux and server administration;
- learn routing, firewalling, VLANs, DNS, and remote access;
- self-host useful applications;
- build isolated blue-team and red-team environments;
- test monitoring, backup, and recovery procedures;
- document infrastructure changes as the environment develops.


The Cloudflare Tunnel is initiated from inside the network. No public IPv4 address, WAN port forwarding, or direct inbound firewall opening is required for the published applications.

## Network Overview

| Network | Purpose | Status |
|---|---|---|
| Main LAN | Trusted personal devices and infrastructure management | Active |
| SERVICES VLAN 30 | Internal and externally published server applications | Active |
| STORAGE | NAS, backups, and storage-oriented services | Planned |
| DMZ | Public-facing services with stronger isolation | Planned |
| BLUE LAB | Defensive-security tools, monitoring, and analysis systems | Planned |
| RED LAB | Intentionally vulnerable machines and offensive-security testing | Planned |

Current addressing:

| Component | Address | Role |
|---|---:|---|
| pfSense | `10.12.137.1` | Main gateway and firewall |
| Managed switch | `10.12.137.2` | VLAN-aware network switch |
| Proxmox VE | `10.12.137.3` | Virtualization host |
| SERVICES gateway | `10.12.30.1` | Gateway for VLAN 30 |
| Pi-hole | `10.12.30.3` | DNS filtering and local DNS |
| Reverse proxy | `10.12.30.4` | Caddy and Cloudflare Tunnel |
| Vaultwarden | `10.12.30.6` | Password manager |
| Nextcloud | `10.12.30.7` | File synchronization and collaboration |

## Physical Topology

```text
ISP
 |
 v
pfSense
 |
 v
TP-Link managed switch
 |-- trusted client devices
 `-- Proxmox VE trunk
      |-- SERVICES VLAN 30
      |-- STORAGE        planned
      |-- DMZ            planned
      |-- BLUE LAB       planned
      `-- RED LAB        planned
```

The Proxmox host uses a VLAN-aware bridge. Virtual machines and containers receive the appropriate VLAN tag through their virtual network interface.

## Services

| Service | Access | Purpose | Documentation |
|---|---|---|---|
| Pi-hole | Internal | Network-wide DNS filtering and local DNS records | [View documentation](pihole/README.md) |
| Caddy + Cloudflare Tunnel | Infrastructure | Central ingress and hostname-based reverse proxy | [View documentation](reverse-proxy/README.md) |
| Vaultwarden | Public HTTPS and private network | Self-hosted password manager | [View documentation](vaultwarden/README.md) |
| Nextcloud AIO | Public HTTPS and private network | File synchronization, sharing, and collaboration | [View documentation](nextcloud/README.md) |
| Tailscale | Private | Remote access to the homelab and advertised LAN routes | Documentation planned |
| pfSense | Internal | Routing, firewalling, DHCP, VLANs, and VPN integration | Documentation planned |
| Proxmox VE | Internal | Virtual machine and container hosting | Documentation planned |
| Personal website | Public HTTPS | Personal portfolio and project showcase | Managed separately |
| YouTrack | Public redirect | Redirect to the externally hosted YouTrack instance | Not self-hosted |

## Public Service Flow

```text
User
 |
 v
Cloudflare
 |
 v
Cloudflare Tunnel
 |
 v
cloudflared on 10.12.30.4
 |
 v
Caddy on 127.0.0.1:8080
 |-- Vaultwarden -> 10.12.30.6:8000
 `-- Nextcloud  -> 10.12.30.7:11000
```

Caddy listens on loopback and receives requests only from the local `cloudflared` connector. It selects the correct backend using the original hostname.

## DNS

Pi-hole provides DNS filtering for clients and internal services.

```text
Clients
 |
 v
Pi-hole at 10.12.30.3
 |
 v
Cloudflare malware-blocking resolver at 1.1.1.3
```

Pi-hole is an internal service and is not published through the reverse proxy. Firewall rules permit the required client networks to reach TCP and UDP port `53` on the Pi-hole server.

## Remote Access

Tailscale provides private remote access to the home network without exposing administrative interfaces publicly.

It is used for access to services such as:

- pfSense;
- Proxmox VE;
- Pi-hole administration;
- Nextcloud AIO administration;
- server SSH sessions;
- other internal management interfaces.

Administrative interfaces should remain available only from trusted networks or through Tailscale.

## Security Principles

The homelab follows these general rules:

- no direct WAN port forwarding for hosted applications;
- public applications are exposed through the centralized Cloudflare Tunnel;
- administrative interfaces remain private;
- services are separated from trusted clients using VLANs;
- firewall rules allow only required traffic between networks;
- backend application ports should be limited to the reverse-proxy host where practical;
- public registrations are disabled after initial setup when they are not needed;
- operating systems and applications are updated regularly;
- application-aware backups are kept outside the source VM;
- secrets and application data are never committed to this repository.

## Repository Structure

```text
homelab/
├── README.md
├── nextcloud/
│   └── README.md
├── pihole/
│   └── README.md
├── reverse-proxy/
│   └── README.md
└── vaultwarden/
    └── README.md
```

Each service directory contains deployment notes, configuration decisions, verification commands, update procedures, troubleshooting guidance, and security considerations.

## Planned Work

The next stages of the homelab include:

- document the pfSense and Proxmox configuration;
- deploy Zabbix for infrastructure monitoring;
- use Grafana for dashboards and historical visualization;
- publish a status page for selected services;
- create a dedicated storage and backup design;
- build isolated blue-team and red-team laboratory networks;
- add a small DMZ for intentionally public services;
- deploy Immich after the Proxmox host receives more memory;
- establish and test recovery procedures for critical services.

## Backup Approach

A complete backup strategy should combine:

1. application-native backups;
2. Proxmox guest backups;
3. copies stored outside the original VM and disk;
4. encrypted off-site copies for critical data;
5. periodic restore testing.

A snapshot alone is not considered a complete backup.

## Documentation Policy

This repository records enough information to explain how the homelab is designed and operated, but sensitive values are deliberately excluded.

The following must never be committed:

```text
.env files
passwords and recovery codes
Cloudflare Tunnel tokens
API tokens
private keys and certificates
Vaultwarden data
Nextcloud user data
database dumps containing personal data
backup encryption passphrases
backup archives
production configuration exports containing secrets
```

Example hostnames may be used in documentation where publishing the real hostname is unnecessary.

## Status

The homelab is actively developed. Its current focus is reliable self-hosting, network segmentation, secure remote access, service documentation, and preparation for centralized monitoring.
