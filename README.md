# BreezeRMM_LocalDeploy

Notes and procedures for running a **self-hosted [Breeze RMM](https://github.com/lanternops/breeze)** instance on a local network (homelab / LAN), accessed by IP address with HTTPS.

> ⚠️ Unofficial community notes. Not affiliated with or endorsed by the Breeze RMM project. Test in a lab before using in production.

## Problem

A default Breeze install binds everything to `localhost`. As a result:

- the web console only works on the server itself,
- other LAN devices get `SSL_ERROR_INTERNAL_ERROR_ALERT` / `Could not create SSL/TLS secure channel`,
- the Windows agent installer fails with:
  `could not fetch release integrity metadata from https://localhost - refusing to install without a trusted checksum`

## Solution (summary)

1. Point Breeze's public URLs in `.env` at the server's LAN IP.
2. Configure Caddy to serve the IP with its internal CA (`CADDY_LOCAL_CERTS=local_certs`) and add `default_sni <IP>`.
3. Export Caddy's root CA and import it into **Trusted Root** on each Windows endpoint.
4. Copy a fresh installer command from `https://<BREEZE_IP>` and run it in an elevated PowerShell.

## Documentation

| Procedure | Description |
|---|---|
| [LAN access & Windows agent install fix](docs/lan-access-fix.md) | Full step-by-step server and endpoint procedure, troubleshooting and rollback |

## Environment

| Component | Value |
|---|---|
| Host OS | Ubuntu (VM on Proxmox) |
| Deployment | Docker Compose, `/opt/breeze` |
| Reverse proxy | Caddy (bundled) |
| Endpoints | Windows 10/11 |

## Security notes

- Never commit `.env`, certificates, or enrollment secrets to this repo.
- Rotate the agent enrollment secret if an installer command is ever shared or screenshotted.
- Use a static IP / DHCP reservation for the Breeze server.

## Contributing

Issues and pull requests with corrections or additional procedures are welcome.
