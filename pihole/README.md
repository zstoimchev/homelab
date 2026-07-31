# Pi-hole DNS Filtering on Proxmox

This document records the Pi-hole deployment used in the homelab. Pi-hole provides network-wide DNS filtering for clients and forwards allowed DNS queries to Cloudflare's malware-blocking resolver at `1.1.1.3`.

## Current deployment

| Component | Address | Purpose |
|---|---:|---|
| pfSense gateway | `10.12.30.1` | Gateway and DHCP for the SERVICES VLAN |
| Pi-hole | `10.12.30.3` | DNS filtering and local DNS service |
| Upstream DNS | `1.1.1.3` | Cloudflare malware-blocking DNS resolver |
| Network | `10.12.30.0/24` | SERVICES VLAN, VLAN ID `30` |

```text
Clients and servers
       |
       v
Pi-hole — 10.12.30.3:53
       |
       v
Cloudflare DNS — 1.1.1.3:53
```

Pi-hole is an internal service. It is not exposed through the public reverse proxy or Cloudflare Tunnel.

## 1. Prepare the Proxmox guest

A small Debian LXC container is sufficient for Pi-hole. A VM also works if stronger isolation is preferred.

Suggested resources:

```text
Hostname: pihole
CPU:      1 core
Memory:   256–512 MB
Disk:     4–8 GB
Network:  SERVICES VLAN, tag 30
Address:  10.12.30.3/24
Gateway:  10.12.30.1
```

Reserve `10.12.30.3` in pfSense or configure it statically inside the guest. The address must not change because all clients use it as their DNS server.

Verify the network configuration:

```bash
ip -br address
ip route
ping -c 4 10.12.30.1
ping -c 4 1.1.1.3
```

Expected address:

```text
10.12.30.3/24
```

## 2. Update Debian

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y curl ca-certificates
```

If the container is administered as `root`, omit `sudo`.

## 3. Install Pi-hole

Run the official installation script:

```bash
curl -sSL https://install.pi-hole.net | bash
```

During the interactive installer, use the following values:

```text
Network interface:     the active interface, usually eth0
Static address:        10.12.30.3/24
Gateway:               10.12.30.1
Upstream DNS provider: Custom
Custom upstream DNS:   1.1.1.3
Web interface:         enabled
Query logging:         enabled unless there is a specific privacy reason to disable it
Blocklist:             keep the default list initially
```

Record the generated web-interface password securely. It can be changed later with:

```bash
sudo pihole setpassword
```

Depending on the installed Pi-hole version, the password command may also be available as:

```bash
sudo pihole -a -p
```

## 4. Open the administration interface

From a trusted LAN device, open:

```text
http://10.12.30.3/admin
```

The administration interface should remain private. Do not publish it through Cloudflare Tunnel and do not create a WAN port-forward for it.

## 5. Configure the upstream resolver

In the Pi-hole administration interface, open the DNS settings and ensure that the configured upstream resolver is:

```text
1.1.1.3
```

Remove other upstream resolvers if all allowed DNS queries should use this resolver exclusively.

`1.1.1.3` is the selected upstream resolver for this homelab because it provides DNS resolution with malware-domain filtering.

## 6. Advertise Pi-hole through pfSense DHCP

Pi-hole only filters clients that actually use it for DNS. In pfSense, configure the DHCP server for each network that should use Pi-hole.

For the SERVICES VLAN:

```text
Services -> DHCP Server -> SERVICES
DNS server 1: 10.12.30.3
```

Leave the additional DNS-server fields empty unless another intentional internal DNS resolver is available. Adding a public resolver such as `1.1.1.1` as a secondary client DNS server would allow clients to bypass Pi-hole.

Save and apply the DHCP configuration.

Renew the lease on a client or reconnect it to the network. On Linux, check the received DNS server with:

```bash
resolvectl status
```

The client should show:

```text
10.12.30.3
```

## 7. Configure firewall access

Clients that should use Pi-hole must be allowed to reach:

```text
Destination: 10.12.30.3
Protocol:    TCP and UDP
Port:        53
```

Pi-hole must be allowed to reach its upstream resolver:

```text
Source:      10.12.30.3
Destination: 1.1.1.3
Protocol:    TCP and UDP
Port:        53
```

It also needs normal outbound access for operating-system and blocklist updates, including DNS, HTTP/HTTPS, and time synchronization.

When clients are on another VLAN, add the appropriate inter-VLAN rule in pfSense. The rule source must match the client network that is making the DNS request.

Do not expose TCP or UDP port `53` from the WAN. An Internet-accessible open DNS resolver can be abused for DNS amplification attacks.

## 8. Verify DNS resolution

From a Linux client, test Pi-hole directly:

```bash
dig @10.12.30.3 example.com
```

or:

```bash
nslookup example.com 10.12.30.3
```

The response should identify `10.12.30.3` as the DNS server.

Test the normal client configuration:

```bash
dig example.com
```

Then open the Pi-hole dashboard and confirm that the query appears in the query log.

## 9. Verify the Pi-hole service

On the Pi-hole guest:

```bash
sudo pihole status
sudo systemctl status pihole-FTL --no-pager
sudo ss -lntup | grep ':53'
```

Expected state:

```text
Pi-hole blocking: enabled
pihole-FTL:       active
DNS listener:     TCP/UDP port 53
```

## 10. Local DNS records

Pi-hole can also provide readable names for internal systems.

Example records:

```text
pihole.home.arpa        -> 10.12.30.3
reverseproxy.home.arpa -> 10.12.30.4
vaultwarden.home.arpa  -> 10.12.30.6
nextcloud.home.arpa    -> 10.12.30.7
```

Add these under the local DNS section in the Pi-hole administration interface.

Use an internal suffix such as `home.arpa` rather than inventing a public top-level domain. Public service names such as `nextcloud.example.com` continue to be handled through Cloudflare and the central reverse proxy.

## 11. Updating Pi-hole

Update Pi-hole itself with:

```bash
sudo pihole -up
```

Update the operating system separately:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Refresh the gravity database manually when required:

```bash
sudo pihole -g
```

Check the service after an update:

```bash
sudo pihole status
sudo systemctl is-active pihole-FTL
```

## 12. Backup and restore

Use Pi-hole's Teleporter feature before major changes:

```text
Settings -> Teleporter -> Backup
```

Store the exported archive outside the Pi-hole guest, preferably in the homelab backup location or another machine.

A complete recovery plan should include:

```text
Pi-hole Teleporter export
Proxmox backup of the guest
A copy of this README
pfSense DHCP and firewall configuration backup
```

To restore, install Pi-hole on a new guest, assign it `10.12.30.3`, and import the Teleporter archive.

## 13. Useful troubleshooting commands

Show the address and route:

```bash
ip -br address
ip route
```

Check Pi-hole status:

```bash
sudo pihole status
sudo systemctl status pihole-FTL --no-pager
```

Follow Pi-hole service logs:

```bash
sudo journalctl -u pihole-FTL -f
```

Test the configured upstream resolver:

```bash
dig @1.1.1.3 example.com
```

Test Pi-hole locally:

```bash
dig @127.0.0.1 example.com
```

Test Pi-hole from another machine:

```bash
dig @10.12.30.3 example.com
```

Check whether port `53` is listening:

```bash
sudo ss -lntup | grep ':53'
```

## 14. Common problems

### Clients can access the Internet but queries do not appear in Pi-hole

Check the DNS server received through DHCP:

```bash
resolvectl status
```

It must point to `10.12.30.3`. Renew the DHCP lease after changing pfSense.

### Clients on another VLAN cannot resolve names

Confirm that pfSense allows TCP and UDP port `53` from the client VLAN to `10.12.30.3`.

Also check that Pi-hole is configured to listen on the required interfaces and accept requests from the intended networks.

### Pi-hole can resolve locally but clients cannot reach it

Check the guest firewall, Proxmox firewall, and pfSense rules. Test basic reachability first:

```bash
ping 10.12.30.3
```

Then test DNS explicitly:

```bash
dig @10.12.30.3 example.com
```

### DNS fails when Pi-hole is stopped

This is expected when `10.12.30.3` is the only advertised DNS server. The design prevents clients from silently bypassing Pi-hole, but it also means Pi-hole availability is important.

Possible future improvements include a second Pi-hole instance on another host or a monitored recovery procedure.

## 15. Security notes

- Keep the administration interface accessible only from trusted networks or through Tailscale.
- Do not expose DNS port `53` to the Internet.
- Do not publish the Pi-hole admin page through the public reverse proxy.
- Keep Debian and Pi-hole updated.
- Use a strong administration password.
- Back up the configuration before upgrades or major blocklist changes.
- Avoid adding excessive blocklists without reviewing false positives and maintenance impact.

## 16. Monitoring plan

When Zabbix is deployed, monitor at least:

```text
Host availability
CPU and memory usage
Disk usage
pihole-FTL service state
TCP/UDP port 53 availability
DNS response time
Resolution through 10.12.30.3
Reachability of upstream resolver 1.1.1.3
Age of the latest backup
```

The Pi-hole web dashboard can be used for DNS-specific statistics, while Zabbix and Grafana can provide longer-term infrastructure monitoring.

## Final state

```text
Pi-hole address:       10.12.30.3
Network:               SERVICES VLAN 30
Gateway:               10.12.30.1
Upstream resolver:     1.1.1.3
Client DNS via DHCP:   10.12.30.3
Administration page:  http://10.12.30.3/admin
Public exposure:       none
WAN port forwarding:  none
```
