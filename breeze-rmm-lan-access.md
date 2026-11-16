# Breeze RMM – Fix LAN Access and Windows Agent Install (Self-Hosted, IP-Based)

## Purpose

Make a self-hosted Breeze RMM server (Docker Compose, installed in `/opt/breeze`) reachable from other devices on the LAN over HTTPS by its IP address, so that:

- the web console works from any LAN device, and
- the Windows agent installer command works on endpoints.

## Symptoms this fixes

| Where | Symptom |
|---|---|
| Windows installer | `Breeze: could not fetch release integrity metadata from https://localhost - refusing to install without a trusted checksum` |
| Windows PowerShell | `Invoke-WebRequest : The request was aborted: Could not create SSL/TLS secure channel.` |
| Browser | `Secure Connection Failed` / `SSL_ERROR_INTERNAL_ERROR_ALERT` / `ERR_SSL_PROTOCOL_ERROR` |
| Installer command in console | Contains `--server "https://localhost"` |

## Root cause

1. The default install sets all public URLs to `localhost`. The console builds the agent installer command from these values, so endpoints try to reach `https://localhost`, which is the endpoint itself rather than the Breeze server.
2. Caddy (the TLS reverse proxy) only has a certificate for `localhost`. When a client connects by IP, it sends no SNI (server name), Caddy finds no matching certificate and aborts the TLS handshake with alert 80 (`internal_error`).
3. Caddy's certificates come from its own internal CA, which Windows and Firefox do not trust by default. The installer refuses to continue without a trusted TLS connection.

## Prerequisites

- Root or sudo access on the Breeze server.
- Breeze installed with Docker Compose in `/opt/breeze`.
- A **static IP or DHCP reservation** for the Breeze server. If the IP changes, agents lose the server and the certificate no longer matches.
- Local administrator rights on each Windows endpoint.

> Throughout this procedure, replace `10.10.11.220` with your Breeze server's LAN IP.

---

## Part 1 – Server (Ubuntu)

### 1.1 Find the server's LAN IP

```bash
hostname -I
```

Use the first LAN address (e.g. `10.x.x.x` or `192.168.x.x`). Ignore Docker addresses (`172.17.x.x`, `172.18.x.x`).

### 1.2 Review the current `localhost` references

```bash
cd /opt/breeze
grep -rn "localhost" .env docker-compose*.yml 2>/dev/null
```

**Change** (public-facing):

| Setting | New value |
|---|---|
| `PUBLIC_API_URL` | `https://<IP>` |
| `API_URL` | `https://<IP>` |
| `BREEZE_DOMAIN` | `<IP>` (no scheme) |
| `CORS_ALLOWED_ORIGINS` | `https://<IP>` |
| `TURN_REALM` | `<IP>` |
| `DASHBOARD_URL` | `https://<IP>` |
| `PUBLIC_APP_URL` | `https://<IP>` |

**Do not change** (internal to Docker – `localhost` is correct there): `DATABASE_URL`, `REDIS_URL`, `EMAIL_FROM`, `SMTP_FROM`, `MAILGUN_FROM`.

### 1.3 Update `.env` with the server IP

```bash
cd /opt/breeze
IP=10.10.11.220

sed -i.bak -E \
  -e "s#^(PUBLIC_API_URL|API_URL|DASHBOARD_URL|PUBLIC_APP_URL|CORS_ALLOWED_ORIGINS)=https://localhost\$#\1=https://$IP#" \
  -e "s#^(BREEZE_DOMAIN|TURN_REALM)=localhost\$#\1=$IP#" .env

grep -nE "^(PUBLIC_API_URL|API_URL|BREEZE_DOMAIN|CORS_ALLOWED_ORIGINS|TURN_REALM|DASHBOARD_URL|PUBLIC_APP_URL)=" .env
```

All seven lines should now show the IP. A backup is saved as `.env.bak`.

### 1.4 Configure Caddy for the IP with its internal CA

```bash
cd /opt/breeze
IP=10.10.11.220

# Use Caddy's internal CA (no Let's Encrypt for a private IP)
grep -q '^CADDY_LOCAL_CERTS=' .env \
  && sed -i 's/^CADDY_LOCAL_CERTS=.*/CADDY_LOCAL_CERTS=local_certs/' .env \
  || echo 'CADDY_LOCAL_CERTS=local_certs' >> .env

# Serve the site on the IP
grep -q '^CADDY_SITE_ADDRESS=' .env \
  && sed -i "s#^CADDY_SITE_ADDRESS=.*#CADDY_SITE_ADDRESS=$IP#" .env \
  || echo "CADDY_SITE_ADDRESS=$IP" >> .env

grep -nE "^(CADDY_SITE_ADDRESS|CADDY_LOCAL_CERTS)=" .env
```

### 1.5 Add `default_sni` to the Caddyfile

The Caddyfile used by the container is `docker/Caddyfile.prod` (mounted as `/etc/caddy/Caddyfile`). Confirm with:

```bash
grep -n -i caddyfile docker-compose.yml
```

Add `default_sni` inside the existing **global options** block, directly under the `{$CADDY_LOCAL_CERTS}` placeholder:

```bash
cd /opt/breeze
IP=10.10.11.220
cp docker/Caddyfile.prod docker/Caddyfile.prod.bak
sed -i "s/^  {\$CADDY_LOCAL_CERTS}\$/  {\$CADDY_LOCAL_CERTS}\n  default_sni $IP/" docker/Caddyfile.prod
sed -n 75,80p docker/Caddyfile.prod
```

Expected result:

```
{
  {$CADDY_LOCAL_CERTS}
  default_sni 10.10.11.220
  servers {
  ...
```

> Do **not** add a second `{ ... }` global block – Caddy allows only one, and it must be first in the file.

### 1.6 Restart the stack

```bash
cd /opt/breeze
docker compose down && docker compose up -d
```

If you only changed the Caddyfile later, recreate Caddy (a plain `restart` keeps the old file, because `sed -i` replaces the file and single-file bind mounts keep pointing at the old one):

```bash
docker compose up -d --force-recreate caddy
```

### 1.7 Validate the Caddy config

```bash
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

Expect `Valid configuration`. WARN lines about formatting and "listening only on the HTTP port" come from the validate run itself and can be ignored.

### 1.8 Export Caddy's root CA certificate

```bash
cd /opt/breeze
docker compose exec caddy cat /data/caddy/pki/authorities/local/root.crt > ~/breeze-root.crt
cat ~/breeze-root.crt
```

The output must start with `-----BEGIN CERTIFICATE-----` and end with `-----END CERTIFICATE-----`. As root, the file is saved to `/root/breeze-root.crt`.

### 1.9 Test HTTPS from the server

```bash
curl -v --cacert ~/breeze-root.crt https://10.10.11.220 2>&1 | grep -iE "SSL connection|subjectAltName|alert|HTTP/"
```

Expected:

```
* SSL connection using TLSv1.3 / ...
*  subjectAltName: "10.10.11.220" matches cert's IP address!
< HTTP/2 200
```

---

## Part 2 – Windows endpoint

All commands in this part must run in an **elevated** PowerShell (right-click Start, choose **Terminal (Admin)**, then click **Yes**).

### 2.1 Confirm the session is elevated

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole('Administrator')
```

Must return `True`. Otherwise the certificate import fails with `Access is denied (E_ACCESSDENIED)`.

### 2.2 Copy the root certificate to the endpoint

**Option A – SCP** (if SSH to the server is allowed):

```powershell
scp root@10.10.11.220:/root/breeze-root.crt C:\breeze-root.crt
```

**Option B – Copy/paste:**

1. Copy the full PEM block from `cat ~/breeze-root.crt`, including the BEGIN and END lines.
2. Paste it into Notepad. If Notepad's formatting toolbar (H1, bold) is visible, turn formatting off first.
3. Save as `C:\breeze-root.crt`, with **Save as type: All files** so `.txt` is not appended.

Line breaks inside the PEM block are normal and must be kept.

### 2.3 Verify the file

```powershell
certutil -dump C:\breeze-root.crt
```

Should show certificate details with the Issuer `Caddy Local Authority`.

If the file cannot be found, locate it:

```powershell
Get-ChildItem C:\, $HOME\Downloads, $HOME\Documents, $HOME\Desktop -Filter "breeze-root*" -ErrorAction SilentlyContinue | Select FullName
```

### 2.4 Import into the machine's Trusted Root store

```powershell
Import-Certificate -FilePath C:\breeze-root.crt -CertStoreLocation Cert:\LocalMachine\Root
```

Expected output: thumbprint and `CN=Caddy Local Authority - <year> ECC Root`.

### 2.5 Test connectivity and TLS

```powershell
Test-NetConnection 10.10.11.220 -Port 443
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest https://10.10.11.220 -UseBasicParsing | Select StatusCode
```

Expected: `TcpTestSucceeded : True` and `StatusCode 200`.

### 2.6 Install the agent

1. Open the console at `https://10.10.11.220`, not `localhost`.
2. Copy a **fresh** Windows installer command. Do not reuse an old one.
3. Confirm the command contains `--server "https://10.10.11.220"`.
4. Run it in the elevated PowerShell.
5. Approve the device in the console if it appears as pending.

---

## Part 3 – Browser access to the console

- Always use `https://10.10.11.220`. After this change, `https://localhost` no longer works, even on the server itself (`SSL_ERROR_INTERNAL_ERROR_ALERT`).
- **Edge or Chrome on Windows** use the Windows certificate store, so step 2.4 is enough.
- **Firefox** uses its own certificate store. You have two options:
  - **Quick:** on the warning page, click **Advanced**, then **Accept the Risk and Continue**.
  - **Permanent:** go to **Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import**, select `breeze-root.crt`, tick **Trust this CA to identify websites**, and click OK. Copy the file out of `/root` first if Firefox cannot open it there.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `could not fetch release integrity metadata from https://localhost` | Installer command still points to localhost | Complete Part 1 and copy a fresh command from `https://<IP>` |
| `Could not create SSL/TLS secure channel` (Windows) or `SSL_ERROR_INTERNAL_ERROR_ALERT` (browser) | Caddy has no certificate for the requested name or IP | Check `CADDY_SITE_ADDRESS`, `CADDY_LOCAL_CERTS` and `default_sni`, then run `docker compose up -d --force-recreate caddy` |
| `Could not establish trust relationship` / certificate warning | Root CA not trusted on the client | Steps 2.2 to 2.4 (or the Firefox import) |
| `Import-Certificate : Access is denied` | PowerShell not elevated | Open Terminal (Admin) and check with step 2.1 |
| `Import-Certificate : The certificate file could not be found` | Wrong path, or saved as `.crt.txt` | Use the `Get-ChildItem` search in step 2.3 |
| Caddyfile change has no effect | Container still uses the old bind-mounted file | `docker compose up -d --force-recreate caddy` |
| Installer fails downloading the agent binary | `S3_ENDPOINT=http://localhost:9000` not reachable from endpoints | Point `S3_ENDPOINT` at the server IP, then restart the stack |

Useful logs:

```bash
cd /opt/breeze
docker compose ps
docker compose logs caddy --tail 50
docker compose logs api --tail 50
```

---

## Rollback

```bash
cd /opt/breeze
cp .env.bak .env
cp docker/Caddyfile.prod.bak docker/Caddyfile.prod
docker compose down && docker compose up -d
```

To remove the root certificate from Windows (elevated PowerShell):

```powershell
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -like "*Caddy Local Authority*" | Remove-Item
```

---

## Maintenance notes

- **Every new Windows endpoint** needs `breeze-root.crt` in Trusted Root before running the installer. Deploy it at scale with GPO (**Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Trusted Root Certification Authorities**) or with an Intune Trusted Certificate profile.
- **Breeze upgrades** may overwrite `docker/Caddyfile.prod`. After an upgrade, check that the `default_sni` line is still there:
  ```bash
  grep -n default_sni /opt/breeze/docker/Caddyfile.prod
  ```
- **Caddy's root CA** is stored in Caddy's data volume. If that volume is deleted, a new CA is generated and every endpoint needs the new `breeze-root.crt`.
- **Rotate the enrollment secret** in the console if an installer command has been shared, posted or captured in a screenshot.
- **Longer-term option:** use a real hostname (for example `rmm.example.com`) with a publicly trusted certificate obtained through a DNS challenge (such as Cloudflare). This removes the need to distribute the root CA to endpoints.
