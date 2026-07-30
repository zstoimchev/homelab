# Nextcloud AIO behind the Central Reverse Proxy

This document records the Nextcloud AIO installation on a dedicated Debian VM and its connection to the central Caddy and Cloudflare Tunnel VM.

## Deployment details

```text
Nextcloud VM:       10.12.30.7
AIO administration: https://10.12.30.7:8080
Public hostname:    https://nextcloud.example.com
Reverse proxy:      10.12.30.4
AIO Apache backend: http://10.12.30.7:11000
```

```text
Internet
   -> Cloudflare Tunnel on 10.12.30.4
   -> Caddy on 10.12.30.4
   -> Nextcloud AIO Apache on 10.12.30.7:11000
```

The AIO administration interface on port `8080` is separate from the user-facing Nextcloud application on port `11000`.

## 1. Prepare the Debian VM

Reserve `10.12.30.7` in pfSense and place the VM on the SERVICES VLAN.

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent ca-certificates curl nano
sudo systemctl start qemu-guest-agent
```

Enable **QEMU Guest Agent** in the Proxmox VM options. Debian may report that the service is static if `systemctl enable` is used; starting it is sufficient when the QEMU guest-agent device is available.

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

The commands in this document use `sudo docker ...`; the user is intentionally not added to the `docker` group.

## 3. Create the Nextcloud AIO Compose file

```bash
mkdir -p ~/nextcloud
cd ~/nextcloud
nano docker-compose.yaml
```

Use:

```yaml
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    container_name: nextcloud-aio-mastercontainer
    init: true
    restart: always

    ports:
      - "8080:8080"

    environment:
      APACHE_PORT: "11000"
      APACHE_IP_BINDING: "0.0.0.0"
      SKIP_DOMAIN_VALIDATION: "true"

    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro

    stop_grace_period: 30s

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer
```

Why these settings:

- `8080` exposes only the AIO administration interface.
- `APACHE_PORT=11000` creates the user-facing AIO Apache backend.
- `APACHE_IP_BINDING=0.0.0.0` is required because Caddy connects from another VM using `10.12.30.7` rather than localhost.
- `SKIP_DOMAIN_VALIDATION=true` is required for this Cloudflare Tunnel/reverse-proxy deployment.
- Ports `80` and `8443` are not published because AIO's built-in public HTTPS path is not used.

Validate and start the master container:

```bash
cd ~/nextcloud
sudo docker compose config
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker logs nextcloud-aio-mastercontainer --tail 100
```

Never use `docker compose down -v`; deleting the AIO volume removes persistent AIO configuration.

## 4. Complete the AIO installation

Open:

```text
https://10.12.30.7:8080
```

Accept the local certificate warning and enter:

```text
nextcloud.example.com
```

At the time of this installation the following optional components were selected:

```text
Latest stable Nextcloud Hub:  Enabled
Imaginary previews:           Enabled
Office suite:                 Disabled
ClamAV:                       Disabled
Full-text search:             Disabled
Nextcloud Talk:               Disabled
Talk recording:               Disabled
Docker Socket Proxy:          Disabled
HaRP:                         Disabled
Whiteboard:                   Disabled
```

For a future rebuild, select the latest stable Hub release shown by AIO; the marketing name and version will change over time.

Start the containers from the AIO interface and wait until they are healthy.

Useful checks:

```bash
sudo docker ps
sudo docker port nextcloud-aio-apache
sudo ss -lntp | grep ':11000'
```

The expected Apache binding is:

```text
11000/tcp -> 0.0.0.0:11000
```

## 5. Add the reverse proxy to Nextcloud trusted proxies

Inspect the existing entries:

```bash
sudo docker exec --user www-data nextcloud-aio-nextcloud php occ config:system:get trusted_proxies
```

Add the reverse-proxy VM using an unused array index. Index `2` was used in this installation:

```bash
sudo docker exec --user www-data nextcloud-aio-nextcloud php occ config:system:set trusted_proxies 2 \
  --value="10.12.30.4"
```

Verify:

```bash
sudo docker exec --user www-data nextcloud-aio-nextcloud php occ config:system:get trusted_proxies
```

## 6. Test the backend from the reverse-proxy VM

On `10.12.30.4`:

```bash
curl -I -H 'Host: nextcloud.example.com' http://10.12.30.7:11000
```

A valid HTTP response proves that Caddy can reach AIO. Then test Caddy locally:

```bash
curl -I -H 'Host: nextcloud.example.com' http://127.0.0.1:8080
```

The Caddy and Cloudflare configuration is documented in [`../reverse-proxy/README.md`](../reverse-proxy/README.md).

## 7. Cloudflare route

The central tunnel on `10.12.30.4` has this published application:

```text
Public hostname: nextcloud.example.com
Service type:    HTTP
Service URL:     http://127.0.0.1:8080
```

Leave the Cloudflare HTTP Host Header override empty so Caddy receives `Host: nextcloud.example.com`.

After the public route works, access Nextcloud at:

```text
https://nextcloud.example.com
```

Do not publish the AIO administration interface (`10.12.30.7:8080`) through Cloudflare.

## Migrating an existing AIO installation from localhost binding

Changing the master-container environment does not automatically recreate the already existing `nextcloud-aio-apache` container. If Apache still shows:

```text
127.0.0.1:11000->11000/tcp
```

perform this controlled recreation:

1. Confirm the master container has the correct environment:

```bash
sudo docker inspect nextcloud-aio-mastercontainer \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E '^APACHE_(PORT|IP_BINDING)'
```

Expected:

```text
APACHE_PORT=11000
APACHE_IP_BINDING=0.0.0.0
```

2. In `https://10.12.30.7:8080`, click **Stop containers**.
3. Remove only the old Apache container:

```bash
sudo docker rm nextcloud-aio-apache
```

4. Do not remove any volumes or other AIO containers.
5. Return to the AIO interface and click **Start containers**.
6. Verify the recreated binding:

```bash
sudo docker port nextcloud-aio-apache
sudo ss -lntp | grep ':11000'
```

This preserves Nextcloud data and recreates only the stateless frontend container with the correct port binding.

## Removing the old per-VM tunnel

This installation originally had `cloudflared` directly on the Nextcloud VM. After confirming that Nextcloud works through the central reverse proxy:

```bash
sudo systemctl disable --now cloudflared
systemctl is-active cloudflared
```

The result should be `inactive`. Remove the old Cloudflare route and ensure the `nextcloud` DNS record points to the same central tunnel UUID as `vault`.

## User administration commands

List users:

```bash
sudo docker exec --user www-data nextcloud-aio-nextcloud \
  php occ user:list
```

The format is:

```text
- LOGIN_NAME: Display Name
```

Reset a password:

```bash
sudo docker exec -it --user www-data nextcloud-aio-nextcloud \
  php occ user:resetpassword "LOGIN_NAME"
```

Create a user with a clean login name and separate display name:

```bash
sudo docker exec -it --user www-data nextcloud-aio-nextcloud \
  php occ user:add \
  --display-name="Zhivko Stoimchev" \
  example
```

## Backup

Backups can be configured later from the AIO administration interface:

```text
https://10.12.30.7:8080
```

Use the **Backup and restore** section to create an encrypted Borg backup. Prefer a different disk, NAS, or remote destination, and save the Borg passphrase outside the Nextcloud VM. A backup stored only on the same VM disk does not protect against disk or host failure.

Do not rely only on a VM snapshot. A Proxmox backup is useful as an additional recovery layer, not as the only application backup.

## Updating

Use the AIO administration interface to update the managed Nextcloud containers. Do not manually replace individual AIO application containers during normal updates.

The master image can be refreshed with:

```bash
cd ~/nextcloud
sudo docker compose pull
sudo docker compose up -d
```

Update Debian and Docker packages normally:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Create a backup before important upgrades.

## Troubleshooting

### Reverse proxy cannot connect to port 11000

On the Nextcloud VM:

```bash
sudo docker port nextcloud-aio-apache
sudo ss -lntp | grep ':11000'
```

If the binding is `127.0.0.1`, follow the Apache-container recreation procedure above.

### Cloudflare error 1033

The DNS record likely points to a tunnel without a healthy connector. If Vaultwarden works but Nextcloud does not, compare both CNAME targets in Cloudflare DNS. They must point to the same central tunnel UUID.

### Cloudflare 502 or Caddy 502

Test each hop:

```bash
# On the reverse-proxy VM
curl -I -H 'Host: nextcloud.example.com' http://10.12.30.7:11000
curl -I -H 'Host: nextcloud.example.com' http://127.0.0.1:8080
```

### Login redirects or incorrect client IPs

Verify that `10.12.30.4` is present in `trusted_proxies` and that Caddy sends `X-Forwarded-Proto: https`.

## Security notes

- Keep the AIO admin interface on port `8080` limited to trusted LAN/Tailscale access.
- Restrict TCP `11000` so only the reverse-proxy VM can access it when practical.
- Do not commit AIO passwords, user passwords, Borg passphrases, Docker volumes, or application data.
- Disable Cloudflare Rocket Loader and avoid `Cache Everything` rules for the Nextcloud hostname.
- Cloudflare Tunnel can impose request-size and timeout limits; test large uploads before depending on it for large-file workflows.

## Official references

- [Nextcloud AIO repository](https://github.com/nextcloud/all-in-one)
- [Nextcloud AIO reverse proxy documentation](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- [Nextcloud AIO Compose example](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml)
- [Docker Engine installation on Debian](https://docs.docker.com/engine/install/debian/)
- [Docker Compose installation](https://docs.docker.com/compose/install/)
