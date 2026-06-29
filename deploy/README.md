# Deployment — api.b1-crm.com

Odoo 19 is deployed on the server `178.105.45.4` and served at
**https://api.b1-crm.com**.

## Architecture

```
Internet ──HTTPS──> nginx (api.b1-crm.com, Let's Encrypt)
                      │  reverse proxy
                      ▼
              Odoo 19 container  (127.0.0.1:8069, docker compose)
                      │
                      ▼
        host PostgreSQL 16  (127.0.0.1:5432, role "odoo", db "odoo")
```

- **Container stack:** `/opt/odoo/docker-compose.yml` (image `odoo:19`, `network_mode: host`,
  bound to `127.0.0.1:8069`, restart `unless-stopped`).
- **Config:** `/opt/odoo/config/odoo.conf` (`proxy_mode=on`, `list_db=off`, `dbfilter=^odoo$`).
- **Secrets:** `/opt/odoo/.env` (DB password, master password, admin password) — chmod 600.
- **Custom addons:** `/opt/odoo/addons` ← mirrored from this repo's `addons_custom/` by CI.
- **DB:** reuses the host's native PostgreSQL; dedicated role `odoo` with `CREATEDB`.

## CI/CD

GitHub Actions workflow: [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml).

Trigger: push to `main` changing `addons_custom/**` or `deploy/**`, or manual
**Run workflow** (with an optional module list to upgrade with `-u`).

### Required repository secrets
| Secret | Value |
|---|---|
| `SSH_HOST` | `178.105.45.4` |
| `SSH_USER` | `root` |
| `SSH_PRIVATE_KEY` | the dedicated CI deploy key (`/root/.ssh/ci_deploy` on the server) |

## Common server operations

```bash
cd /opt/odoo
docker compose ps                 # status
docker compose logs -f odoo       # live logs
docker compose restart odoo       # restart
docker compose pull && docker compose up -d   # update Odoo image

# install / upgrade a module manually
docker compose run --rm odoo odoo -c /etc/odoo/odoo.conf -d odoo -u <module> --stop-after-init
```

## Rollback (restore the previous site)

A pre-deploy backup was taken at `/root/backups/pre-odoo-<timestamp>/`
(nginx configs, Postgres globals + `postgres` DB dump). The previous
`api.b1-crm.com` nginx config is also kept at
`/etc/nginx/sites-available/api.b1-crm.com.pre-odoo.bak`.
