# Vaultwarden behind the Central Reverse Proxy

This document records the Vaultwarden installation on a dedicated Debian VM and its connection to the central Caddy and Cloudflare Tunnel VM.

## Deployment details

```text
Vaultwarden VM:  10.12.30.6
Public hostname: https://vault.zstoimchev.com
Backend:         http://10.12.30.6:8000
Reverse proxy:   10.12.30.4
Data directory:  ~/vaultwarden/data
```

```text
Internet
   -> Cloudflare Tunnel on 10.12.30.4
   -> Caddy on 10.12.30.4
   -> Vaultwarden on 10.12.30.6:8000
```

## 1. Prepare the Debian VM

Reserve `10.12.30.6` in pfSense and place the VM on the SERVICES VLAN.

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent ca-certificates curl nano
sudo systemctl start qemu-guest-agent
```

Enable **QEMU Guest Agent** in the Proxmox VM options. Debian may report that the unit is static if `systemctl enable` is used.

## 2. Install Docker Engine and Docker Compose

Remove conflicting packages if present:

```bash
sudo apt remove -y docker.io docker-compose docker-doc podman-docker containerd runc
```

Add Docker's official repository:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF_DOCKER
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF_DOCKER

sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Verify:

```bash
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world
sudo docker compose version
```

Use `sudo docker ...` unless the security implications of adding a user to the `docker` group are deliberately accepted.

## 3. Create the Vaultwarden Compose file

```bash
mkdir -p ~/vaultwarden
cd ~/vaultwarden
nano compose.yaml
```

Initial configuration:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped

    environment:
      DOMAIN: "https://vault.example.com"
      SIGNUPS_ALLOWED: "true"

    volumes:
      - ./data:/data

    ports:
      - "10.12.30.6:8000:80"
```

Important details:

- `DOMAIN` must use the final external HTTPS URL.
- The `data` directory contains the persistent database, keys, attachments, and other state.
- The host binding must use the Vaultwarden VM's own address, `10.12.30.6`.
- Do not bind the container to the reverse-proxy VM's address (`10.12.30.4`); an operating system cannot bind a socket to an address that it does not own.
- Sign-ups are enabled only long enough to create the first account.

Validate and start:

```bash
cd ~/vaultwarden
sudo docker compose config
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker logs vaultwarden --tail 100
```

Wait for the health status to become healthy.

Verify the listener:

```bash
sudo ss -lntp | grep ':8000'
curl -I http://10.12.30.6:8000
```

Expected Docker port mapping:

```text
10.12.30.6:8000->80/tcp
```

## 4. Test from the reverse-proxy VM

On `10.12.30.4`:

```bash
curl -I -H 'Host: vault.example.com' http://10.12.30.6:8000
```

Then test through Caddy:

```bash
curl -I -H 'Host: vault.example.com' http://127.0.0.1:8080
```

The Caddy and Cloudflare setup is documented in [`../reverse-proxy/README.md`](../reverse-proxy/README.md).

## 5. Cloudflare route

The central tunnel on `10.12.30.4` has this published application:

```text
Public hostname: vault.example.com
Service type:    HTTP
Service URL:     http://127.0.0.1:8080
```

Leave the Cloudflare HTTP Host Header override empty so Caddy receives `Host: vault.example.com`.

Open:

```text
https://vault.example.com
```

## 6. Create the first account

Create the account only through the public HTTPS URL. Choose a unique, strong master password and keep an offline emergency copy. Vaultwarden cannot recover a forgotten master password.

After login, create a harmless test item and confirm that it can be read again.

## 7. Disable public registration

Edit the Compose file:

```bash
cd ~/vaultwarden
nano compose.yaml
```

Change:

```yaml
SIGNUPS_ALLOWED: "true"
```

to:

```yaml
SIGNUPS_ALLOWED: "false"
```

Apply the change:

```bash
sudo docker compose up -d
sudo docker exec vaultwarden printenv SIGNUPS_ALLOWED
```

Expected output:

```text
false
```

Check the site in a private browser window and confirm that public account creation is no longer available.

This installation does not configure `ADMIN_TOKEN`, so the Vaultwarden `/admin` panel remains disabled. That is appropriate for a simple single-user instance unless the panel is specifically needed later.

## 8. Configure clients

Install the official Bitwarden clients and choose the self-hosted environment:

```text
Server URL: https://vault.example.com
```

Configure this URL before signing in on:

- Fedora browser extension or desktop client
- iPhone Bitwarden app
- Samsung tablet Bitwarden app

Create one test login on one device, synchronize, and confirm that it appears on the others.

## 9. Enable two-step login

In the web vault:

```text
Settings -> Security -> Two-step login
```

Enable an authenticator or hardware security key. Store the recovery code outside Vaultwarden. Do not keep the only copy of the Vaultwarden second factor inside the same vault.

## Backup

All persistent application state is under:

```text
~/vaultwarden/data
```

Create an online SQLite database backup:

```bash
sudo docker exec vaultwarden /vaultwarden backup
```

A complete recovery copy must also preserve the rest of the `data` directory, including attachments and cryptographic keys. For a simple consistent archive:

```bash
cd ~/vaultwarden
sudo docker compose stop
sudo tar -czf "/tmp/vaultwarden-full-$(date +%F-%H%M).tar.gz" compose.yaml data
sudo docker compose start
```

Copy the archive to another physical disk, NAS, or encrypted off-site destination. Do not keep the only backup inside the Vaultwarden VM.

A Proxmox backup is useful as an additional layer, but it should not be the only Vaultwarden backup. Test restoration periodically.

## Updating

Create a backup first, then:

```bash
cd ~/vaultwarden
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker logs vaultwarden --tail 100
```

Optionally remove unused old images:

```bash
sudo docker image prune -f
```

Update Debian normally:

```bash
sudo apt update
sudo apt full-upgrade -y
```

## Troubleshooting

### `cannot assign requested address`

Example:

```text
failed to bind host port 10.12.30.4:8000/tcp: cannot assign requested address
```

The Compose file is trying to bind to an address owned by another VM. On the Vaultwarden VM the mapping must be:

```yaml
ports:
  - "10.12.30.6:8000:80"
```

### Permission denied on `/var/run/docker.sock`

Use `sudo`:

```bash
sudo docker compose ps
sudo docker compose up -d
```

The normal account is not a member of the privileged `docker` group.

### Reverse proxy gets connection refused

On the Vaultwarden VM:

```bash
sudo docker compose ps
sudo ss -lntp | grep ':8000'
curl -I http://10.12.30.6:8000
```

On the reverse-proxy VM:

```bash
curl -I http://10.12.30.6:8000
```

### Web vault loads but clients cannot log in

Confirm that the clients use the exact self-hosted URL:

```text
https://vault.example.com
```

Also confirm that Cloudflare Access has not been placed in front of Vaultwarden, because an additional browser-login layer can interfere with native clients.

## Security notes

- Disable public sign-ups after the account is created.
- Use a unique master password and enable two-step login.
- Keep the Cloudflare route public only through HTTPS; do not forward WAN ports to the VM.
- Restrict TCP `8000` so only `10.12.30.4` can reach it when practical.
- Do not commit the `data` directory, master passwords, recovery codes, tunnel tokens, or backup archives to GitHub.
- Avoid Cloudflare cache rules for `vault.example.com`.

## Official references

- [Vaultwarden repository](https://github.com/dani-garcia/vaultwarden)
- [Vaultwarden Docker Compose guide](https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose)
- [Disabling Vaultwarden registration](https://github.com/dani-garcia/vaultwarden/wiki/Disable-registration-of-new-users)
- [Vaultwarden backup guide](https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault)
- [Docker Engine installation on Debian](https://docs.docker.com/engine/install/debian/)
- [Docker Compose installation](https://docs.docker.com/compose/install/)
