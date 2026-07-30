# Central Reverse Proxy with Caddy and Cloudflare Tunnel

This document records the central ingress setup used for the homelab. One `cloudflared` connector runs on a dedicated reverse-proxy VM, and Caddy routes requests to internal services by hostname.

## Current topology

| Service | Address | Public hostname | Internal backend |
|---|---:|---|---|
| Pi-hole | `10.12.30.3` | — | DNS |
| Reverse proxy | `10.12.30.4` | — | Caddy on `127.0.0.1:8080` |
| Vaultwarden | `10.12.30.6` | `vault.example.com` | `http://10.12.30.6:8000` |
| Nextcloud AIO | `10.12.30.7` | `nextcloud.example.com` | `http://10.12.30.7:11000` |

```text
Internet
   |
   v
Cloudflare
   |
   v  one outbound Cloudflare Tunnel
cloudflared on 10.12.30.4
   |
   v
Caddy on 127.0.0.1:8080
   |-- vault.example.com     -> 10.12.30.6:8000
   `-- nextcloud.example.com -> 10.12.30.7:11000
```

No public IP, WAN port forwarding, or inbound firewall opening is required. The tunnel connector initiates the connection to Cloudflare.

## 1. Prepare the Debian VM

Use a small Debian Server VM on the SERVICES VLAN and reserve its address in pfSense.

```text
Hostname: reverseproxy
Address:  10.12.30.4/24
Gateway:  10.12.30.1
DNS:      10.12.30.3
```

Update the system and install basic tools:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent ca-certificates curl gpg nano
```

Start the QEMU guest agent:

```bash
sudo systemctl start qemu-guest-agent
systemctl is-active qemu-guest-agent
```

On Debian the guest-agent unit can be static, so `systemctl enable` may report that the unit has no installation section. Enable **QEMU Guest Agent** in the Proxmox VM options and perform a full stop/start if the guest device is not present.

## 2. Install Caddy

Install Caddy from its stable Debian repository:

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install -y caddy
```

Check the installation:

```bash
caddy version
systemctl status caddy --no-pager
```

## 3. Configure Caddy

Back up the default file:

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.default
```

Replace `/etc/caddy/Caddyfile` with:

```caddyfile
{
	servers {
		trusted_proxies static 127.0.0.1/32
		trusted_proxies_strict
		client_ip_headers CF-Connecting-IP X-Forwarded-For
	}
}

:8080 {
	bind 127.0.0.1

	@vaultwarden host vault.example.com
	handle @vaultwarden {
		reverse_proxy 10.12.30.6:8000 {
			header_up X-Forwarded-Proto https
		}
	}

	@nextcloud host nextcloud.example.com
	handle @nextcloud {
		reverse_proxy 10.12.30.7:11000 {
			header_up X-Forwarded-Proto https
		}
	}

	handle {
		respond "Unknown hostname" 404
	}
}
```

Why this layout:

- Caddy listens only on `127.0.0.1:8080`; it is not exposed to the VLAN.
- The local `cloudflared` service is the only intended caller.
- Both Cloudflare hostnames point to the same Caddy listener.
- Caddy selects the backend from the original HTTP `Host` header.
- Public HTTPS terminates at Cloudflare, so `X-Forwarded-Proto` is set to `https` for the applications.

Format, validate, and reload:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
sudo systemctl status caddy --no-pager
```

Confirm that Caddy is loopback-only:

```bash
sudo ss -lntp | grep ':8080'
```

Expected listener:

```text
127.0.0.1:8080
```

## 4. Test the internal backends

Before involving Cloudflare, confirm that the reverse-proxy VM can reach both application VMs:

```bash
curl -I -H 'Host: vault.example.com' http://10.12.30.6:8000

curl -I -H 'Host: nextcloud.example.com' http://10.12.30.7:11000
```

Then test Caddy's hostname routing locally:

```bash
curl -I -H 'Host: vault.example.com' http://127.0.0.1:8080

curl -I -H 'Host: nextcloud.example.com' http://127.0.0.1:8080
```

Test the fallback route:

```bash
curl -i -H 'Host: invalid.example.com' http://127.0.0.1:8080
```

The fallback should return `404 Not Found`.

## 5. Install cloudflared

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings

curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update
sudo apt install -y cloudflared
cloudflared --version
```

## 6. Create the central Cloudflare Tunnel

In Cloudflare:

1. Open **Networking -> Tunnels**.
2. Create a tunnel named `homelab-reverse-proxy`.
3. Choose Linux.
4. Copy the generated installation command.
5. Run the command on `10.12.30.4`. It resembles:

```bash
sudo cloudflared service install YOUR_TUNNEL_TOKEN
```

The token is a secret. Do not commit it to GitHub, paste it into notes, or include it in screenshots.

Verify the connector:

```bash
sudo systemctl is-enabled cloudflared
sudo systemctl is-active cloudflared
sudo systemctl status cloudflared --no-pager
sudo journalctl -u cloudflared -n 50 --no-pager
```

The Cloudflare dashboard should report the tunnel as **Healthy**.

## 7. Add the published application routes

Inside `homelab-reverse-proxy`, create these two routes:

| Public hostname | Service type | Service URL |
|---|---|---|
| `vault.example.com` | HTTP | `http://127.0.0.1:8080` |
| `nextcloud.example.com` | HTTP | `http://127.0.0.1:8080` |

Leave any **HTTP Host Header override** empty. Caddy needs the original hostname to select the correct backend.

Cloudflare creates proxied DNS records for the routes. Both records should point to the same central tunnel target:

```text
vault      CNAME  <CENTRAL-TUNNEL-UUID>.cfargotunnel.com
nextcloud  CNAME  <CENTRAL-TUNNEL-UUID>.cfargotunnel.com
```

## 8. Verify public access

```bash
curl -I https://vault.example.com
curl -I https://nextcloud.example.com
```

Also test both services with their browser and native clients.

## Adding another service later

1. Make the service reachable from `10.12.30.4` on a private backend port.
2. Add another hostname matcher and `reverse_proxy` block to the Caddyfile.
3. Validate and reload Caddy.
4. Add a Cloudflare published application route pointing the hostname to `http://127.0.0.1:8080`.
5. Leave the Host Header override blank.

Example:

```caddyfile
@grafana host grafana.example.com
handle @grafana {
	reverse_proxy 10.12.30.20:3000 {
		header_up X-Forwarded-Proto https
	}
}
```

## Updating

```bash
sudo apt update
sudo apt full-upgrade -y
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl restart caddy
sudo systemctl restart cloudflared
```

## Troubleshooting

### Cloudflare error 1033

Cloudflare cannot find a healthy connector for the tunnel referenced by the hostname.

Check:

```bash
sudo systemctl is-active cloudflared
sudo journalctl -u cloudflared -n 100 --no-pager
```

If one hostname works and another shows 1033, inspect the DNS records. The failing hostname may still point to an old tunnel UUID. Delete the stale route/DNS record and recreate the route under the central tunnel.

### Cloudflare 502 Bad Gateway

The tunnel is connected, but `cloudflared` or Caddy cannot reach the configured origin.

```bash
curl -I http://127.0.0.1:8080
curl -I -H 'Host: vault.example.com' http://127.0.0.1:8080
curl -I -H 'Host: nextcloud.example.com' http://127.0.0.1:8080
sudo journalctl -u caddy -n 100 --no-pager
sudo journalctl -u cloudflared -n 100 --no-pager
```

### Caddy returns 404

The request reached Caddy, but the `Host` header did not match a configured hostname. Confirm that the Cloudflare route has no Host Header override and that the hostname in the Caddyfile is exact.

### Backend connection refused

Test the application directly from the reverse-proxy VM. The service must bind to its own VM address, not only to `127.0.0.1`.

## Security notes

- Do not expose Caddy's port `8080` to the LAN or Internet; keep it on loopback.
- Do not create WAN port forwards for these services.
- Restrict `10.12.30.6:8000` and `10.12.30.7:11000` so only `10.12.30.4` can reach them when practical.
- Since these VMs are on the same VLAN, use a guest firewall or Proxmox firewall for same-VLAN restrictions.
- Do not commit Cloudflare tokens, passwords, private keys, application data, or backup passphrases.

## Official references

- [Caddy installation](https://caddyserver.com/docs/install)
- [Caddy reverse_proxy directive](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Caddy trusted proxies options](https://caddyserver.com/docs/caddyfile/options#trusted-proxies)
- [Cloudflare Tunnel dashboard setup](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)
- [Cloudflare published applications](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/routing-to-tunnel/)
- [Cloudflare Tunnel common errors](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/troubleshoot-tunnels/common-errors/)
