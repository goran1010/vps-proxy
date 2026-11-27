# vps-proxy

Caddy for every app on the VPS: one reverse proxy that holds ports 80 and 443, gets each domain's HTTPS certificate, and forwards each domain to its app over a shared Docker network called `web`.

## First-time setup

On the VPS, as root:

```bash
git clone https://github.com/goran1010/vps-proxy.git /root/vps-proxy
docker network create web
cd /root/vps-proxy && docker compose up -d
```

Create `web` before starting any app: every app declares it as external, and Compose does not create it.

## The `web` network

- Only each app's public service joins `web`, under an alias prefixed with the app's name, such as `atlas-server`. Caddy proxies to aliases, never to service names, since several apps may call their service `server`
- Databases and other internal services stay on the app's own network. A container on two networks also looks up short names such as `db` on `web`, so another app's `db` there could answer instead of its own
- No stack makes `web` its default network (`name: web` under `networks.default`), which would put all of its services on it

## Adding an app

1. Point the app's domain at the VPS with an A record, and wait until `dig +short <domain>` returns the VPS's IP: Caddy requests the certificate as soon as it reloads
2. In the app's compose file, add its public service to `web` under its own alias:

   ```yaml
   services:
     server:
       networks:
         default:
         web:
           aliases:
             - <app>-server

   networks:
     web:
       external: true
   ```

3. Add a block to `caddy/Caddyfile`:

   ```text
   <domain> {
   	reverse_proxy <app>-server:<port>
   }
   ```

4. Commit and push, then reload Caddy on the VPS

## Reloading Caddy

After a change to `caddy/Caddyfile`:

```bash
cd /root/vps-proxy
git pull
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

If the new config is invalid, the reload fails with the error and Caddy keeps running the old one.

## Logs

Access logs are off. Caddy logs its own messages: startup, reloads, certificates, and errors such as a 502 while an app is down.

```bash
cd /root/vps-proxy
docker compose logs -f caddy           # follow live
docker compose logs --tail 200 caddy   # the last 200 lines
```

## `COMPOSE_FILE`

An app that runs with more than one compose file sets them in its own `.env`, for example:

```text
COMPOSE_FILE=docker-compose.yml:docker-compose.prod.yml
```

Compose reads `.env` from the directory it runs in, so the setting stays in that app's directory. Never export `COMPOSE_FILE` in the shell profile: it would apply in every directory, so `docker compose` here would act on that app instead of Caddy, and `docker compose down` would stop the app.

## VPS notes

### Swap

Hetzner's images come without swap, so when memory runs out the kernel kills a process right away, often a database. A 2 GB swap file, used only under memory pressure:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swappiness.conf
sysctl -p /etc/sysctl.d/99-swappiness.conf
```

### Hostname

Hetzner's images run cloud-init, which sets the hostname and rewrites `/etc/hosts` on every boot from the server's name in the Cloud Console. To rename the server:

1. Rename it in the Cloud Console
2. Check that the metadata service returns the new name: `curl -s http://169.254.169.254/hetzner/v1/metadata/hostname`
3. Run `hostnamectl set-hostname <name>`, and replace the old name on the `127.0.1.1` line in `/etc/hosts`

If the metadata still returns the old name, stop cloud-init from managing both before step 3:

```bash
printf 'preserve_hostname: true\nmanage_etc_hosts: false\n' > /etc/cloud/cloud.cfg.d/99-hostname.cfg
```
