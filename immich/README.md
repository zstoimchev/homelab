# Immich behind the Central Reverse Proxy

This document records the Immich installation and its connection to the central Caddy and Cloudflare Tunnel VM.

## Deployment details

```text
Immich VM:       10.12.30.8
Public hostname: https://immich.example.com
Backend:         http://10.12.30.8:2283
Reverse proxy:   10.12.30.4
Media directory: /srv/immich
Database:        /var/lib/immich/postgres
```

```text
Internet
   -> Cloudflare Tunnel on 10.12.30.4
   -> Caddy on 10.12.30.4
   -> Immich on 10.12.30.8:2283
```

Immich runs with Docker Compose. The media library is stored under `/srv/immich`, while PostgreSQL data is kept under `/var/lib/immich/postgres`.

## 1. Install Immich

Create the application directory and download the Compose files from the current Immich release:

```bash
sudo mkdir -p /opt/immich
sudo chown "$USER":"$USER" /opt/immich
cd /opt/immich

wget -O docker-compose.yml   https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml

wget -O .env   https://github.com/immich-app/immich/releases/latest/download/example.env
```

Generate a database password:

```bash
openssl rand -hex 24
```

Edit `/opt/immich/.env` and configure the persistent storage locations, timezone, version, and database credentials:

```env
UPLOAD_LOCATION=/srv/immich
DB_DATA_LOCATION=/var/lib/immich/postgres
TZ=Europe/Ljubljana
IMMICH_VERSION=v3

DB_PASSWORD=<GENERATED_PASSWORD>
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

The `.env` file contains the database password and must not be committed to GitHub.

Validate the configuration, pull the images, and start Immich:

```bash
sudo docker compose config
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
```

Wait until the Immich server, PostgreSQL, and Redis containers report healthy.

## 2. Complete the initial configuration

Open Immich locally at:

```text
http://10.12.30.8:2283
```

Create the first account, which becomes the administrator.

The following optional features are currently disabled:

- Machine Learning;
- Google Cast;
- Storage Template Engine.

Machine Learning can be enabled later if more resources are assigned to the VM.

## 3. Configure the reverse proxy

The central Caddy and Cloudflare setup is documented in [`../reverse-proxy/README.md`](../reverse-proxy/README.md).

Add Immich to the existing Caddy `:8080` block:

```caddyfile
@immich host immich.example.com
handle @immich {
    reverse_proxy 10.12.30.8:2283 {
        header_up X-Forwarded-Proto https
    }
}
```

Validate and reload Caddy:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
```

Add another published application to the existing Cloudflare Tunnel:

```text
Public hostname: immich.example.com
Service type:    HTTP
Service URL:     http://127.0.0.1:8080
```

Leave the HTTP Host Header override empty so Caddy receives the original hostname.

After the route is active, Immich is available at:

```text
https://immich.example.com
```

No WAN port forwarding to the Immich VM is required.

## 4. Configure the mobile application

Install the Immich mobile application and use the public URL as the server address:

```text
https://immich.example.com
```

The local address can still be used for testing:

```text
http://10.12.30.8:2283
```

Select the phone albums that should be backed up and enable backup from the Immich application.

## Backup

A complete Immich backup must protect both the media library and the database.

```text
Media:     /srv/immich
Database:  /var/lib/immich/postgres
```

Immich database backups do not contain the actual photos and videos, so the media directory must be backed up separately.

Backups should be copied to another physical disk, NAS, or encrypted remote destination. A Proxmox backup is useful as an additional recovery layer, but should not be the only copy of the media library.

## Updating

Before an important upgrade, verify that backups are available.

Update Immich with:

```bash
cd /opt/immich
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
```

Optionally remove unused old Docker images:

```bash
sudo docker image prune -f
```

Review the Immich release notes before major-version upgrades.

## Troubleshooting

Check container status and recent logs:

```bash
cd /opt/immich
sudo docker compose ps
sudo docker compose logs --tail=100
```

Test the Immich server locally:

```bash
curl http://127.0.0.1:2283/api/server/ping
```

Expected response:

```json
{"res":"pong"}
```

If the public hostname fails, test the backend directly from the reverse-proxy VM:

```bash
curl -I http://10.12.30.8:2283
curl -I -H 'Host: immich.example.com' http://127.0.0.1:8080
```

If large videos upload locally but fail through the public hostname, check Cloudflare upload-size and timeout limits before troubleshooting Immich itself.

## Security notes

- Do not commit `.env`, database passwords, photos, videos, database dumps, or backup archives.
- Do not create a WAN port forward to TCP `2283`.
- Public access is provided through the centralized Cloudflare Tunnel and Caddy reverse proxy.
- Keep Docker and Immich updated.
- Keep an independent backup of the media library outside the Immich VM.

## Official references

- [Immich documentation](https://docs.immich.app/)
- [Immich Docker Compose installation](https://docs.immich.app/install/docker-compose/)
- [Immich reverse proxy documentation](https://docs.immich.app/administration/reverse-proxy/)
- [Immich backup and restore](https://docs.immich.app/administration/backup-and-restore/)
- [Immich mobile backup](https://docs.immich.app/features/mobile-backup/)
