# Home-Lab Stack (Pi-hole • Unbound • Traefik • Cloudflared • Prometheus • Grafana)

## What You Need to Know

**Why this stack matters:**\
You get secure remote access, private DNS, ad‑blocking, observability,
and a production‑grade reverse‑proxy---all running locally on a Pi or
Linux server.

**What this README gives you:**
- Clear setup steps
- How `.env` works
- How Cloudflare tunnels + origin certs integrate
- How every container fits together
- Maintenance + troubleshooting
- A complete section of the most useful Docker commands you will ever
need

**Bottom line:**
Follow this document and you'll have a hardened, monitored,
cloud‑tunneled infrastructure running in about 20 minutes.

------------------------------------------------------------------------

# Full Documentation

## 1. Overview

This Docker stack provides a private, secure, and observable home‑lab
environment featuring:

-   **Pi‑hole** --- DNS sinkholing + ad‑blocking\
-   **Unbound** --- Private recursive DNS resolver\
-   **Traefik** --- Reverse proxy + TLS termination\
-   **Cloudflared** --- Zero‑trust secure remote access (no open
    ports!)\
-   **Prometheus** --- Metrics collection\
-   **Grafana** --- Dashboards for your entire network\
-   **Exporters** --- Pi-hole stats + system metrics

Everything lives inside the Docker network `mon`, except Node Exporter
which runs on the host.

------------------------------------------------------------------------

## 2. Prerequisites

-   Linux host or Raspberry Pi (arm64 recommended)
-   Docker & Docker Compose v2+
-   Cloudflare account + domain\
-   Cloudflare tunnel created (JSON creds + cert.pem)
-   Cloudflare Origin Certificates stored on host:
    -   `/etc/ssl/<domain>/origin-cert.pem`
    -   `/etc/ssl/<domain>/origin-key.pem`

------------------------------------------------------------------------

## 3. Directory Layout

Recommended folder structure:

    .
    ├── docker-compose.yml
    ├── .env
    ├── env_template
    ├── traefik/
    │   ├── traefik.yml
    │   └── dynamic/
    │       └── *.yml
    ├── cloudflared/
    │   ├── config.yml
    │   ├── <TUNNEL-UUID>.json
    │   └── cert.pem
    ├── unbound/
    │   └── unbound.conf (optional)
    ├── etc-pihole/
    ├── etc-dnsmasq.d/
    ├── prometheus/
    │   └── prometheus.yml
    └── grafana/
        └── provisioning/
            ├── datasources/
            └── dashboards/

------------------------------------------------------------------------

## 4. Creating the `.env` File

Copy the template:

``` bash
cp env_template .env
nano .env
```

Variables to define:

  | Variable | Purpose |
  | - | - |
  | `CLOUDFLARED_CERTS_DOMAIN` | Domain used for origin certs |
  | `CLOUDFLARED_DNS` | Pi-hole hostname via Cloudflare |
  | `CLOUDFLARED_GRAFANA` | Grafana hostname |
  | `PIHOLE_PASSWORD` | Pi-hole admin password |
  | `GRAFANA_USER` / `GRAFANA_PASSWORD` | Grafana admin creds |

------------------------------------------------------------------------

## 5. Cloudflared Setup

Create a tunnel:

``` bash
cloudflared tunnel login
cloudflared tunnel create <NAME>
```

This produces:

-   `<TUNNEL-UUID>.json`
-   `cert.pem`

Copy them:

``` bash
cp ~/.cloudflared/<TUNNEL-UUID>.json ./cloudflared/
cp ~/.cloudflared/cert.pem ./cloudflared/
```

Example `config.yml`:

``` yaml
tunnel: <TUNNEL-UUID>
credentials-file: /etc/cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: ${CLOUDFLARED_DNS}
    service: https://traefik:443
  - hostname: ${CLOUDFLARED_GRAFANA}
    service: https://traefik:443
  - service: http_status:404
```

------------------------------------------------------------------------

## 6. Cloudflare Origin Certificates

On the host (not in repo):

``` bash
sudo mkdir -p /etc/ssl/${CLOUDFLARED_CERTS_DOMAIN}
sudo cp origin-cert.pem origin-key.pem /etc/ssl/${CLOUDFLARED_CERTS_DOMAIN}/
```

Traefik mounts them as:

    /certs/origin-cert.pem
    /certs/origin-key.pem

------------------------------------------------------------------------

## 7. Service Overview

### Traefik

-   TLS termination\
-   Dashboard enabled\
-   Private bind: `127.0.0.1:80`, `127.0.0.1:443`\
-   Receives traffic only from Cloudflared
-   **Traefik is currently broken**

### Cloudflared

-   Runs tunnel
-   Exposes Grafana + Pi-hole securely

### Unbound

-   Local recursive DNS
-   Pi-hole upstream

### Pi-hole

-   Exposes DNS on port 53
-   Web UI routed through Traefik

### Prometheus

-   Stores metrics
-   Scrapes node exporter + pihole exporter

### Node Exporter

-   Exposes host metrics at `:9100`

### Pi-hole Exporter

-   Exposes stats for Prometheus

### Grafana

-   Provisioned datasources/dashboards
-   Routed through Traefik

------------------------------------------------------------------------

## 8. Running the Stack

Start everything:

``` bash
docker compose up -d
```

Check status:

``` bash
docker compose ps
```

Stop:

``` bash
docker compose down
```

Stop + remove volumes:

``` bash
docker compose down -v
```

------------------------------------------------------------------------

## 9. Troubleshooting

### Port 53 in use

Disable systemd-resolved or uninstall native Pi-hole.

### Cloudflared can't connect

-   Wrong tunnel UUID
-   Missing `<TUNNEL-UUID>.json`
-   Wrong DNS CNAME targets

### Traefik TLS issues

-   Origin cert/key path
-   File permissions
-   Wrong `CLOUDFLARED_CERTS_DOMAIN`

------------------------------------------------------------------------

# 10. Extremely Useful Docker Commands

## 🔍 Logs

**Follow logs for a single container:**

``` bash
docker logs -f <container>
```

**Follow logs with timestamps:**

``` bash
docker logs -ft <container>
```

------------------------------------------------------------------------

## 🐚 Enter a container shell

**Bash:**

``` bash
docker exec -it <container> bash
```

**Sh (lightweight images):**

``` bash
docker exec -it <container> sh
```

------------------------------------------------------------------------

## ▶️ Run a script inside a container

``` bash
docker exec -it <container> /path/to/script.sh
```

------------------------------------------------------------------------

## 🧹 Clean orphans

``` bash
docker compose down --remove-orphans
```

------------------------------------------------------------------------

## 🧼 Cleanup (safe)

**Unused images:**

``` bash
docker image prune
```

**Unused volumes (safe):**

``` bash
docker volume prune
```

**Everything unused (aggressive):**

``` bash
docker system prune -a
```

------------------------------------------------------------------------

## 📦 Inspect containers / networks / volumes

**Full container inspection:**

``` bash
docker inspect <container>
```

**List networks:**

``` bash
docker network ls
```

**Inspect a network:**

``` bash
docker network inspect <network>
```

------------------------------------------------------------------------

## ♻️ Rebuild & restart a service

``` bash
docker compose up -d --force-recreate --build <service>
```

------------------------------------------------------------------------

## 🔄 Restart a container

``` bash
docker restart <container>
```

------------------------------------------------------------------------

## 📁 View container file structure

``` bash
docker exec -it <container> ls -al /
```

------------------------------------------------------------------------

## 🧪 Test DNS resolution inside containers

From Pi-hole:

``` bash
docker exec -it pihole dig google.com
```

From Unbound:

``` bash
docker exec -it unbound dig @127.0.0.1 -p 5335 cloudflare.com
```

------------------------------------------------------------------------

# 11. Backups

### Pi-hole

    etc-pihole/
    etc-dnsmasq.d/

### Prometheus

    prom-data volume

### Grafana

    grafana-data volume

Backup by:

``` bash
docker run --rm -v grafana-data:/data -v $(pwd):/backup alpine tar czvf /backup/grafana.tar.gz /data
```

------------------------------------------------------------------------

# 12. Final Notes

This stack is designed to be:

-   Secure
-   Observable
-   Zero-trust capable
-   Easy to maintain
-   Perfect for Raspberry Pi homelabs

Extend it easily with TailScale, Loki, Alertmanager, exporters, and
more.

Enjoy your fortress. 🛡️🔥
